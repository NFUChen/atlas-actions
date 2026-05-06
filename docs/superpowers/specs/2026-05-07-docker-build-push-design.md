# Docker Build & Push Reusable Workflow - Design Spec

## Overview

A reusable GitHub Actions workflow for building and pushing Docker images to any container registry. Supports multi-platform builds, custom build arguments, and configurable tagging. Follows the same patterns established by the existing Atlas workflows in this repo.

## Workflow File

`.github/workflows/docker-build-push.yml`

## Inputs

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `registry` | string | yes | — | Registry URL (e.g. `ghcr.io`, `docker.io`, `123456.dkr.ecr.us-east-1.amazonaws.com`) |
| `image-name` | string | yes | — | Full image name (e.g. `ghcr.io/org/app` or `myorg/myapp`) |
| `tags` | string | yes | — | Newline-separated list of tags to apply (e.g. `latest`, `abc1234`, `v1.0.0`). Each tag is combined with `image-name` to form the full image reference. |
| `dockerfile` | string | no | `Dockerfile` | Path to Dockerfile relative to context |
| `context` | string | no | `.` | Docker build context path |
| `platforms` | string | no | `linux/amd64` | Comma-separated target platforms (e.g. `linux/amd64,linux/arm64`) |
| `build-args` | string | no | `""` | Newline-separated build arguments (e.g. `NODE_ENV=production`) |
| `push` | boolean | no | `true` | Whether to push the image after building. Set to `false` for build-only validation. |

## Secrets

| Name | Required | Description |
|------|----------|-------------|
| `registry-username` | no | Registry username. Not needed for GHCR when using `GITHUB_TOKEN`. |
| `registry-password` | no | Registry password or access token. Not needed for GHCR when using `GITHUB_TOKEN`. |

## Job Structure

Single job: `build`

### Steps

1. **Checkout caller repo** — `actions/checkout@v4`
2. **Set up QEMU** — `docker/setup-qemu-action@v3` (enables multi-platform builds)
3. **Set up Docker Buildx** — `docker/setup-buildx-action@v3`
4. **Login to registry** — `docker/login-action@v3`
   - If `registry` is `ghcr.io`: uses `github.actor` as username and `GITHUB_TOKEN` as password (automatic, no secrets needed from caller)
   - Otherwise: uses `registry-username` and `registry-password` secrets provided by caller
   - Login step is skipped entirely when `push` is `false` (build-only mode)
5. **Build and push** — `docker/build-push-action@v6`
   - Constructs full image tags by combining `image-name` with each tag from `tags` input
   - Passes `platforms`, `build-args`, `dockerfile`, `context`, and `push` inputs
6. **Write job summary** — Displays built image name, tags, platforms, and push status in the GitHub Actions job summary

### Auth Logic

```
if push == false:
    skip login entirely
elif registry == "ghcr.io":
    username = github.actor
    password = secrets.GITHUB_TOKEN (automatic)
else:
    username = secrets.registry-username
    password = secrets.registry-password
```

## Caller Usage Examples

### GHCR — Build on PR, push on merge

```yaml
# .github/workflows/docker.yml
name: Docker

on:
  pull_request:
  push:
    branches: [main]

jobs:
  build:
    uses: NFUChen/cloud-actions/.github/workflows/docker-build-push.yml@main
    with:
      registry: ghcr.io
      image-name: ghcr.io/${{ github.repository }}
      tags: |
        ${{ github.sha }}
        ${{ github.ref_name }}
      push: ${{ github.event_name == 'push' }}
      platforms: linux/amd64,linux/arm64
```

### Docker Hub

```yaml
jobs:
  build:
    uses: NFUChen/cloud-actions/.github/workflows/docker-build-push.yml@main
    with:
      registry: docker.io
      image-name: myorg/myapp
      tags: |
        latest
        ${{ github.sha }}
      build-args: |
        NODE_ENV=production
        APP_VERSION=1.0.0
    secrets:
      registry-username: ${{ secrets.DOCKERHUB_USERNAME }}
      registry-password: ${{ secrets.DOCKERHUB_TOKEN }}
```

### Build-only validation (no push)

```yaml
jobs:
  validate:
    uses: NFUChen/cloud-actions/.github/workflows/docker-build-push.yml@main
    with:
      registry: ghcr.io
      image-name: ghcr.io/${{ github.repository }}
      tags: |
        ${{ github.sha }}
      push: false
```

## Key Design Decisions

1. **Caller-specified tags**: No auto-tagging magic. The caller is responsible for computing and passing tags. This keeps the workflow simple and predictable.
2. **GHCR auto-auth**: When `registry` is `ghcr.io`, the workflow uses `GITHUB_TOKEN` automatically so callers don't need to pass registry credentials for the most common case.
3. **Push boolean**: A single `push` input controls whether the image is pushed, enabling build-only validation in PRs without a separate workflow.
4. **Multi-platform via input**: Defaults to `linux/amd64` for speed, but callers can add `linux/arm64` etc. via the `platforms` input.
5. **Newline-separated tags and build-args**: Using newlines instead of commas avoids ambiguity in values and aligns with how `docker/build-push-action` expects these inputs.

## Actions Used

| Action | Version | Purpose |
|--------|---------|---------|
| `actions/checkout` | v4 | Checkout caller repo |
| `docker/setup-qemu-action` | v3 | Multi-platform emulation |
| `docker/setup-buildx-action` | v3 | Docker Buildx builder |
| `docker/login-action` | v3 | Registry authentication |
| `docker/build-push-action` | v6 | Build and push image |
