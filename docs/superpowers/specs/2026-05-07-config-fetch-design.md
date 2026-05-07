# Config Fetch Composite Actions — Design Spec

## Problem

K8s deployments often require config files (Helm values, ConfigMaps, env files, application config) stored in cloud storage (S3, Azure Blob, GCS) or private Git repos. Currently, cloud-actions has no mechanism to pull config from external sources before deployment. Config containing sensitive data (DB credentials, API keys) must never be persisted as GitHub artifacts.

## Solution

Four composite actions that download config files from cloud storage or Git repos directly into the runner filesystem. Callers compose these actions as steps in their own workflows alongside deployment commands. Config exists only in runner memory and is destroyed when the job ends.

## Architecture

```
.github/actions/
  config-fetch-s3/action.yml           # AWS S3
  config-fetch-azure-blob/action.yml   # Azure Blob Storage
  config-fetch-gcs/action.yml          # Google Cloud Storage
  config-fetch-git/action.yml          # Private Git Repo
```

No changes to existing K8s deploy workflows. No artifacts. No cross-job config transfer.

### Why Composite Actions (not Reusable Workflows)

- Config fetch and deploy must run in the same job so config stays on the runner filesystem
- Reusable workflows run as separate jobs with separate runners — config would need artifact transfer
- Composite actions run as steps within the caller's job — config is just a local directory
- Avoids persisting sensitive data in GitHub artifact storage

## Security Model

### No Artifact Persistence
Config files are downloaded to the runner filesystem and never uploaded as artifacts. When the job ends, the runner is destroyed along with all downloaded files.

### Authentication
Each provider supports both OIDC (keyless, recommended) and credential-based authentication, matching the pattern used in existing Terraform workflows.

| Provider | OIDC | Credential-based |
|----------|------|-------------------|
| S3 | `role-arn` via `aws-actions/configure-aws-credentials` | `aws-access-key-id` + `aws-secret-access-key` |
| Azure Blob | `azure/login` with federated credentials | `azure/login` with `client-secret` |
| GCS | `google-github-actions/auth` with Workload Identity | SA Key JSON |
| Git | GitHub App installation token | PAT |

### Log Protection
- Downloaded file contents are never echoed to logs
- Credential inputs are masked with `::add-mask::`
- Job summary shows only file paths and counts, never content

## Composite Action Specifications

### config-fetch-s3

**Inputs:**

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `bucket` | yes | — | S3 bucket name |
| `path` | yes | — | Object key or prefix (ending with `/` for multi-file) |
| `destination` | no | `./config` | Local directory to download into |
| `aws-region` | yes | — | AWS region |
| `role-arn` | no | `""` | IAM role ARN for OIDC authentication |
| `aws-access-key-id` | no | `""` | AWS access key ID (required when `role-arn` is empty) |
| `aws-secret-access-key` | no | `""` | AWS secret access key (required when `role-arn` is empty) |

**Behavior:**
- If `path` ends with `/`: `aws s3 cp s3://$BUCKET/$PATH $DEST/ --recursive --quiet`
- Otherwise: `aws s3 cp s3://$BUCKET/$PATH $DEST/ --quiet`
- Uses AWS CLI (pre-installed on GitHub runners)

### config-fetch-azure-blob

**Inputs:**

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `storage-account` | yes | — | Azure Storage account name |
| `container` | yes | — | Blob container name |
| `path` | yes | — | Blob name or pattern (e.g. `configs/*.yaml`) |
| `destination` | no | `./config` | Local directory to download into |
| `use-oidc` | no | `false` | Use OIDC federated credentials |
| `client-id` | yes | — | Service Principal or app registration client ID |
| `tenant-id` | yes | — | Azure AD tenant ID |
| `subscription-id` | yes | — | Azure subscription ID |
| `client-secret` | no | `""` | Service Principal password (required when `use-oidc` is false) |

**Behavior:**
- If `path` contains wildcard (`*`): `az storage blob download-batch --source $CONTAINER --destination $DEST --pattern "$PATH"`
- Otherwise: `az storage blob download --container-name $CONTAINER --name $PATH --file $DEST/$(basename $PATH)`
- Uses Azure CLI (pre-installed on GitHub runners)

### config-fetch-gcs

**Inputs:**

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `bucket` | yes | — | GCS bucket name |
| `path` | yes | — | Object path or prefix with wildcard (e.g. `configs/*`) |
| `destination` | no | `./config` | Local directory to download into |
| `workload-identity-provider` | no | `""` | Workload Identity Federation provider for OIDC |
| `service-account` | no | `""` | Service account email for OIDC impersonation |
| `gcp-credentials-json` | no | `""` | Service account key JSON (required when WIF is empty) |

**Behavior:**
- If `path` contains wildcard (`*`): `gsutil cp gs://$BUCKET/$PATH $DEST/`
- Otherwise: `gsutil cp gs://$BUCKET/$PATH $DEST/`
- Uses gcloud/gsutil (pre-installed on GitHub runners)

### config-fetch-git

**Inputs:**

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `repository` | yes | — | Repository in `owner/repo` format |
| `ref` | no | `main` | Branch, tag, or commit SHA to checkout |
| `path` | yes | — | Subdirectory to fetch from the repo |
| `destination` | no | `./config` | Local directory to copy config into |
| `token` | yes | — | GitHub PAT or App installation token with repo access |

**Behavior:**
```bash
git clone --depth 1 --sparse https://x-access-token:$TOKEN@github.com/$REPO.git _config-repo
cd _config-repo
git sparse-checkout set "$PATH"
cp -r "$PATH"/* ../"$DEST"/
cd ..
rm -rf _config-repo
```
- Uses sparse checkout to minimize clone size
- Cleans up cloned repo after copying config files
- Token is masked in logs via `::add-mask::`

## Caller Usage Examples

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
      id-token: write  # for OIDC
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

### Azure Blob Config + Manifest Deploy

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

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

      - uses: azure/setup-kubectl@v4

      - name: Write kubeconfig
        env:
          KUBECONFIG_B64: ${{ secrets.KUBECONFIG }}
        run: |
          mkdir -p ~/.kube
          echo "$KUBECONFIG_B64" | base64 -d > ~/.kube/config
          chmod 600 ~/.kube/config

      - name: Apply manifests
        run: kubectl apply -f ./config/ -n production
```

### Private Git Repo Config + Helm Deploy

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: NFUChen/cloud-actions/.github/actions/config-fetch-git@main
        with:
          repository: myorg/k8s-configs
          ref: main
          path: "production/myapp"
          destination: "./config"
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
            -f ./config/values-prod.yaml
```

### GCS Config + Multiple Config Sources

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@v4

      # Fetch shared config from GCS
      - uses: NFUChen/cloud-actions/.github/actions/config-fetch-gcs@main
        with:
          bucket: shared-configs
          path: "common/*"
          destination: "./config/shared"
          workload-identity-provider: projects/123/locations/global/workloadIdentityPools/github/providers/github
          service-account: ci@my-project.iam.gserviceaccount.com

      # Fetch environment-specific config from private Git repo
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

## Deliverables

1. `.github/actions/config-fetch-s3/action.yml`
2. `.github/actions/config-fetch-azure-blob/action.yml`
3. `.github/actions/config-fetch-gcs/action.yml`
4. `.github/actions/config-fetch-git/action.yml`
5. `docs/CONFIG-FETCH.md` — usage documentation
6. Updated `README.md` — add Config Fetch domain to workflow table
7. `.claude/skills/cloud-actions-config-fetch/SKILL.md` — Claude Code skill for generating caller workflows

## Out of Scope

- Template rendering (envsubst, variable substitution) — callers handle this themselves
- Config validation or schema checking
- Config encryption/decryption (use provider-native encryption like S3 SSE, Azure SSE, GCS CMEK)
- HashiCorp Vault integration
