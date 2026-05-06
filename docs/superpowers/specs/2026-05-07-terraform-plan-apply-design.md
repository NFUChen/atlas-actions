# Terraform Plan & Apply Reusable Workflows - Design Spec

## Overview

Three reusable GitHub Actions workflows for Terraform plan and apply, one per cloud provider: AWS, Azure, and GCP. Each workflow contains two jobs -- `plan` (runs freely) and `apply` (gated by GitHub Environment protection rules requiring manual approval). Each provider supports both OIDC and traditional credential authentication.

## Workflow Files

| File | Purpose |
|------|---------|
| `.github/workflows/terraform-aws.yml` | Terraform plan + apply with AWS auth |
| `.github/workflows/terraform-azure.yml` | Terraform plan + apply with Azure auth |
| `.github/workflows/terraform-gcp.yml` | Terraform plan + apply with GCP auth |

## Shared Inputs (all 3 workflows)

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `working-directory` | string | no | `.` | Directory containing Terraform configuration files |
| `terraform-version` | string | no | `latest` | Terraform version to install |
| `environment` | string | yes | -- | GitHub Environment name for approval gate (e.g. `production`, `staging`) |
| `plan-args` | string | no | `""` | Extra arguments for `terraform plan` (e.g. `-var-file=prod.tfvars`) |
| `apply-args` | string | no | `""` | Extra arguments for `terraform apply` (e.g. `-var-file=prod.tfvars`) |

## Job Structure (all 3 workflows)

Each workflow has two jobs with identical Terraform steps but provider-specific auth.

### Job 1: `plan`

Runs on every workflow call. No environment gate.

1. **Checkout caller repo** -- `actions/checkout@v4`
2. **Setup Terraform** -- `hashicorp/setup-terraform@v3` with `terraform-version` input
3. **Provider-specific auth** -- varies per workflow (see Auth sections below), skipped if not pushing (plan-only scenarios handled by caller)
4. **Terraform init** -- `terraform init` in `working-directory`
5. **Terraform validate** -- `terraform validate` in `working-directory`
6. **Terraform plan** -- `terraform plan -out=tfplan ${{ inputs.plan-args }}` in `working-directory`
7. **Write plan to job summary** -- display plan output in GitHub Actions job summary
8. **Upload plan artifact** -- `actions/upload-artifact@v4` to save `working-directory/tfplan` for the apply job

### Job 2: `apply`

Depends on `plan` job. Uses `environment: ${{ inputs.environment }}` to trigger GitHub Environment protection rules. A designated reviewer must approve in the GitHub UI before this job runs.

1. **Checkout caller repo** -- `actions/checkout@v4`
2. **Setup Terraform** -- `hashicorp/setup-terraform@v3` with `terraform-version` input
3. **Provider-specific auth** -- same auth as plan job
4. **Download plan artifact** -- `actions/download-artifact@v4` to retrieve `tfplan`
5. **Terraform init** -- `terraform init` in `working-directory`
6. **Terraform apply** -- `terraform apply ${{ inputs.apply-args }} tfplan` in `working-directory`
7. **Write apply result to job summary** -- display apply output in GitHub Actions job summary

### Approval Gate

The `apply` job specifies `environment: ${{ inputs.environment }}`. The caller must configure the GitHub Environment (e.g. `production`) with required reviewers under Settings > Environments. When the workflow runs, the `plan` job executes immediately and the `apply` job waits for manual approval in the GitHub UI.

---

## Workflow 1: AWS (`terraform-aws.yml`)

### Auth Inputs

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `aws-region` | string | yes | -- | AWS region (e.g. `us-east-1`) |
| `role-arn` | string | no | `""` | IAM role ARN for OIDC authentication. If set, OIDC is used instead of access keys. |

### Auth Secrets

| Name | Required | Description |
|------|----------|-------------|
| `aws-access-key-id` | no | AWS access key ID (required when `role-arn` is empty) |
| `aws-secret-access-key` | no | AWS secret access key (required when `role-arn` is empty) |

### Auth Logic

```
if role-arn is set:
    use OIDC: configure-aws-credentials with role-to-assume
    requires id-token: write permission
else:
    use Access Keys: configure-aws-credentials with aws-access-key-id + aws-secret-access-key
```

### Auth Step

```yaml
- name: Configure AWS credentials (OIDC)
  if: ${{ inputs.role-arn != '' }}
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ inputs.role-arn }}
    aws-region: ${{ inputs.aws-region }}

- name: Configure AWS credentials (Access Key)
  if: ${{ inputs.role-arn == '' }}
  uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.aws-access-key-id }}
    aws-secret-access-key: ${{ secrets.aws-secret-access-key }}
    aws-region: ${{ inputs.aws-region }}
```

### Permissions

```yaml
permissions:
  contents: read
  id-token: write  # required for OIDC
```

### Caller Example (OIDC)

```yaml
jobs:
  terraform:
    uses: NFUChen/cloud-actions/.github/workflows/terraform-aws.yml@main
    with:
      working-directory: infra/aws
      environment: production
      aws-region: us-east-1
      role-arn: arn:aws:iam::123456789012:role/github-actions-terraform
      plan-args: -var-file=prod.tfvars
      apply-args: -var-file=prod.tfvars
```

### Caller Example (Access Keys)

```yaml
jobs:
  terraform:
    uses: NFUChen/cloud-actions/.github/workflows/terraform-aws.yml@main
    with:
      working-directory: infra/aws
      environment: production
      aws-region: us-east-1
      plan-args: -var-file=prod.tfvars
      apply-args: -var-file=prod.tfvars
    secrets:
      aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
      aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

---

## Workflow 2: Azure (`terraform-azure.yml`)

### Auth Inputs

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `use-oidc` | boolean | no | `false` | Use OIDC federated credentials instead of client secret |

### Auth Secrets

| Name | Required | Description |
|------|----------|-------------|
| `client-id` | yes | Service Principal or app registration client ID |
| `tenant-id` | yes | Azure AD tenant ID |
| `subscription-id` | yes | Azure subscription ID |
| `client-secret` | no | Service Principal password (required when `use-oidc` is `false`) |

### Auth Logic

```
if use-oidc is true:
    use azure/login with client-id + tenant-id + subscription-id (federated)
    requires id-token: write permission
else:
    use azure/login with client-id + client-secret + tenant-id
```

### Auth Step

```yaml
- name: Login to Azure (OIDC)
  if: ${{ inputs.use-oidc }}
  uses: azure/login@v2
  with:
    client-id: ${{ secrets.client-id }}
    tenant-id: ${{ secrets.tenant-id }}
    subscription-id: ${{ secrets.subscription-id }}

- name: Login to Azure (Service Principal)
  if: ${{ !inputs.use-oidc }}
  uses: azure/login@v2
  with:
    creds: '{"clientId":"${{ secrets.client-id }}","clientSecret":"${{ secrets.client-secret }}","subscriptionId":"${{ secrets.subscription-id }}","tenantId":"${{ secrets.tenant-id }}"}'
```

### Permissions

```yaml
permissions:
  contents: read
  id-token: write  # required for OIDC
```

### Terraform Environment Variables

Azure auth for Terraform requires setting ARM environment variables after `azure/login`:

```yaml
- name: Set Terraform Azure env vars
  run: |
    echo "ARM_CLIENT_ID=${{ secrets.client-id }}" >> "$GITHUB_ENV"
    echo "ARM_TENANT_ID=${{ secrets.tenant-id }}" >> "$GITHUB_ENV"
    echo "ARM_SUBSCRIPTION_ID=${{ secrets.subscription-id }}" >> "$GITHUB_ENV"
```

For OIDC, also set:
```yaml
    echo "ARM_USE_OIDC=true" >> "$GITHUB_ENV"
```

For Service Principal, also set:
```yaml
    echo "ARM_CLIENT_SECRET=${{ secrets.client-secret }}" >> "$GITHUB_ENV"
```

### Caller Example (OIDC)

```yaml
jobs:
  terraform:
    uses: NFUChen/cloud-actions/.github/workflows/terraform-azure.yml@main
    with:
      working-directory: infra/azure
      environment: production
      use-oidc: true
      plan-args: -var-file=prod.tfvars
      apply-args: -var-file=prod.tfvars
    secrets:
      client-id: ${{ secrets.AZURE_CLIENT_ID }}
      tenant-id: ${{ secrets.AZURE_TENANT_ID }}
      subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

### Caller Example (Service Principal)

```yaml
jobs:
  terraform:
    uses: NFUChen/cloud-actions/.github/workflows/terraform-azure.yml@main
    with:
      working-directory: infra/azure
      environment: production
      plan-args: -var-file=prod.tfvars
      apply-args: -var-file=prod.tfvars
    secrets:
      client-id: ${{ secrets.AZURE_CLIENT_ID }}
      tenant-id: ${{ secrets.AZURE_TENANT_ID }}
      subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      client-secret: ${{ secrets.AZURE_CLIENT_SECRET }}
```

---

## Workflow 3: GCP (`terraform-gcp.yml`)

### Auth Inputs

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `workload-identity-provider` | string | no | `""` | Workload Identity Federation provider (e.g. `projects/123456/locations/global/workloadIdentityPools/my-pool/providers/my-provider`). If set, OIDC is used. |
| `service-account` | string | no | `""` | Service account email for OIDC impersonation (required when `workload-identity-provider` is set) |

### Auth Secrets

| Name | Required | Description |
|------|----------|-------------|
| `gcp-credentials-json` | no | Service account key JSON (required when `workload-identity-provider` is empty) |

### Auth Logic

```
if workload-identity-provider is set:
    use google-github-actions/auth with WIF + service-account
    requires id-token: write permission
else:
    use google-github-actions/auth with credentials_json
```

### Auth Step

```yaml
- name: Authenticate to GCP (OIDC)
  if: ${{ inputs.workload-identity-provider != '' }}
  uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: ${{ inputs.workload-identity-provider }}
    service_account: ${{ inputs.service-account }}

- name: Authenticate to GCP (SA Key)
  if: ${{ inputs.workload-identity-provider == '' }}
  uses: google-github-actions/auth@v2
  with:
    credentials_json: ${{ secrets.gcp-credentials-json }}
```

### Permissions

```yaml
permissions:
  contents: read
  id-token: write  # required for OIDC
```

### Caller Example (OIDC)

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
      apply-args: -var-file=prod.tfvars
```

### Caller Example (SA Key)

```yaml
jobs:
  terraform:
    uses: NFUChen/cloud-actions/.github/workflows/terraform-gcp.yml@main
    with:
      working-directory: infra/gcp
      environment: production
      plan-args: -var-file=prod.tfvars
      apply-args: -var-file=prod.tfvars
    secrets:
      gcp-credentials-json: ${{ secrets.GCP_SA_KEY }}
```

---

## Key Design Decisions

1. **Combined plan+apply per workflow**: Single workflow with two jobs instead of separate plan/apply workflows. The approval gate between jobs provides the safety boundary.
2. **Environment protection rules**: Uses GitHub's native environment approval mechanism. The `apply` job declares `environment: ${{ inputs.environment }}`, which triggers required reviewers configured in the repo's Environment settings.
3. **Plan artifact**: The `plan` job saves `tfplan` as an artifact, and the `apply` job downloads it. This ensures the exact plan that was reviewed is what gets applied.
4. **Dual auth support**: Each provider supports both OIDC (modern, no long-lived secrets) and traditional credentials (access keys / SP / SA key). The workflow detects which method to use based on whether the OIDC-specific input is provided.
5. **Separate workflows per provider**: Consistent with the Docker workflow pattern. Each workflow has explicit, provider-specific inputs and secrets.
6. **id-token: write permission**: All workflows request this permission to support OIDC. It has no effect when OIDC is not used.
7. **Terraform version input**: Callers can pin a specific version for reproducibility, or use `latest` (default).
8. **apply-args limitation**: When applying a saved plan file, Terraform ignores most CLI arguments (e.g. `-var`, `-var-file`). The `apply-args` input is useful for flags like `-parallelism` only. Variable files should be passed via `plan-args` at plan time.

## Actions Used

| Action | Version | Used In | Purpose |
|--------|---------|---------|---------|
| `actions/checkout` | v4 | All | Checkout caller repo |
| `hashicorp/setup-terraform` | v3 | All | Install Terraform CLI |
| `actions/upload-artifact` | v4 | All (plan job) | Upload tfplan artifact |
| `actions/download-artifact` | v4 | All (apply job) | Download tfplan artifact |
| `aws-actions/configure-aws-credentials` | v4 | AWS | AWS authentication |
| `azure/login` | v2 | Azure | Azure authentication |
| `google-github-actions/auth` | v2 | GCP | GCP authentication |
