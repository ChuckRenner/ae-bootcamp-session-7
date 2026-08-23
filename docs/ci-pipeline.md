# CI Pipeline

The Todo Service CI pipeline is split into a reusable workflow and a small caller workflow. The reusable workflow keeps the platform checks consistent across services, while each service's caller chooses when to run planning, deployment, and image publishing.

## Reusable Workflow

[`.github/workflows/golden-path-ci.yml`](../.github/workflows/golden-path-ci.yml) is invoked with `workflow_call`. It accepts Node.js and Terraform versions, plus booleans controlling Terraform planning, Terraform apply, and Docker image publishing.

### Jobs

- **`lint`** checks the backend and frontend with ESLint. It catches style and static-analysis problems before code is tested or built.
- **`test`** installs dependencies and runs the backend Jest suite with coverage. Jest's configured coverage threshold causes the job to fail when coverage is below 80%; the resulting line, branch, function, and statement percentages are added to the job summary.
- **`security-scan`** runs Checkov against `infra/` when Terraform planning is enabled. `--hard-fail-on HIGH` prevents high-severity infrastructure findings from passing CI.
- **`terraform-plan`** authenticates to AWS with GitHub Actions OIDC, installs the requested Terraform version, initializes the dev stack, and creates a saved plan. For non-apply runs it supplies mock VPC and subnet values for local-style planning. The plan summary is added to the job summary and the saved plan is uploaded as an artifact.
- **`docker-build`** builds the backend and frontend Dockerfiles on pull requests after lint and tests pass. It validates the images without pushing them or requiring AWS credentials.
- **`terraform-apply`** runs when enabled, after the Terraform plan. It uses OIDC credentials, removes the local mock-provider settings, initializes the S3 backend, downloads the saved plan, applies it, and publishes the deployed load balancer URL in the job summary.
- **`build-and-push`** runs when enabled after apply. It reads the ECR repository URLs from Terraform outputs, authenticates to ECR, builds and pushes both images with the commit SHA and `latest` tags, then forces a new ECS deployment.

The reusable workflow grants read access to repository contents and pull-request write access at workflow scope. AWS jobs narrow their job permissions and request `id-token: write` for OIDC.

## Adoption

A service team adds a caller workflow such as this:

```yaml
name: Todo Service CI

on:
  push:
    branches:
      - main
  pull_request:

permissions:
  contents: read
  pull-requests: write
  id-token: write

jobs:
  call-golden-path:
    uses: ./.github/workflows/golden-path-ci.yml
    with:
      node_version: "20"
      run_terraform_plan: true
      run_terraform_apply: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
      build_and_push: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
    secrets:
      aws_role_arn: ${{ secrets.AWS_ROLE_ARN }}
```

The caller's top-level `id-token: write` permission is required even though the reusable workflow declares OIDC permissions for its individual jobs. Pull requests run the validation and planning path; pushes to `main` additionally enable apply and image publishing.

## Required Checks

- **Lint**: Verifies backend and frontend JavaScript meet the repository's ESLint rules. This keeps avoidable quality issues out of the branch.
- **Test**: Runs Jest with coverage and enforces the 80% minimum. This protects the API behavior and makes coverage regressions visible.
- **Security scan**: Uses Checkov to inspect Terraform infrastructure and blocks high-severity findings. This provides an infrastructure policy gate before deployment.
- **Terraform plan**: Confirms the dev stack can initialize and produce an infrastructure plan. Reviewers can inspect the uploaded plan artifact and summary before changes are applied.
- **Docker build**: Builds both service images on pull requests. This catches Dockerfile and image-build failures before they reach ECR.

## OIDC Secret Configuration

The `terraform-plan` job uses:

```yaml
role-to-assume: ${{ secrets.aws_role_arn }}
```

Configure the repository secret named `AWS_ROLE_ARN` with the ARN of the AWS IAM role trusted by the repository's GitHub OIDC identity provider. The caller maps that repository secret to the reusable workflow's lowercase secret input:

```yaml
secrets:
  aws_role_arn: ${{ secrets.AWS_ROLE_ARN }}
```

Do not hardcode the role ARN in either workflow. The caller must also declare `id-token: write` at the workflow level. The IAM role should limit access to the Terraform resources and ECR/ECS operations required by this pipeline.

For Terraform apply and image publishing, the same OIDC secret is used by the `terraform-apply` and `build-and-push` jobs. Pull-request runs should use a role policy appropriate for the plan workflow and should not receive broader deployment permissions than necessary.
