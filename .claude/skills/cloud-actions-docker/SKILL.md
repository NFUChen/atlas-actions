---
name: cloud-actions-docker
description: Generate GitHub Actions caller workflows for Docker image build & push to container registries (GHCR, Docker Hub, AWS ECR, Azure ACR). Use this skill whenever the user mentions Docker builds, container images, pushing to a registry, multi-platform builds, CI/CD for Docker, or wants to set up automated Docker image pipelines in GitHub Actions. Also use when the user asks about cloud-actions Docker workflows, container registry CI/CD, or Dockerfile automation.
---

# Docker Build & Push Workflows

Generate caller workflow YAML for the `NFUChen/cloud-actions` Docker reusable workflows. Four separate workflows, one per container registry.

## Available Workflows

| Workflow | File | Registry | Auth Method |
|----------|------|----------|-------------|
| GHCR | `docker-build-push-ghcr.yml` | GitHub Container Registry | Automatic via `GITHUB_TOKEN` |
| Docker Hub | `docker-build-push-dockerhub.yml` | Docker Hub | Username + access token |
| AWS ECR | `docker-build-push-ecr.yml` | AWS Elastic Container Registry | AWS access key |
| Azure ACR | `docker-build-push-acr.yml` | Azure Container Registry | Service Principal |

## Choosing the Right Workflow

Ask the user which registry they use. If unsure, recommend GHCR -- it requires no extra secrets and integrates natively with GitHub.

## Base URL

```
NFUChen/cloud-actions/.github/workflows/<workflow-file>@main
```

## Shared Inputs (all 4 workflows)

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `image-name` | string | yes | -- | Full image name including registry prefix |
| `tags` | string | yes | -- | Newline-separated list of tags |
| `dockerfile` | string | no | `Dockerfile` | Path to Dockerfile relative to context |
| `context` | string | no | `.` | Docker build context path |
| `platforms` | string | no | `linux/amd64` | Comma-separated target platforms |
| `build-args` | string | no | `""` | Newline-separated build arguments |
| `push` | boolean | no | `true` | Push image after building |

## Registry-Specific Reference

### GHCR

No extra inputs. No caller secrets needed -- uses `GITHUB_TOKEN` automatically.

```yaml
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

### Docker Hub

Extra secrets: `username`, `token`

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

Extra input: `aws-region` (required). Extra secrets: `aws-access-key-id`, `aws-secret-access-key`. ECR repository must already exist.

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

### Azure ACR

Extra input: `registry` (required). Extra secrets: `client-id`, `client-secret`.

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

## Common Patterns

### Build on PR, Push on Merge

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
```

### Build-Only Validation (No Push)

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

### Multi-Platform Build

```yaml
with:
  platforms: linux/amd64,linux/arm64
```

### Build Arguments

```yaml
with:
  build-args: |
    NODE_ENV=production
    APP_VERSION=1.0.0
    GIT_SHA=${{ github.sha }}
```

## Generation Guidelines

When generating caller workflows:

- Ask which registry the user needs if not specified -- recommend GHCR for simplicity
- For `image-name`, use the correct registry prefix format for the chosen registry
- Default tags to `${{ github.sha }}` -- it's always unique and traceable
- Include `push: ${{ github.event_name == 'push' }}` pattern for PR + push workflows
- Only add `platforms` if user mentions multi-arch or ARM support
- Only add `build-args` if user mentions build-time variables
- For ECR, remind user the repository must already exist
