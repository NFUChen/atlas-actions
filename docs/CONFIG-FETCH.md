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
