# Config Fetch Composite Actions — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create 4 composite actions that download config from cloud storage (S3, Azure Blob, GCS) and private Git repos directly into the runner filesystem for use by downstream deploy steps.

**Architecture:** Each provider is a standalone composite action under `.github/actions/config-fetch-<provider>/action.yml`. Callers use these as steps within their own jobs — no artifacts, no cross-job transfer, sensitive data never leaves the runner. Auth follows existing Terraform workflow patterns (OIDC + credential-based).

**Tech Stack:** GitHub Actions composite actions, AWS CLI, Azure CLI, gcloud/gsutil, git sparse-checkout

---

## File Map

| File | Action | Responsibility |
|------|--------|----------------|
| `.github/actions/config-fetch-s3/action.yml` | Create | Download config from AWS S3 (single file or prefix) |
| `.github/actions/config-fetch-azure-blob/action.yml` | Create | Download config from Azure Blob Storage (single blob or pattern) |
| `.github/actions/config-fetch-gcs/action.yml` | Create | Download config from Google Cloud Storage (single object or wildcard) |
| `.github/actions/config-fetch-git/action.yml` | Create | Clone config from private Git repo (sparse checkout) |
| `docs/CONFIG-FETCH.md` | Create | Usage documentation for all 4 actions |
| `README.md` | Modify | Add Config Fetch row to workflow table, update skill install instructions |
| `.claude/skills/cloud-actions-config-fetch/SKILL.md` | Create | Claude Code skill for generating caller workflows with config fetch |

---

### Task 1: config-fetch-s3 composite action

**Files:**
- Create: `.github/actions/config-fetch-s3/action.yml`

- [ ] **Step 1: Create the action.yml file**

```yaml
name: Fetch Config from S3
description: Download configuration files from an AWS S3 bucket into the runner filesystem.

inputs:
  bucket:
    description: "S3 bucket name"
    required: true
  path:
    description: "S3 object key (single file) or prefix ending with / (multi-file recursive download)"
    required: true
  destination:
    description: "Local directory to download config files into"
    required: false
    default: "./config"
  aws-region:
    description: "AWS region (e.g. us-east-1)"
    required: true
  role-arn:
    description: "IAM role ARN for OIDC authentication. If set, OIDC is used instead of access keys."
    required: false
    default: ""
  aws-access-key-id:
    description: "AWS access key ID (required when role-arn is empty)"
    required: false
    default: ""
  aws-secret-access-key:
    description: "AWS secret access key (required when role-arn is empty)"
    required: false
    default: ""

runs:
  using: composite
  steps:
    - name: Mask credentials
      if: ${{ inputs.aws-access-key-id != '' }}
      shell: bash
      run: |
        echo "::add-mask::${{ inputs.aws-access-key-id }}"
        echo "::add-mask::${{ inputs.aws-secret-access-key }}"

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
        aws-access-key-id: ${{ inputs.aws-access-key-id }}
        aws-secret-access-key: ${{ inputs.aws-secret-access-key }}
        aws-region: ${{ inputs.aws-region }}

    - name: Download config from S3
      shell: bash
      env:
        S3_BUCKET: ${{ inputs.bucket }}
        S3_PATH: ${{ inputs.path }}
        DEST: ${{ inputs.destination }}
      run: |
        mkdir -p "$DEST"
        if [[ "$S3_PATH" == */ ]]; then
          aws s3 cp "s3://$S3_BUCKET/$S3_PATH" "$DEST/" --recursive --quiet
        else
          aws s3 cp "s3://$S3_BUCKET/$S3_PATH" "$DEST/" --quiet
        fi
        FILE_COUNT=$(find "$DEST" -type f | wc -l | tr -d ' ')
        echo "Downloaded $FILE_COUNT file(s) from s3://$S3_BUCKET/$S3_PATH to $DEST/"
```

- [ ] **Step 2: Validate YAML syntax**

Run: `python3 -c "import yaml; yaml.safe_load(open('.github/actions/config-fetch-s3/action.yml'))"`
Expected: No output (valid YAML)

- [ ] **Step 3: Commit**

```bash
git add .github/actions/config-fetch-s3/action.yml
git commit -m "feat: add config-fetch-s3 composite action"
```

---

### Task 2: config-fetch-azure-blob composite action

**Files:**
- Create: `.github/actions/config-fetch-azure-blob/action.yml`

- [ ] **Step 1: Create the action.yml file**

```yaml
name: Fetch Config from Azure Blob Storage
description: Download configuration files from an Azure Blob Storage container into the runner filesystem.

inputs:
  storage-account:
    description: "Azure Storage account name"
    required: true
  container:
    description: "Blob container name"
    required: true
  path:
    description: "Blob name (single file) or pattern with wildcard e.g. configs/*.yaml (multi-file)"
    required: true
  destination:
    description: "Local directory to download config files into"
    required: false
    default: "./config"
  use-oidc:
    description: "Use OIDC federated credentials instead of client secret (true/false)"
    required: false
    default: "false"
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
    default: ""

runs:
  using: composite
  steps:
    - name: Mask credentials
      if: ${{ inputs.client-secret != '' }}
      shell: bash
      run: |
        echo "::add-mask::${{ inputs.client-secret }}"

    - name: Login to Azure (OIDC)
      if: ${{ inputs.use-oidc == 'true' }}
      uses: azure/login@v2
      with:
        client-id: ${{ inputs.client-id }}
        tenant-id: ${{ inputs.tenant-id }}
        subscription-id: ${{ inputs.subscription-id }}

    - name: Login to Azure (Service Principal)
      if: ${{ inputs.use-oidc != 'true' }}
      uses: azure/login@v2
      with:
        creds: '{"clientId":"${{ inputs.client-id }}","clientSecret":"${{ inputs.client-secret }}","subscriptionId":"${{ inputs.subscription-id }}","tenantId":"${{ inputs.tenant-id }}"}'

    - name: Download config from Azure Blob
      shell: bash
      env:
        STORAGE_ACCOUNT: ${{ inputs.storage-account }}
        CONTAINER: ${{ inputs.container }}
        BLOB_PATH: ${{ inputs.path }}
        DEST: ${{ inputs.destination }}
      run: |
        mkdir -p "$DEST"
        if [[ "$BLOB_PATH" == *"*"* ]]; then
          az storage blob download-batch \
            --account-name "$STORAGE_ACCOUNT" \
            --source "$CONTAINER" \
            --destination "$DEST" \
            --pattern "$BLOB_PATH" \
            --only-show-errors
        else
          az storage blob download \
            --account-name "$STORAGE_ACCOUNT" \
            --container-name "$CONTAINER" \
            --name "$BLOB_PATH" \
            --file "$DEST/$(basename "$BLOB_PATH")" \
            --only-show-errors
        fi
        FILE_COUNT=$(find "$DEST" -type f | wc -l | tr -d ' ')
        echo "Downloaded $FILE_COUNT file(s) from $STORAGE_ACCOUNT/$CONTAINER/$BLOB_PATH to $DEST/"
```

- [ ] **Step 2: Validate YAML syntax**

Run: `python3 -c "import yaml; yaml.safe_load(open('.github/actions/config-fetch-azure-blob/action.yml'))"`
Expected: No output (valid YAML)

- [ ] **Step 3: Commit**

```bash
git add .github/actions/config-fetch-azure-blob/action.yml
git commit -m "feat: add config-fetch-azure-blob composite action"
```

---

### Task 3: config-fetch-gcs composite action

**Files:**
- Create: `.github/actions/config-fetch-gcs/action.yml`

- [ ] **Step 1: Create the action.yml file**

```yaml
name: Fetch Config from GCS
description: Download configuration files from a Google Cloud Storage bucket into the runner filesystem.

inputs:
  bucket:
    description: "GCS bucket name"
    required: true
  path:
    description: "Object path (single file) or prefix with wildcard e.g. configs/* (multi-file)"
    required: true
  destination:
    description: "Local directory to download config files into"
    required: false
    default: "./config"
  workload-identity-provider:
    description: "Workload Identity Federation provider for OIDC. If set, OIDC is used instead of SA key."
    required: false
    default: ""
  service-account:
    description: "Service account email for OIDC impersonation (required when workload-identity-provider is set)"
    required: false
    default: ""
  gcp-credentials-json:
    description: "Service account key JSON (required when workload-identity-provider is empty)"
    required: false
    default: ""

runs:
  using: composite
  steps:
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
        credentials_json: ${{ inputs.gcp-credentials-json }}

    - name: Download config from GCS
      shell: bash
      env:
        GCS_BUCKET: ${{ inputs.bucket }}
        GCS_PATH: ${{ inputs.path }}
        DEST: ${{ inputs.destination }}
      run: |
        mkdir -p "$DEST"
        if [[ "$GCS_PATH" == *"*"* ]]; then
          gsutil -q cp "gs://$GCS_BUCKET/$GCS_PATH" "$DEST/"
        else
          gsutil -q cp "gs://$GCS_BUCKET/$GCS_PATH" "$DEST/"
        fi
        FILE_COUNT=$(find "$DEST" -type f | wc -l | tr -d ' ')
        echo "Downloaded $FILE_COUNT file(s) from gs://$GCS_BUCKET/$GCS_PATH to $DEST/"
```

- [ ] **Step 2: Validate YAML syntax**

Run: `python3 -c "import yaml; yaml.safe_load(open('.github/actions/config-fetch-gcs/action.yml'))"`
Expected: No output (valid YAML)

- [ ] **Step 3: Commit**

```bash
git add .github/actions/config-fetch-gcs/action.yml
git commit -m "feat: add config-fetch-gcs composite action"
```

---

### Task 4: config-fetch-git composite action

**Files:**
- Create: `.github/actions/config-fetch-git/action.yml`

- [ ] **Step 1: Create the action.yml file**

```yaml
name: Fetch Config from Git Repo
description: Clone configuration files from a private Git repository using sparse checkout into the runner filesystem.

inputs:
  repository:
    description: "Repository in owner/repo format (e.g. myorg/k8s-configs)"
    required: true
  ref:
    description: "Branch, tag, or commit SHA to checkout"
    required: false
    default: "main"
  path:
    description: "Subdirectory to fetch from the repository"
    required: true
  destination:
    description: "Local directory to copy config files into"
    required: false
    default: "./config"
  token:
    description: "GitHub PAT or App installation token with repo access"
    required: true

runs:
  using: composite
  steps:
    - name: Mask token
      shell: bash
      run: |
        echo "::add-mask::${{ inputs.token }}"

    - name: Clone and extract config
      shell: bash
      env:
        REPO: ${{ inputs.repository }}
        REF: ${{ inputs.ref }}
        REPO_PATH: ${{ inputs.path }}
        DEST: ${{ inputs.destination }}
        TOKEN: ${{ inputs.token }}
      run: |
        mkdir -p "$DEST"
        CLONE_DIR="_config-repo-$$"
        git clone --depth 1 --branch "$REF" --sparse \
          "https://x-access-token:${TOKEN}@github.com/${REPO}.git" \
          "$CLONE_DIR" 2>&1 | grep -v 'x-access-token'
        cd "$CLONE_DIR"
        git sparse-checkout set "$REPO_PATH"
        cd ..
        cp -r "$CLONE_DIR/$REPO_PATH"/* "$DEST/" 2>/dev/null || \
          cp -r "$CLONE_DIR/$REPO_PATH" "$DEST/" 2>/dev/null
        rm -rf "$CLONE_DIR"
        FILE_COUNT=$(find "$DEST" -type f | wc -l | tr -d ' ')
        echo "Downloaded $FILE_COUNT file(s) from $REPO@$REF:$REPO_PATH to $DEST/"
```

- [ ] **Step 2: Validate YAML syntax**

Run: `python3 -c "import yaml; yaml.safe_load(open('.github/actions/config-fetch-git/action.yml'))"`
Expected: No output (valid YAML)

- [ ] **Step 3: Commit**

```bash
git add .github/actions/config-fetch-git/action.yml
git commit -m "feat: add config-fetch-git composite action"
```

---

### Task 5: Documentation — docs/CONFIG-FETCH.md

**Files:**
- Create: `docs/CONFIG-FETCH.md`

- [ ] **Step 1: Write the documentation**

```markdown
# Config Fetch Actions

Composite GitHub Actions for downloading configuration files from cloud storage or private Git repos. Config files are downloaded directly into the runner filesystem — no artifacts are created, so sensitive data never leaves the runner.

## Actions

| Action | Provider | Description |
|--------|----------|-------------|
| `config-fetch-s3` | AWS S3 | Download from S3 bucket (single file or prefix) |
| `config-fetch-azure-blob` | Azure Blob Storage | Download from blob container (single blob or pattern) |
| `config-fetch-gcs` | Google Cloud Storage | Download from GCS bucket (single object or wildcard) |
| `config-fetch-git` | Private Git Repo | Clone subdirectory via sparse checkout |

## Security

- **No artifacts** — config stays on the runner filesystem, destroyed when the job ends
- **OIDC recommended** — all cloud providers support keyless authentication
- **Log protection** — credentials are masked, file contents are never printed
- **Minimal footprint** — uses tools pre-installed on GitHub runners (AWS CLI, Azure CLI, gsutil, git)

## Usage

These are composite actions, not reusable workflows. Use them as steps within your own job — they run in the same runner as your deploy commands.

```
uses: NFUChen/cloud-actions/.github/actions/config-fetch-<provider>@main
```

### S3

```yaml
- uses: NFUChen/cloud-actions/.github/actions/config-fetch-s3@main
  with:
    bucket: my-config-bucket
    path: "configs/production/"        # trailing / = recursive download
    destination: "./config"
    aws-region: ap-northeast-1
    role-arn: arn:aws:iam::123456789:role/github-actions
```

**Single file:**

```yaml
- uses: NFUChen/cloud-actions/.github/actions/config-fetch-s3@main
  with:
    bucket: my-config-bucket
    path: "configs/values-prod.yaml"   # no trailing / = single file
    destination: "./config"
    aws-region: ap-northeast-1
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

### Azure Blob Storage

```yaml
- uses: NFUChen/cloud-actions/.github/actions/config-fetch-azure-blob@main
  with:
    storage-account: myconfigs
    container: k8s-configs
    path: "production/*.yaml"          # wildcard = batch download
    destination: "./config"
    use-oidc: "true"
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

**Single file:**

```yaml
- uses: NFUChen/cloud-actions/.github/actions/config-fetch-azure-blob@main
  with:
    storage-account: myconfigs
    container: k8s-configs
    path: "production/values-prod.yaml"
    destination: "./config"
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
    client-secret: ${{ secrets.AZURE_CLIENT_SECRET }}
```

### Google Cloud Storage

```yaml
- uses: NFUChen/cloud-actions/.github/actions/config-fetch-gcs@main
  with:
    bucket: shared-configs
    path: "common/*"                   # wildcard = multi-file
    destination: "./config"
    workload-identity-provider: projects/123/locations/global/workloadIdentityPools/github/providers/github
    service-account: ci@my-project.iam.gserviceaccount.com
```

**Single file with SA key:**

```yaml
- uses: NFUChen/cloud-actions/.github/actions/config-fetch-gcs@main
  with:
    bucket: shared-configs
    path: "configs/values-prod.yaml"
    destination: "./config"
    gcp-credentials-json: ${{ secrets.GCP_SA_KEY }}
```

### Private Git Repo

```yaml
- uses: NFUChen/cloud-actions/.github/actions/config-fetch-git@main
  with:
    repository: myorg/k8s-configs
    ref: main
    path: "production/myapp"
    destination: "./config"
    token: ${{ secrets.CONFIG_REPO_TOKEN }}
```

## Inputs

### S3

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `bucket` | yes | — | S3 bucket name |
| `path` | yes | — | Object key or prefix (ending with `/` for recursive) |
| `destination` | no | `./config` | Local download directory |
| `aws-region` | yes | — | AWS region |
| `role-arn` | no | `""` | IAM role ARN for OIDC |
| `aws-access-key-id` | no | `""` | Access key (required without OIDC) |
| `aws-secret-access-key` | no | `""` | Secret key (required without OIDC) |

### Azure Blob Storage

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `storage-account` | yes | — | Storage account name |
| `container` | yes | — | Blob container name |
| `path` | yes | — | Blob name or pattern (`*` for wildcard) |
| `destination` | no | `./config` | Local download directory |
| `use-oidc` | no | `false` | Use OIDC federated credentials |
| `client-id` | yes | — | Service Principal client ID |
| `tenant-id` | yes | — | Azure AD tenant ID |
| `subscription-id` | yes | — | Azure subscription ID |
| `client-secret` | no | `""` | SP password (required without OIDC) |

### Google Cloud Storage

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `bucket` | yes | — | GCS bucket name |
| `path` | yes | — | Object path or wildcard prefix |
| `destination` | no | `./config` | Local download directory |
| `workload-identity-provider` | no | `""` | WIF provider for OIDC |
| `service-account` | no | `""` | SA email for OIDC |
| `gcp-credentials-json` | no | `""` | SA key JSON (required without WIF) |

### Private Git Repo

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `repository` | yes | — | Repository `owner/repo` |
| `ref` | no | `main` | Branch, tag, or SHA |
| `path` | yes | — | Subdirectory to fetch |
| `destination` | no | `./config` | Local download directory |
| `token` | yes | — | PAT or App installation token |

## Common Patterns

### S3 Config + Helm Deploy

```yaml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@v4

      - uses: NFUChen/cloud-actions/.github/actions/config-fetch-s3@main
        with:
          bucket: my-config-bucket
          path: "configs/production/"
          destination: "./config"
          aws-region: ap-northeast-1
          role-arn: arn:aws:iam::123456789:role/github-actions

      - uses: azure/setup-helm@v4

      - name: Write kubeconfig
        env:
          KUBECONFIG_B64: ${{ secrets.KUBECONFIG }}
        run: |
          mkdir -p ~/.kube
          echo "$KUBECONFIG_B64" | base64 -d > ~/.kube/config
          chmod 600 ~/.kube/config

      - name: Deploy with Helm
        run: |
          helm upgrade --install myapp charts/myapp \
            -n production --create-namespace \
            -f ./config/values-prod.yaml \
            --set image.tag=${{ github.sha }}
```

### Multiple Config Sources

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@v4

      - uses: NFUChen/cloud-actions/.github/actions/config-fetch-gcs@main
        with:
          bucket: shared-configs
          path: "common/*"
          destination: "./config/shared"
          workload-identity-provider: projects/123/locations/global/workloadIdentityPools/github/providers/github
          service-account: ci@my-project.iam.gserviceaccount.com

      - uses: NFUChen/cloud-actions/.github/actions/config-fetch-git@main
        with:
          repository: myorg/env-configs
          ref: main
          path: "production"
          destination: "./config/env"
          token: ${{ secrets.CONFIG_REPO_TOKEN }}

      - uses: azure/setup-helm@v4

      - name: Write kubeconfig
        env:
          KUBECONFIG_B64: ${{ secrets.KUBECONFIG }}
        run: |
          mkdir -p ~/.kube
          echo "$KUBECONFIG_B64" | base64 -d > ~/.kube/config
          chmod 600 ~/.kube/config

      - name: Deploy with Helm
        run: |
          helm upgrade --install myapp charts/myapp \
            -n production --create-namespace \
            -f ./config/shared/base-values.yaml \
            -f ./config/env/values-prod.yaml \
            --set image.tag=${{ github.sha }}
```

### Docker Build + Config Fetch + Deploy

```yaml
name: Build & Deploy
on:
  push:
    branches: [main]

jobs:
  build:
    uses: NFUChen/cloud-actions/.github/workflows/docker-build-push-ghcr.yml@main
    with:
      image-name: ghcr.io/${{ github.repository }}
      tags: |
        ${{ github.sha }}
      push: true

  deploy:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@v4

      - uses: NFUChen/cloud-actions/.github/actions/config-fetch-s3@main
        with:
          bucket: my-config-bucket
          path: "configs/production/"
          destination: "./config"
          aws-region: ap-northeast-1
          role-arn: arn:aws:iam::123456789:role/github-actions

      - uses: azure/setup-helm@v4

      - name: Write kubeconfig
        env:
          KUBECONFIG_B64: ${{ secrets.KUBECONFIG }}
        run: |
          mkdir -p ~/.kube
          echo "$KUBECONFIG_B64" | base64 -d > ~/.kube/config
          chmod 600 ~/.kube/config

      - name: Deploy with Helm
        run: |
          helm upgrade --install myapp charts/myapp \
            -n production --create-namespace \
            -f ./config/values-prod.yaml \
            --set image.tag=${{ github.sha }}
```

## Authentication

### OIDC (Recommended)

All three cloud providers support OIDC / keyless auth. This eliminates long-lived secrets:

- **AWS**: Set `role-arn` and ensure the job has `permissions: id-token: write`
- **Azure**: Set `use-oidc: "true"` and ensure the job has `permissions: id-token: write`
- **GCS**: Set `workload-identity-provider` + `service-account` and ensure the job has `permissions: id-token: write`

### Credential-based

For environments without OIDC:

- **AWS**: Pass `aws-access-key-id` + `aws-secret-access-key` via `${{ secrets.* }}`
- **Azure**: Pass `client-secret` via `${{ secrets.* }}`
- **GCS**: Pass `gcp-credentials-json` via `${{ secrets.* }}`
- **Git**: Pass `token` (PAT or GitHub App installation token) via `${{ secrets.* }}`

## Path Conventions

| Provider | Single File | Multi-File |
|----------|-------------|------------|
| S3 | `configs/values.yaml` | `configs/production/` (trailing `/`) |
| Azure Blob | `production/values.yaml` | `production/*.yaml` (wildcard `*`) |
| GCS | `configs/values.yaml` | `configs/*` (wildcard `*`) |
| Git | n/a (always copies directory) | `production/myapp` (subdirectory path) |
```

- [ ] **Step 2: Commit**

```bash
git add docs/CONFIG-FETCH.md
git commit -m "docs: add config fetch composite actions documentation"
```

---

### Task 6: Update README.md

**Files:**
- Modify: `README.md:8-11` (workflow table)
- Modify: `README.md:63-67` (skills table)
- Modify: `README.md:80-84` (symlink instructions)
- Modify: `README.md:97-101` (copy instructions)
- Modify: `README.md:112-116` (global install instructions)

- [ ] **Step 1: Add Config Fetch rows to the workflow table**

Find this block in `README.md`:

```markdown
| **K8s** | `k8s-deploy-manifest.yml` | Deploy raw K8s manifests via `kubectl apply` |
| **K8s** | `k8s-deploy-helm.yml` | Deploy Helm charts via `helm upgrade --install` |
```

Add after it:

```markdown
| **Config** | `actions/config-fetch-s3` | Fetch config files from AWS S3 |
| **Config** | `actions/config-fetch-azure-blob` | Fetch config files from Azure Blob Storage |
| **Config** | `actions/config-fetch-gcs` | Fetch config files from Google Cloud Storage |
| **Config** | `actions/config-fetch-git` | Fetch config files from private Git repo |
```

- [ ] **Step 2: Add Config Fetch skill to the skills table**

Find this block in `README.md`:

```markdown
| `cloud-actions-k8s` | K8s deploy, kubectl apply, Helm upgrade, cluster deployment |
```

Add after it:

```markdown
| `cloud-actions-config-fetch` | Config fetch, download config, S3 config, Azure Blob config, GCS config, Git config |
```

- [ ] **Step 3: Add Config Fetch to skill install instructions (all 3 options)**

Add `cloud-actions-config-fetch` symlink/copy lines alongside the existing atlas/docker/k8s lines in each installation option section.

For Option 1 (submodule), add after the k8s symlink:
```bash
ln -s ../../.cloud-actions/.claude/skills/cloud-actions-config-fetch .claude/skills/cloud-actions-config-fetch
```

For Option 2 (copy), add after the k8s copy:
```bash
cp -r /tmp/cloud-actions/.claude/skills/cloud-actions-config-fetch .claude/skills/
```

For Option 3 (global), add after the k8s copy:
```bash
cp -r /tmp/cloud-actions/.claude/skills/cloud-actions-config-fetch ~/.claude/skills/
```

- [ ] **Step 4: Add Config Fetch docs link**

Find:
```markdown
- [K8s (Deployment)](docs/K8S.md)
```

Add after it:
```markdown
- [Config Fetch](docs/CONFIG-FETCH.md)
```

- [ ] **Step 5: Commit**

```bash
git add README.md
git commit -m "docs: add config fetch to README workflow table, skills, and install instructions"
```

---

### Task 7: Claude Code skill — cloud-actions-config-fetch

**Files:**
- Create: `.claude/skills/cloud-actions-config-fetch/SKILL.md`

- [ ] **Step 1: Create the skill file**

The skill follows the exact pattern of `.claude/skills/cloud-actions-k8s/SKILL.md`. It teaches the AI agent about all 4 config-fetch composite actions so it can generate correct caller workflow YAML.

```markdown
---
name: cloud-actions-config-fetch
description: Generate GitHub Actions workflow steps using the config-fetch composite actions to download configuration from AWS S3, Azure Blob Storage, Google Cloud Storage, or private Git repos. Use this skill whenever the user mentions fetching config, downloading configuration, config from S3/Azure/GCS/Git, external config sources, or config injection for deployments. Also use when the user asks about cloud-actions config-fetch actions or pulling config from cloud storage.
---

# Config Fetch Composite Actions

Generate workflow steps for the `NFUChen/cloud-actions` config-fetch composite actions. Four actions: one per cloud storage provider plus private Git repos.

## Available Actions

| Action | Provider | Method |
|--------|----------|--------|
| `config-fetch-s3` | AWS S3 | `aws s3 cp` -- download from S3 bucket |
| `config-fetch-azure-blob` | Azure Blob | `az storage blob download` -- download from blob container |
| `config-fetch-gcs` | GCS | `gsutil cp` -- download from GCS bucket |
| `config-fetch-git` | Private Git | `git sparse-checkout` -- clone subdirectory from repo |

## Choosing the Right Action

- **S3**: Config stored in AWS S3 buckets
- **Azure Blob**: Config stored in Azure Storage blob containers
- **GCS**: Config stored in Google Cloud Storage buckets
- **Git**: Config stored in a separate private GitHub repository

If the user isn't sure, ask where their config files are stored.

## Base URL

```
NFUChen/cloud-actions/.github/actions/config-fetch-<provider>@main
```

## Important: These are composite actions, NOT reusable workflows

Callers use these as `steps` within their own job, not as `uses:` at the job level:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: NFUChen/cloud-actions/.github/actions/config-fetch-s3@main  # step-level
        with:
          ...
```

NOT:

```yaml
jobs:
  fetch:
    uses: NFUChen/cloud-actions/.github/actions/config-fetch-s3@main  # WRONG - not a workflow
```

## Security

- Config is downloaded to the runner filesystem only — no artifacts
- Sensitive data is destroyed when the job ends
- Credentials are masked in logs
- OIDC authentication is recommended over long-lived secrets

## S3 Reference

### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `bucket` | yes | -- | S3 bucket name |
| `path` | yes | -- | Object key or prefix (ending with `/` for recursive) |
| `destination` | no | `./config` | Local download directory |
| `aws-region` | yes | -- | AWS region |
| `role-arn` | no | `""` | IAM role ARN for OIDC |
| `aws-access-key-id` | no | `""` | Access key (required without OIDC) |
| `aws-secret-access-key` | no | `""` | Secret key (required without OIDC) |

### Example -- OIDC

```yaml
- uses: NFUChen/cloud-actions/.github/actions/config-fetch-s3@main
  with:
    bucket: my-config-bucket
    path: "configs/production/"
    destination: "./config"
    aws-region: ap-northeast-1
    role-arn: arn:aws:iam::123456789:role/github-actions
```

### Example -- Access Key

```yaml
- uses: NFUChen/cloud-actions/.github/actions/config-fetch-s3@main
  with:
    bucket: my-config-bucket
    path: "configs/values-prod.yaml"
    destination: "./config"
    aws-region: ap-northeast-1
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

## Azure Blob Reference

### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `storage-account` | yes | -- | Storage account name |
| `container` | yes | -- | Blob container name |
| `path` | yes | -- | Blob name or wildcard pattern |
| `destination` | no | `./config` | Local download directory |
| `use-oidc` | no | `false` | Use OIDC federated credentials |
| `client-id` | yes | -- | Service Principal client ID |
| `tenant-id` | yes | -- | Azure AD tenant ID |
| `subscription-id` | yes | -- | Azure subscription ID |
| `client-secret` | no | `""` | SP password (required without OIDC) |

### Example -- OIDC

```yaml
- uses: NFUChen/cloud-actions/.github/actions/config-fetch-azure-blob@main
  with:
    storage-account: myconfigs
    container: k8s-configs
    path: "production/*.yaml"
    destination: "./config"
    use-oidc: "true"
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

### Example -- Service Principal

```yaml
- uses: NFUChen/cloud-actions/.github/actions/config-fetch-azure-blob@main
  with:
    storage-account: myconfigs
    container: k8s-configs
    path: "production/values-prod.yaml"
    destination: "./config"
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
    client-secret: ${{ secrets.AZURE_CLIENT_SECRET }}
```

## GCS Reference

### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `bucket` | yes | -- | GCS bucket name |
| `path` | yes | -- | Object path or wildcard prefix |
| `destination` | no | `./config` | Local download directory |
| `workload-identity-provider` | no | `""` | WIF provider for OIDC |
| `service-account` | no | `""` | SA email for OIDC |
| `gcp-credentials-json` | no | `""` | SA key JSON (required without WIF) |

### Example -- OIDC

```yaml
- uses: NFUChen/cloud-actions/.github/actions/config-fetch-gcs@main
  with:
    bucket: shared-configs
    path: "common/*"
    destination: "./config"
    workload-identity-provider: projects/123/locations/global/workloadIdentityPools/github/providers/github
    service-account: ci@my-project.iam.gserviceaccount.com
```

### Example -- SA Key

```yaml
- uses: NFUChen/cloud-actions/.github/actions/config-fetch-gcs@main
  with:
    bucket: shared-configs
    path: "configs/values-prod.yaml"
    destination: "./config"
    gcp-credentials-json: ${{ secrets.GCP_SA_KEY }}
```

## Git Repo Reference

### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `repository` | yes | -- | Repository `owner/repo` |
| `ref` | no | `main` | Branch, tag, or SHA |
| `path` | yes | -- | Subdirectory to fetch |
| `destination` | no | `./config` | Local download directory |
| `token` | yes | -- | PAT or App installation token |

### Example

```yaml
- uses: NFUChen/cloud-actions/.github/actions/config-fetch-git@main
  with:
    repository: myorg/k8s-configs
    ref: main
    path: "production/myapp"
    destination: "./config"
    token: ${{ secrets.CONFIG_REPO_TOKEN }}
```

## Path Conventions

| Provider | Single File | Multi-File |
|----------|-------------|------------|
| S3 | `configs/values.yaml` | `configs/production/` (trailing `/`) |
| Azure Blob | `production/values.yaml` | `production/*.yaml` (wildcard `*`) |
| GCS | `configs/values.yaml` | `configs/*` (wildcard `*`) |
| Git | n/a | `production/myapp` (subdirectory) |

## Common Patterns

### Config Fetch + Helm Deploy

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@v4

      - uses: NFUChen/cloud-actions/.github/actions/config-fetch-s3@main
        with:
          bucket: my-config-bucket
          path: "configs/production/"
          destination: "./config"
          aws-region: ap-northeast-1
          role-arn: arn:aws:iam::123456789:role/github-actions

      - uses: azure/setup-helm@v4

      - name: Write kubeconfig
        env:
          KUBECONFIG_B64: ${{ secrets.KUBECONFIG }}
        run: |
          mkdir -p ~/.kube
          echo "$KUBECONFIG_B64" | base64 -d > ~/.kube/config
          chmod 600 ~/.kube/config

      - name: Deploy with Helm
        run: |
          helm upgrade --install myapp charts/myapp \
            -n production --create-namespace \
            -f ./config/values-prod.yaml \
            --set image.tag=${{ github.sha }}
```

### Multiple Config Sources

Combine multiple config-fetch actions in one job:

```yaml
steps:
  - uses: NFUChen/cloud-actions/.github/actions/config-fetch-gcs@main
    with:
      bucket: shared-configs
      path: "common/*"
      destination: "./config/shared"
      workload-identity-provider: ...
      service-account: ...

  - uses: NFUChen/cloud-actions/.github/actions/config-fetch-git@main
    with:
      repository: myorg/env-configs
      path: "production"
      destination: "./config/env"
      token: ${{ secrets.CONFIG_REPO_TOKEN }}
```

## Generation Guidelines

When generating workflow steps:

- Ask which cloud provider stores the config if not clear
- Always ask whether they use OIDC or credential-based auth
- Default `destination` to `./config` unless user specifies otherwise
- For OIDC, remind user to add `permissions: id-token: write` to the job
- These are steps, not jobs -- generate them inside the user's existing deploy job
- If user mentions K8s deploy, generate the full pattern: checkout → config-fetch → helm/kubectl setup → kubeconfig → deploy
- For multiple config sources, use different `destination` subdirectories to avoid conflicts
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/cloud-actions-config-fetch/SKILL.md
git commit -m "feat: add cloud-actions-config-fetch Claude Code skill"
```

---
