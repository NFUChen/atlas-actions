# Docker Actions

Reusable GitHub Actions workflows for building and pushing Docker images. One workflow per container registry.

## Workflows

| Workflow | Registry | Auth |
|----------|----------|------|
| `docker-build-push-ghcr.yml` | GitHub Container Registry | Automatic via `GITHUB_TOKEN` |
| `docker-build-push-dockerhub.yml` | Docker Hub | Username + access token |
| `docker-build-push-ecr.yml` | AWS ECR | AWS access key |
| `docker-build-push-acr.yml` | Azure ACR | Service Principal |

## Prerequisites

- A `Dockerfile` in your repo
- Registry credentials configured as repo secrets (except GHCR which uses `GITHUB_TOKEN`)

## Usage

### GHCR — Build on PR, Push on Merge

```yaml
name: Docker

on:
  pull_request:
  push:
    branches: [main]

jobs:
  build:
    uses: NFUChen/cloud-actions/.github/workflows/docker-build-push-ghcr.yml@main
    with:
      image-name: ghcr.io/${{ github.repository }}
      tags: |
        ${{ github.sha }}
        ${{ github.ref_name }}
      push: ${{ github.event_name == 'push' }}
      platforms: linux/amd64,linux/arm64
```

No secrets needed — GHCR uses `GITHUB_TOKEN` automatically.

### Docker Hub

```yaml
jobs:
  build:
    uses: NFUChen/cloud-actions/.github/workflows/docker-build-push-dockerhub.yml@main
    with:
      image-name: myorg/myapp
      tags: |
        latest
        ${{ github.sha }}
      build-args: |
        NODE_ENV=production
    secrets:
      username: ${{ secrets.DOCKERHUB_USERNAME }}
      token: ${{ secrets.DOCKERHUB_TOKEN }}
```

### AWS ECR

```yaml
jobs:
  build:
    uses: NFUChen/cloud-actions/.github/workflows/docker-build-push-ecr.yml@main
    with:
      image-name: 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp
      aws-region: us-east-1
      tags: |
        latest
        ${{ github.sha }}
    secrets:
      aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
      aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

The ECR repository must already exist.

### Azure ACR

```yaml
jobs:
  build:
    uses: NFUChen/cloud-actions/.github/workflows/docker-build-push-acr.yml@main
    with:
      registry: myregistry.azurecr.io
      image-name: myregistry.azurecr.io/myapp
      tags: |
        latest
        ${{ github.sha }}
    secrets:
      client-id: ${{ secrets.ACR_CLIENT_ID }}
      client-secret: ${{ secrets.ACR_CLIENT_SECRET }}
```

### Build-Only Validation (No Push)

Any workflow supports build-only mode by setting `push: false`. No login is performed.

```yaml
jobs:
  validate:
    uses: NFUChen/cloud-actions/.github/workflows/docker-build-push-ghcr.yml@main
    with:
      image-name: ghcr.io/${{ github.repository }}
      tags: |
        ${{ github.sha }}
      push: false
```

## Shared Inputs (all workflows)

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `image-name` | string | yes | — | Full image name including registry prefix |
| `tags` | string | yes | — | Newline-separated list of tags |
| `dockerfile` | string | no | `Dockerfile` | Path to Dockerfile relative to context |
| `context` | string | no | `.` | Docker build context path |
| `platforms` | string | no | `linux/amd64` | Comma-separated target platforms |
| `build-args` | string | no | `""` | Newline-separated build arguments |
| `push` | boolean | no | `true` | Push image after building |

## Registry-Specific Inputs

### ECR

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `aws-region` | string | yes | AWS region (e.g. `us-east-1`) |

### ACR

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `registry` | string | yes | ACR registry URL (e.g. `myregistry.azurecr.io`) |

## Secrets

### GHCR

No secrets needed. Uses `GITHUB_TOKEN` automatically. The calling repo must grant `packages: write` permission.

### Docker Hub

| Name | Required | Description |
|------|----------|-------------|
| `username` | yes | Docker Hub username |
| `token` | yes | Docker Hub access token |

### AWS ECR

| Name | Required | Description |
|------|----------|-------------|
| `aws-access-key-id` | yes | AWS access key ID |
| `aws-secret-access-key` | yes | AWS secret access key |

### Azure ACR

| Name | Required | Description |
|------|----------|-------------|
| `client-id` | yes | Service Principal application (client) ID |
| `client-secret` | yes | Service Principal password (client secret) |

## Multi-Platform Builds

Defaults to `linux/amd64`. Add platforms via the `platforms` input:

```yaml
with:
  platforms: linux/amd64,linux/arm64
```

Uses QEMU emulation. Multi-platform builds are slower than single-platform.

## Build Arguments

Pass build-time variables via `build-args`, one per line:

```yaml
with:
  build-args: |
    NODE_ENV=production
    APP_VERSION=1.0.0
    GIT_SHA=${{ github.sha }}
```
