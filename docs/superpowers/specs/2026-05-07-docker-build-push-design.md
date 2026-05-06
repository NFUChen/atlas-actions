# Docker Build & Push Reusable Workflows - Design Spec

## Overview

Four separate reusable GitHub Actions workflows for building and pushing Docker images, one per container registry: GHCR, Docker Hub, AWS ECR, and Azure ACR. Each workflow is fully self-contained with registry-specific authentication. All share the same build inputs (tags, platforms, dockerfile, context, build-args, push). Follows the same standalone pattern established by the existing Atlas workflows.

## Workflow Files

| File | Purpose |
|------|---------|
| `.github/workflows/docker-build-push-ghcr.yml` | Build & push to GitHub Container Registry |
| `.github/workflows/docker-build-push-dockerhub.yml` | Build & push to Docker Hub |
| `.github/workflows/docker-build-push-ecr.yml` | Build & push to AWS ECR |
| `.github/workflows/docker-build-push-acr.yml` | Build & push to Azure ACR |

## Shared Inputs (all 4 workflows)

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `image-name` | string | yes | -- | Full image name including registry prefix (e.g. `ghcr.io/org/app`, `123456.dkr.ecr.us-east-1.amazonaws.com/myapp`) |
| `tags` | string | yes | -- | Newline-separated list of tags (e.g. `latest`, `abc1234`). Each tag is combined with `image-name` to form the full image reference. |
| `dockerfile` | string | no | `Dockerfile` | Path to Dockerfile relative to context |
| `context` | string | no | `.` | Docker build context path |
| `platforms` | string | no | `linux/amd64` | Comma-separated target platforms (e.g. `linux/amd64,linux/arm64`) |
| `build-args` | string | no | `""` | Newline-separated build arguments (e.g. `NODE_ENV=production`) |
| `push` | boolean | no | `true` | Whether to push the image after building. Set to `false` for build-only validation. |

## Shared Steps (all 4 workflows)

Every workflow follows the same step sequence:

1. **Checkout caller repo** -- `actions/checkout@v4`
2. **Set up QEMU** -- `docker/setup-qemu-action@v3` (enables multi-platform builds)
3. **Set up Docker Buildx** -- `docker/setup-buildx-action@v3`
4. **Registry-specific login** -- varies per workflow (see below), skipped when `push` is `false`
5. **Compute image tags** -- shell script combining `image-name` with each tag from `tags` input
6. **Build and push** -- `docker/build-push-action@v6` with `platforms`, `build-args`, `dockerfile`, `context`, `push`, and computed tags
7. **Write job summary** -- displays image name, tags, platforms, and push status

---

## Workflow 1: GHCR (`docker-build-push-ghcr.yml`)

### Extra Inputs

None.

### Secrets

None required from caller. Uses `GITHUB_TOKEN` automatically.

### Permissions

```yaml
permissions:
  contents: read
  packages: write
```

### Login Step

```yaml
- name: Login to GHCR
  if: ${{ inputs.push }}
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}
```

### Caller Example

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

---

## Workflow 2: Docker Hub (`docker-build-push-dockerhub.yml`)

### Extra Inputs

None.

### Secrets

| Name | Required | Description |
|------|----------|-------------|
| `username` | yes | Docker Hub username |
| `token` | yes | Docker Hub access token |

### Login Step

```yaml
- name: Login to Docker Hub
  if: ${{ inputs.push }}
  uses: docker/login-action@v3
  with:
    registry: docker.io
    username: ${{ secrets.username }}
    password: ${{ secrets.token }}
```

### Caller Example

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

---

## Workflow 3: AWS ECR (`docker-build-push-ecr.yml`)

### Extra Inputs

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `aws-region` | string | yes | -- | AWS region for ECR (e.g. `us-east-1`) |

### Secrets

| Name | Required | Description |
|------|----------|-------------|
| `aws-access-key-id` | yes | AWS access key ID |
| `aws-secret-access-key` | yes | AWS secret access key |

### Assumptions

- The ECR repository must already exist. The workflow does not create it.

### Login Steps

```yaml
- name: Configure AWS credentials
  if: ${{ inputs.push }}
  uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.aws-access-key-id }}
    aws-secret-access-key: ${{ secrets.aws-secret-access-key }}
    aws-region: ${{ inputs.aws-region }}

- name: Login to Amazon ECR
  if: ${{ inputs.push }}
  uses: aws-actions/amazon-ecr-login@v2
```

### Caller Example

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

---

## Workflow 4: Azure ACR (`docker-build-push-acr.yml`)

### Extra Inputs

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `registry` | string | yes | -- | ACR registry URL (e.g. `myregistry.azurecr.io`) |

### Secrets

| Name | Required | Description |
|------|----------|-------------|
| `client-id` | yes | Azure Service Principal application (client) ID |
| `client-secret` | yes | Azure Service Principal password (client secret) |

### Login Step

```yaml
- name: Login to Azure ACR
  if: ${{ inputs.push }}
  uses: docker/login-action@v3
  with:
    registry: ${{ inputs.registry }}
    username: ${{ secrets.client-id }}
    password: ${{ secrets.client-secret }}
```

### Caller Example

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

---

## Key Design Decisions

1. **Separate workflows per registry**: Each workflow has explicit, registry-specific inputs and secrets. No generic `registry-username`/`registry-password` confusion. Callers see exactly what credentials they need.
2. **Caller-specified tags**: No auto-tagging. The caller computes and passes tags. Simple and predictable.
3. **Push boolean**: Controls whether the image is pushed, enabling build-only validation in PRs.
4. **Multi-platform via input**: Defaults to `linux/amd64`. Callers add platforms as needed.
5. **Newline-separated tags and build-args**: Aligns with `docker/build-push-action` input format.
6. **ECR repo must exist**: The workflow does not auto-create ECR repositories.
7. **Azure ACR uses Service Principal**: Login via `docker/login-action` with SP client ID and secret.
8. **AWS ECR uses Access Keys**: Login via `aws-actions/configure-aws-credentials` + `aws-actions/amazon-ecr-login`.

## Actions Used

| Action | Version | Used In | Purpose |
|--------|---------|---------|---------|
| `actions/checkout` | v4 | All | Checkout caller repo |
| `docker/setup-qemu-action` | v3 | All | Multi-platform emulation |
| `docker/setup-buildx-action` | v3 | All | Docker Buildx builder |
| `docker/login-action` | v3 | GHCR, Docker Hub, ACR | Registry authentication |
| `docker/build-push-action` | v6 | All | Build and push image |
| `aws-actions/configure-aws-credentials` | v4 | ECR | AWS credential configuration |
| `aws-actions/amazon-ecr-login` | v2 | ECR | ECR authentication |
