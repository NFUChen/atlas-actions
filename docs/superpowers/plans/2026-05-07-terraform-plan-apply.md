# Terraform Plan & Apply Workflows — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create three reusable GitHub Actions workflows for Terraform plan and apply — one per cloud provider (AWS, Azure, GCP) — with environment-gated approval, dual auth (OIDC + traditional), and unified documentation.

**Architecture:** Three standalone `workflow_call` workflows, each with two jobs: `plan` (runs freely, uploads tfplan artifact) and `apply` (gated by GitHub Environment protection rules, downloads and applies the saved plan). Auth detection uses conditional steps based on whether OIDC-specific inputs are provided.

**Tech Stack:** GitHub Actions YAML, Terraform CLI, AWS/Azure/GCP auth actions

**Spec:** `docs/superpowers/specs/2026-05-07-terraform-plan-apply-design.md`

---

## File Structure

| File | Action | Responsibility |
|------|--------|----------------|
| `.github/workflows/terraform-aws.yml` | Create | AWS Terraform workflow with OIDC + Access Key auth |
| `.github/workflows/terraform-azure.yml` | Create | Azure Terraform workflow with OIDC + Service Principal auth |
| `.github/workflows/terraform-gcp.yml` | Create | GCP Terraform workflow with OIDC + SA Key auth |
| `docs/TERRAFORM.md` | Create | Usage documentation for all 3 workflows |

---

### Task 1: Create the AWS Terraform workflow

**Files:**
- Create: `.github/workflows/terraform-aws.yml`

- [ ] **Step 1: Create the workflow file**

Create `.github/workflows/terraform-aws.yml` with this exact content:

```yaml
name: Terraform (AWS)

on:
  workflow_call:
    inputs:
      working-directory:
        description: "Directory containing Terraform configuration files"
        required: false
        type: string
        default: "."
      terraform-version:
        description: "Terraform version to install"
        required: false
        type: string
        default: "latest"
      environment:
        description: "GitHub Environment name for approval gate (e.g. production)"
        required: true
        type: string
      plan-args:
        description: "Extra arguments for terraform plan (e.g. -var-file=prod.tfvars)"
        required: false
        type: string
        default: ""
      apply-args:
        description: "Extra arguments for terraform apply (e.g. -parallelism=10). Note: -var and -var-file are ignored when applying a saved plan."
        required: false
        type: string
        default: ""
      aws-region:
        description: "AWS region (e.g. us-east-1)"
        required: true
        type: string
      role-arn:
        description: "IAM role ARN for OIDC authentication. If set, OIDC is used instead of access keys."
        required: false
        type: string
        default: ""
    secrets:
      aws-access-key-id:
        description: "AWS access key ID (required when role-arn is empty)"
        required: false
      aws-secret-access-key:
        description: "AWS secret access key (required when role-arn is empty)"
        required: false

jobs:
  plan:
    name: Terraform Plan
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - name: Checkout caller repo
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ inputs.terraform-version }}

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

      - name: Terraform init
        working-directory: ${{ inputs.working-directory }}
        run: terraform init

      - name: Terraform validate
        working-directory: ${{ inputs.working-directory }}
        run: terraform validate

      - name: Terraform plan
        id: plan
        working-directory: ${{ inputs.working-directory }}
        run: |
          terraform plan -out=tfplan -no-color ${{ inputs.plan-args }} 2>&1 | tee plan-output.txt
          echo "exitcode=$?" >> "$GITHUB_OUTPUT"

      - name: Write plan to job summary
        working-directory: ${{ inputs.working-directory }}
        run: |
          {
            echo "## Terraform Plan (AWS)"
            echo ""
            echo "**Working directory**: \`${{ inputs.working-directory }}\`"
            echo "**Environment**: \`${{ inputs.environment }}\`"
            echo ""
            echo '```'
            cat plan-output.txt
            echo '```'
          } >> "$GITHUB_STEP_SUMMARY"

      - name: Upload plan artifact
        uses: actions/upload-artifact@v4
        with:
          name: tfplan
          path: ${{ inputs.working-directory }}/tfplan
          retention-days: 5

  apply:
    name: Terraform Apply
    needs: plan
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    permissions:
      contents: read
      id-token: write
    steps:
      - name: Checkout caller repo
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ inputs.terraform-version }}

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

      - name: Download plan artifact
        uses: actions/download-artifact@v4
        with:
          name: tfplan
          path: ${{ inputs.working-directory }}

      - name: Terraform init
        working-directory: ${{ inputs.working-directory }}
        run: terraform init

      - name: Terraform apply
        working-directory: ${{ inputs.working-directory }}
        run: |
          terraform apply -no-color ${{ inputs.apply-args }} tfplan 2>&1 | tee apply-output.txt

      - name: Write apply result to job summary
        working-directory: ${{ inputs.working-directory }}
        run: |
          {
            echo "## Terraform Apply (AWS)"
            echo ""
            echo "**Working directory**: \`${{ inputs.working-directory }}\`"
            echo "**Environment**: \`${{ inputs.environment }}\`"
            echo ""
            echo '```'
            cat apply-output.txt
            echo '```'
          } >> "$GITHUB_STEP_SUMMARY"
```

- [ ] **Step 2: Validate YAML syntax**

Run:
```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/terraform-aws.yml'))" && echo "YAML valid"
```
Expected: `YAML valid`

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/terraform-aws.yml
git commit -m "feat: add Terraform plan & apply workflow for AWS

Two-job workflow with environment approval gate.
Supports OIDC (role-arn) and Access Key authentication.
Plan artifact is uploaded and applied after approval."
```

---

### Task 2: Create the Azure Terraform workflow

**Files:**
- Create: `.github/workflows/terraform-azure.yml`

- [ ] **Step 1: Create the workflow file**

Create `.github/workflows/terraform-azure.yml` with this exact content:

```yaml
name: Terraform (Azure)

on:
  workflow_call:
    inputs:
      working-directory:
        description: "Directory containing Terraform configuration files"
        required: false
        type: string
        default: "."
      terraform-version:
        description: "Terraform version to install"
        required: false
        type: string
        default: "latest"
      environment:
        description: "GitHub Environment name for approval gate (e.g. production)"
        required: true
        type: string
      plan-args:
        description: "Extra arguments for terraform plan (e.g. -var-file=prod.tfvars)"
        required: false
        type: string
        default: ""
      apply-args:
        description: "Extra arguments for terraform apply (e.g. -parallelism=10). Note: -var and -var-file are ignored when applying a saved plan."
        required: false
        type: string
        default: ""
      use-oidc:
        description: "Use OIDC federated credentials instead of client secret"
        required: false
        type: boolean
        default: false
    secrets:
      client-id:
        description: "Service Principal or app registration client ID"
        required: true
      tenant-id:
        description: "Azure AD tenant ID"
        required: true
      subscription-id:
        description: "Azure subscription ID"
        required: true
      client-secret:
        description: "Service Principal password (required when use-oidc is false)"
        required: false

jobs:
  plan:
    name: Terraform Plan
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    env:
      ARM_CLIENT_ID: ${{ secrets.client-id }}
      ARM_TENANT_ID: ${{ secrets.tenant-id }}
      ARM_SUBSCRIPTION_ID: ${{ secrets.subscription-id }}
    steps:
      - name: Checkout caller repo
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ inputs.terraform-version }}

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

      - name: Set Terraform ARM env vars (OIDC)
        if: ${{ inputs.use-oidc }}
        run: echo "ARM_USE_OIDC=true" >> "$GITHUB_ENV"

      - name: Set Terraform ARM env vars (Service Principal)
        if: ${{ !inputs.use-oidc }}
        run: echo "ARM_CLIENT_SECRET=${{ secrets.client-secret }}" >> "$GITHUB_ENV"

      - name: Terraform init
        working-directory: ${{ inputs.working-directory }}
        run: terraform init

      - name: Terraform validate
        working-directory: ${{ inputs.working-directory }}
        run: terraform validate

      - name: Terraform plan
        working-directory: ${{ inputs.working-directory }}
        run: |
          terraform plan -out=tfplan -no-color ${{ inputs.plan-args }} 2>&1 | tee plan-output.txt

      - name: Write plan to job summary
        working-directory: ${{ inputs.working-directory }}
        run: |
          {
            echo "## Terraform Plan (Azure)"
            echo ""
            echo "**Working directory**: \`${{ inputs.working-directory }}\`"
            echo "**Environment**: \`${{ inputs.environment }}\`"
            echo ""
            echo '```'
            cat plan-output.txt
            echo '```'
          } >> "$GITHUB_STEP_SUMMARY"

      - name: Upload plan artifact
        uses: actions/upload-artifact@v4
        with:
          name: tfplan
          path: ${{ inputs.working-directory }}/tfplan
          retention-days: 5

  apply:
    name: Terraform Apply
    needs: plan
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    permissions:
      contents: read
      id-token: write
    env:
      ARM_CLIENT_ID: ${{ secrets.client-id }}
      ARM_TENANT_ID: ${{ secrets.tenant-id }}
      ARM_SUBSCRIPTION_ID: ${{ secrets.subscription-id }}
    steps:
      - name: Checkout caller repo
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ inputs.terraform-version }}

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

      - name: Set Terraform ARM env vars (OIDC)
        if: ${{ inputs.use-oidc }}
        run: echo "ARM_USE_OIDC=true" >> "$GITHUB_ENV"

      - name: Set Terraform ARM env vars (Service Principal)
        if: ${{ !inputs.use-oidc }}
        run: echo "ARM_CLIENT_SECRET=${{ secrets.client-secret }}" >> "$GITHUB_ENV"

      - name: Download plan artifact
        uses: actions/download-artifact@v4
        with:
          name: tfplan
          path: ${{ inputs.working-directory }}

      - name: Terraform init
        working-directory: ${{ inputs.working-directory }}
        run: terraform init

      - name: Terraform apply
        working-directory: ${{ inputs.working-directory }}
        run: |
          terraform apply -no-color ${{ inputs.apply-args }} tfplan 2>&1 | tee apply-output.txt

      - name: Write apply result to job summary
        working-directory: ${{ inputs.working-directory }}
        run: |
          {
            echo "## Terraform Apply (Azure)"
            echo ""
            echo "**Working directory**: \`${{ inputs.working-directory }}\`"
            echo "**Environment**: \`${{ inputs.environment }}\`"
            echo ""
            echo '```'
            cat apply-output.txt
            echo '```'
          } >> "$GITHUB_STEP_SUMMARY"
```

- [ ] **Step 2: Validate YAML syntax**

Run:
```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/terraform-azure.yml'))" && echo "YAML valid"
```
Expected: `YAML valid`

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/terraform-azure.yml
git commit -m "feat: add Terraform plan & apply workflow for Azure

Two-job workflow with environment approval gate.
Supports OIDC federated credentials and Service Principal auth.
Sets ARM_* env vars for Terraform Azure provider."
```

---

### Task 3: Create the GCP Terraform workflow

**Files:**
- Create: `.github/workflows/terraform-gcp.yml`

- [ ] **Step 1: Create the workflow file**

Create `.github/workflows/terraform-gcp.yml` with this exact content:

```yaml
name: Terraform (GCP)

on:
  workflow_call:
    inputs:
      working-directory:
        description: "Directory containing Terraform configuration files"
        required: false
        type: string
        default: "."
      terraform-version:
        description: "Terraform version to install"
        required: false
        type: string
        default: "latest"
      environment:
        description: "GitHub Environment name for approval gate (e.g. production)"
        required: true
        type: string
      plan-args:
        description: "Extra arguments for terraform plan (e.g. -var-file=prod.tfvars)"
        required: false
        type: string
        default: ""
      apply-args:
        description: "Extra arguments for terraform apply (e.g. -parallelism=10). Note: -var and -var-file are ignored when applying a saved plan."
        required: false
        type: string
        default: ""
      workload-identity-provider:
        description: "Workload Identity Federation provider. If set, OIDC is used instead of SA key."
        required: false
        type: string
        default: ""
      service-account:
        description: "Service account email for OIDC impersonation (required when workload-identity-provider is set)"
        required: false
        type: string
        default: ""
    secrets:
      gcp-credentials-json:
        description: "Service account key JSON (required when workload-identity-provider is empty)"
        required: false

jobs:
  plan:
    name: Terraform Plan
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - name: Checkout caller repo
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ inputs.terraform-version }}

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

      - name: Terraform init
        working-directory: ${{ inputs.working-directory }}
        run: terraform init

      - name: Terraform validate
        working-directory: ${{ inputs.working-directory }}
        run: terraform validate

      - name: Terraform plan
        working-directory: ${{ inputs.working-directory }}
        run: |
          terraform plan -out=tfplan -no-color ${{ inputs.plan-args }} 2>&1 | tee plan-output.txt

      - name: Write plan to job summary
        working-directory: ${{ inputs.working-directory }}
        run: |
          {
            echo "## Terraform Plan (GCP)"
            echo ""
            echo "**Working directory**: \`${{ inputs.working-directory }}\`"
            echo "**Environment**: \`${{ inputs.environment }}\`"
            echo ""
            echo '```'
            cat plan-output.txt
            echo '```'
          } >> "$GITHUB_STEP_SUMMARY"

      - name: Upload plan artifact
        uses: actions/upload-artifact@v4
        with:
          name: tfplan
          path: ${{ inputs.working-directory }}/tfplan
          retention-days: 5

  apply:
    name: Terraform Apply
    needs: plan
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    permissions:
      contents: read
      id-token: write
    steps:
      - name: Checkout caller repo
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ inputs.terraform-version }}

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

      - name: Download plan artifact
        uses: actions/download-artifact@v4
        with:
          name: tfplan
          path: ${{ inputs.working-directory }}

      - name: Terraform init
        working-directory: ${{ inputs.working-directory }}
        run: terraform init

      - name: Terraform apply
        working-directory: ${{ inputs.working-directory }}
        run: |
          terraform apply -no-color ${{ inputs.apply-args }} tfplan 2>&1 | tee apply-output.txt

      - name: Write apply result to job summary
        working-directory: ${{ inputs.working-directory }}
        run: |
          {
            echo "## Terraform Apply (GCP)"
            echo ""
            echo "**Working directory**: \`${{ inputs.working-directory }}\`"
            echo "**Environment**: \`${{ inputs.environment }}\`"
            echo ""
            echo '```'
            cat apply-output.txt
            echo '```'
          } >> "$GITHUB_STEP_SUMMARY"
```

- [ ] **Step 2: Validate YAML syntax**

Run:
```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/terraform-gcp.yml'))" && echo "YAML valid"
```
Expected: `YAML valid`

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/terraform-gcp.yml
git commit -m "feat: add Terraform plan & apply workflow for GCP

Two-job workflow with environment approval gate.
Supports OIDC (Workload Identity Federation) and SA key auth.
Plan artifact is uploaded and applied after approval."
```

---

### Task 4: Create documentation

**Files:**
- Create: `docs/TERRAFORM.md`

- [ ] **Step 1: Create the documentation file**

Create `docs/TERRAFORM.md` with this exact content:

````markdown
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
````

- [ ] **Step 2: Commit**

```bash
git add docs/TERRAFORM.md
git commit -m "docs: add Terraform plan & apply workflow documentation

Covers all 3 provider workflows (AWS, Azure, GCP) with OIDC
and traditional auth examples, inputs, secrets, and
environment approval setup instructions."
```

---

### Task 5: Validate all workflows

- [ ] **Step 1: Validate all YAML files**

Run:
```bash
for f in .github/workflows/terraform-*.yml; do
  python3 -c "import yaml; yaml.safe_load(open('$f'))" && echo "$f: YAML valid"
done
```
Expected: 3 lines, each ending with `YAML valid`.

- [ ] **Step 2: Verify all files exist**

Run:
```bash
ls -la .github/workflows/terraform-aws.yml
ls -la .github/workflows/terraform-azure.yml
ls -la .github/workflows/terraform-gcp.yml
ls -la docs/TERRAFORM.md
```
Expected: All 4 files listed.

- [ ] **Step 3: Review git log**

Run:
```bash
git log --oneline -6
```
Expected: 4 new commits — one per workflow file, one for docs.
