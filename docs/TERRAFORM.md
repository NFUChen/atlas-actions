# Terraform Actions

Reusable GitHub Actions workflows for Terraform plan and apply. One workflow per cloud provider. Each workflow runs `terraform plan`, then waits for manual approval via GitHub Environment protection rules before running `terraform apply`.

## Workflows

| Workflow | Provider | Auth Options |
|----------|----------|-------------|
| `terraform-aws.yml` | AWS | OIDC (role ARN) or Access Keys |
| `terraform-azure.yml` | Azure | OIDC (federated) or Service Principal |
| `terraform-gcp.yml` | GCP | OIDC (Workload Identity) or SA Key |

## Prerequisites

- Terraform configuration in your repo
- A GitHub Environment configured with required reviewers (Settings > Environments)
- Cloud credentials configured as repo secrets

## How It Works

1. Caller workflow triggers the reusable workflow
2. **Plan job** runs immediately: `terraform init` > `validate` > `plan`
3. Plan output is shown in the job summary and saved as an artifact
4. **Apply job** waits for manual approval via the GitHub Environment gate
5. A reviewer approves in the GitHub UI
6. Apply job runs: downloads the saved plan > `terraform init` > `terraform apply`

## Usage

### AWS (OIDC)

```yaml
name: Terraform

on:
  push:
    branches: [main]
    paths: [infra/aws/**]

jobs:
  terraform:
    uses: NFUChen/cloud-actions/.github/workflows/terraform-aws.yml@main
    with:
      working-directory: infra/aws
      environment: production
      aws-region: us-east-1
      role-arn: arn:aws:iam::123456789012:role/github-actions-terraform
      plan-args: -var-file=prod.tfvars
```

### AWS (Access Keys)

```yaml
jobs:
  terraform:
    uses: NFUChen/cloud-actions/.github/workflows/terraform-aws.yml@main
    with:
      working-directory: infra/aws
      environment: production
      aws-region: us-east-1
      plan-args: -var-file=prod.tfvars
    secrets:
      aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
      aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

### Azure (OIDC)

```yaml
jobs:
  terraform:
    uses: NFUChen/cloud-actions/.github/workflows/terraform-azure.yml@main
    with:
      working-directory: infra/azure
      environment: production
      use-oidc: true
      plan-args: -var-file=prod.tfvars
    secrets:
      client-id: ${{ secrets.AZURE_CLIENT_ID }}
      tenant-id: ${{ secrets.AZURE_TENANT_ID }}
      subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

### Azure (Service Principal)

```yaml
jobs:
  terraform:
    uses: NFUChen/cloud-actions/.github/workflows/terraform-azure.yml@main
    with:
      working-directory: infra/azure
      environment: production
      plan-args: -var-file=prod.tfvars
    secrets:
      client-id: ${{ secrets.AZURE_CLIENT_ID }}
      tenant-id: ${{ secrets.AZURE_TENANT_ID }}
      subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      client-secret: ${{ secrets.AZURE_CLIENT_SECRET }}
```

### GCP (OIDC / Workload Identity Federation)

```yaml
jobs:
  terraform:
    uses: NFUChen/cloud-actions/.github/workflows/terraform-gcp.yml@main
    with:
      working-directory: infra/gcp
      environment: production
      workload-identity-provider: projects/123456/locations/global/workloadIdentityPools/my-pool/providers/my-provider
      service-account: terraform@my-project.iam.gserviceaccount.com
      plan-args: -var-file=prod.tfvars
```

### GCP (Service Account Key)

```yaml
jobs:
  terraform:
    uses: NFUChen/cloud-actions/.github/workflows/terraform-gcp.yml@main
    with:
      working-directory: infra/gcp
      environment: production
      plan-args: -var-file=prod.tfvars
    secrets:
      gcp-credentials-json: ${{ secrets.GCP_SA_KEY }}
```

## Shared Inputs (all workflows)

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `working-directory` | string | no | `.` | Directory containing Terraform config |
| `terraform-version` | string | no | `latest` | Terraform version to install |
| `environment` | string | yes | — | GitHub Environment name for approval gate |
| `plan-args` | string | no | `""` | Extra args for `terraform plan` |
| `apply-args` | string | no | `""` | Extra args for `terraform apply` (limited — see note) |

> **Note:** When applying a saved plan file, Terraform ignores most CLI arguments like `-var` and `-var-file`. Pass variable files via `plan-args` at plan time. The `apply-args` input is useful for flags like `-parallelism` only.

## Provider-Specific Inputs

### AWS

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `aws-region` | string | yes | — | AWS region |
| `role-arn` | string | no | `""` | IAM role ARN for OIDC. If set, OIDC is used. |

### Azure

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `use-oidc` | boolean | no | `false` | Use OIDC federated credentials |

### GCP

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `workload-identity-provider` | string | no | `""` | WIF provider. If set, OIDC is used. |
| `service-account` | string | no | `""` | SA email for OIDC impersonation |

## Secrets

### AWS

| Name | Required | Description |
|------|----------|-------------|
| `aws-access-key-id` | no | Access key (when not using OIDC) |
| `aws-secret-access-key` | no | Secret key (when not using OIDC) |

### Azure

| Name | Required | Description |
|------|----------|-------------|
| `client-id` | yes | SP or app registration client ID |
| `tenant-id` | yes | Azure AD tenant ID |
| `subscription-id` | yes | Azure subscription ID |
| `client-secret` | no | SP password (when not using OIDC) |

### GCP

| Name | Required | Description |
|------|----------|-------------|
| `gcp-credentials-json` | no | SA key JSON (when not using OIDC) |

## Setting Up Environment Approval

1. Go to your repo's **Settings > Environments**
2. Create an environment (e.g. `production`)
3. Enable **Required reviewers** and add the approvers
4. Pass the environment name via the `environment` input when calling the workflow

The `plan` job runs immediately. The `apply` job waits for approval in the GitHub UI.
