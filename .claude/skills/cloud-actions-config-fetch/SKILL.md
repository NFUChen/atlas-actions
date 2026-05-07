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

- Config is downloaded to the runner filesystem only -- no artifacts
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
- If user mentions K8s deploy, generate the full pattern: checkout -> config-fetch -> helm/kubectl setup -> kubeconfig -> deploy
- For multiple config sources, use different `destination` subdirectories to avoid conflicts
