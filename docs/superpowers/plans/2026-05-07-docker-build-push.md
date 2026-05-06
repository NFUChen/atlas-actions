# Docker Build & Push Registry Workflows — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create four separate reusable GitHub Actions workflows for building and pushing Docker images — one per registry (GHCR, Docker Hub, AWS ECR, Azure ACR) — plus unified documentation.

**Architecture:** Four standalone `workflow_call` workflows, each with registry-specific auth and shared build logic (QEMU, Buildx, build-push-action). No shared composite actions or cross-workflow dependencies. Each workflow is fully self-contained.

**Tech Stack:** GitHub Actions YAML, Docker Buildx, QEMU, AWS Actions (ECR only)

**Spec:** `docs/superpowers/specs/2026-05-07-docker-build-push-design.md`

---

## File Structure

| File | Action | Responsibility |
|------|--------|----------------|
| `.github/workflows/docker-build-push-ghcr.yml` | Create | GHCR workflow with GITHUB_TOKEN auto-auth |
| `.github/workflows/docker-build-push-dockerhub.yml` | Create | Docker Hub workflow with username/token auth |
| `.github/workflows/docker-build-push-ecr.yml` | Create | AWS ECR workflow with Access Key auth |
| `.github/workflows/docker-build-push-acr.yml` | Create | Azure ACR workflow with Service Principal auth |
| `docs/DOCKER.md` | Create | Usage documentation for all 4 workflows |

---

### Task 1: Create the GHCR workflow

**Files:**
- Create: `.github/workflows/docker-build-push-ghcr.yml`

- [ ] **Step 1: Create the workflow file**

Create `.github/workflows/docker-build-push-ghcr.yml` with this exact content:

```yaml
name: Docker Build & Push (GHCR)

on:
  workflow_call:
    inputs:
      image-name:
        description: "Full image name (e.g. ghcr.io/org/app)"
        required: true
        type: string
      tags:
        description: "Newline-separated list of tags (e.g. latest, abc1234, v1.0.0)"
        required: true
        type: string
      dockerfile:
        description: "Path to Dockerfile relative to context"
        required: false
        type: string
        default: "Dockerfile"
      context:
        description: "Docker build context path"
        required: false
        type: string
        default: "."
      platforms:
        description: "Comma-separated target platforms (e.g. linux/amd64,linux/arm64)"
        required: false
        type: string
        default: "linux/amd64"
      build-args:
        description: "Newline-separated build arguments (e.g. NODE_ENV=production)"
        required: false
        type: string
        default: ""
      push:
        description: "Whether to push the image after building (set false for build-only validation)"
        required: false
        type: boolean
        default: true

jobs:
  build:
    name: Build & Push
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - name: Checkout caller repo
        uses: actions/checkout@v4

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GHCR
        if: ${{ inputs.push }}
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Compute image tags
        id: tags
        env:
          IMAGE_NAME: ${{ inputs.image-name }}
          TAGS_INPUT: ${{ inputs.tags }}
        run: |
          FULL_TAGS=""
          while IFS= read -r tag; do
            tag=$(echo "$tag" | xargs)
            [ -z "$tag" ] && continue
            if [ -n "$FULL_TAGS" ]; then
              FULL_TAGS="${FULL_TAGS},"
            fi
            FULL_TAGS="${FULL_TAGS}${IMAGE_NAME}:${tag}"
          done <<< "$TAGS_INPUT"
          echo "tags=${FULL_TAGS}" >> "$GITHUB_OUTPUT"

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: ${{ inputs.context }}
          file: ${{ inputs.context }}/${{ inputs.dockerfile }}
          platforms: ${{ inputs.platforms }}
          push: ${{ inputs.push }}
          tags: ${{ steps.tags.outputs.tags }}
          build-args: ${{ inputs.build-args }}

      - name: Write job summary
        env:
          IMAGE_NAME: ${{ inputs.image-name }}
          TAGS_INPUT: ${{ inputs.tags }}
          PLATFORMS: ${{ inputs.platforms }}
          PUSH: ${{ inputs.push }}
        run: |
          {
            echo "## Docker Build & Push (GHCR)"
            echo ""
            if [ "$PUSH" = "true" ]; then
              echo "**Status**: Pushed"
            else
              echo "**Status**: Built (not pushed)"
            fi
            echo "**Image**: \`${IMAGE_NAME}\`"
            echo "**Platforms**: \`${PLATFORMS}\`"
            echo ""
            echo "### Tags"
            echo ""
            while IFS= read -r tag; do
              tag=$(echo "$tag" | xargs)
              [ -z "$tag" ] && continue
              echo "- \`${IMAGE_NAME}:${tag}\`"
            done <<< "$TAGS_INPUT"
          } >> "$GITHUB_STEP_SUMMARY"
```

- [ ] **Step 2: Validate YAML syntax**

Run:
```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/docker-build-push-ghcr.yml'))" && echo "YAML valid"
```
Expected: `YAML valid`

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/docker-build-push-ghcr.yml
git commit -m "feat: add Docker build & push workflow for GHCR

Uses GITHUB_TOKEN for automatic authentication.
Supports multi-platform builds, caller-specified tags,
build args, and build-only validation mode."
```

---

### Task 2: Create the Docker Hub workflow

**Files:**
- Create: `.github/workflows/docker-build-push-dockerhub.yml`

- [ ] **Step 1: Create the workflow file**

Create `.github/workflows/docker-build-push-dockerhub.yml` with this exact content:

```yaml
name: Docker Build & Push (Docker Hub)

on:
  workflow_call:
    inputs:
      image-name:
        description: "Full image name (e.g. myorg/myapp)"
        required: true
        type: string
      tags:
        description: "Newline-separated list of tags (e.g. latest, abc1234, v1.0.0)"
        required: true
        type: string
      dockerfile:
        description: "Path to Dockerfile relative to context"
        required: false
        type: string
        default: "Dockerfile"
      context:
        description: "Docker build context path"
        required: false
        type: string
        default: "."
      platforms:
        description: "Comma-separated target platforms (e.g. linux/amd64,linux/arm64)"
        required: false
        type: string
        default: "linux/amd64"
      build-args:
        description: "Newline-separated build arguments (e.g. NODE_ENV=production)"
        required: false
        type: string
        default: ""
      push:
        description: "Whether to push the image after building (set false for build-only validation)"
        required: false
        type: boolean
        default: true
    secrets:
      username:
        description: "Docker Hub username"
        required: true
      token:
        description: "Docker Hub access token"
        required: true

jobs:
  build:
    name: Build & Push
    runs-on: ubuntu-latest
    steps:
      - name: Checkout caller repo
        uses: actions/checkout@v4

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        if: ${{ inputs.push }}
        uses: docker/login-action@v3
        with:
          registry: docker.io
          username: ${{ secrets.username }}
          password: ${{ secrets.token }}

      - name: Compute image tags
        id: tags
        env:
          IMAGE_NAME: ${{ inputs.image-name }}
          TAGS_INPUT: ${{ inputs.tags }}
        run: |
          FULL_TAGS=""
          while IFS= read -r tag; do
            tag=$(echo "$tag" | xargs)
            [ -z "$tag" ] && continue
            if [ -n "$FULL_TAGS" ]; then
              FULL_TAGS="${FULL_TAGS},"
            fi
            FULL_TAGS="${FULL_TAGS}${IMAGE_NAME}:${tag}"
          done <<< "$TAGS_INPUT"
          echo "tags=${FULL_TAGS}" >> "$GITHUB_OUTPUT"

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: ${{ inputs.context }}
          file: ${{ inputs.context }}/${{ inputs.dockerfile }}
          platforms: ${{ inputs.platforms }}
          push: ${{ inputs.push }}
          tags: ${{ steps.tags.outputs.tags }}
          build-args: ${{ inputs.build-args }}

      - name: Write job summary
        env:
          IMAGE_NAME: ${{ inputs.image-name }}
          TAGS_INPUT: ${{ inputs.tags }}
          PLATFORMS: ${{ inputs.platforms }}
          PUSH: ${{ inputs.push }}
        run: |
          {
            echo "## Docker Build & Push (Docker Hub)"
            echo ""
            if [ "$PUSH" = "true" ]; then
              echo "**Status**: Pushed"
            else
              echo "**Status**: Built (not pushed)"
            fi
            echo "**Image**: \`${IMAGE_NAME}\`"
            echo "**Platforms**: \`${PLATFORMS}\`"
            echo ""
            echo "### Tags"
            echo ""
            while IFS= read -r tag; do
              tag=$(echo "$tag" | xargs)
              [ -z "$tag" ] && continue
              echo "- \`${IMAGE_NAME}:${tag}\`"
            done <<< "$TAGS_INPUT"
          } >> "$GITHUB_STEP_SUMMARY"
```

- [ ] **Step 2: Validate YAML syntax**

Run:
```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/docker-build-push-dockerhub.yml'))" && echo "YAML valid"
```
Expected: `YAML valid`

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/docker-build-push-dockerhub.yml
git commit -m "feat: add Docker build & push workflow for Docker Hub

Requires username and token secrets from caller.
Supports multi-platform builds, caller-specified tags,
build args, and build-only validation mode."
```

---

### Task 3: Create the AWS ECR workflow

**Files:**
- Create: `.github/workflows/docker-build-push-ecr.yml`

- [ ] **Step 1: Create the workflow file**

Create `.github/workflows/docker-build-push-ecr.yml` with this exact content:

```yaml
name: Docker Build & Push (ECR)

on:
  workflow_call:
    inputs:
      image-name:
        description: "Full ECR image name (e.g. 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp)"
        required: true
        type: string
      aws-region:
        description: "AWS region for ECR (e.g. us-east-1)"
        required: true
        type: string
      tags:
        description: "Newline-separated list of tags (e.g. latest, abc1234, v1.0.0)"
        required: true
        type: string
      dockerfile:
        description: "Path to Dockerfile relative to context"
        required: false
        type: string
        default: "Dockerfile"
      context:
        description: "Docker build context path"
        required: false
        type: string
        default: "."
      platforms:
        description: "Comma-separated target platforms (e.g. linux/amd64,linux/arm64)"
        required: false
        type: string
        default: "linux/amd64"
      build-args:
        description: "Newline-separated build arguments (e.g. NODE_ENV=production)"
        required: false
        type: string
        default: ""
      push:
        description: "Whether to push the image after building (set false for build-only validation)"
        required: false
        type: boolean
        default: true
    secrets:
      aws-access-key-id:
        description: "AWS access key ID"
        required: true
      aws-secret-access-key:
        description: "AWS secret access key"
        required: true

jobs:
  build:
    name: Build & Push
    runs-on: ubuntu-latest
    steps:
      - name: Checkout caller repo
        uses: actions/checkout@v4

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

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

      - name: Compute image tags
        id: tags
        env:
          IMAGE_NAME: ${{ inputs.image-name }}
          TAGS_INPUT: ${{ inputs.tags }}
        run: |
          FULL_TAGS=""
          while IFS= read -r tag; do
            tag=$(echo "$tag" | xargs)
            [ -z "$tag" ] && continue
            if [ -n "$FULL_TAGS" ]; then
              FULL_TAGS="${FULL_TAGS},"
            fi
            FULL_TAGS="${FULL_TAGS}${IMAGE_NAME}:${tag}"
          done <<< "$TAGS_INPUT"
          echo "tags=${FULL_TAGS}" >> "$GITHUB_OUTPUT"

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: ${{ inputs.context }}
          file: ${{ inputs.context }}/${{ inputs.dockerfile }}
          platforms: ${{ inputs.platforms }}
          push: ${{ inputs.push }}
          tags: ${{ steps.tags.outputs.tags }}
          build-args: ${{ inputs.build-args }}

      - name: Write job summary
        env:
          IMAGE_NAME: ${{ inputs.image-name }}
          TAGS_INPUT: ${{ inputs.tags }}
          PLATFORMS: ${{ inputs.platforms }}
          PUSH: ${{ inputs.push }}
        run: |
          {
            echo "## Docker Build & Push (ECR)"
            echo ""
            if [ "$PUSH" = "true" ]; then
              echo "**Status**: Pushed"
            else
              echo "**Status**: Built (not pushed)"
            fi
            echo "**Image**: \`${IMAGE_NAME}\`"
            echo "**Platforms**: \`${PLATFORMS}\`"
            echo ""
            echo "### Tags"
            echo ""
            while IFS= read -r tag; do
              tag=$(echo "$tag" | xargs)
              [ -z "$tag" ] && continue
              echo "- \`${IMAGE_NAME}:${tag}\`"
            done <<< "$TAGS_INPUT"
          } >> "$GITHUB_STEP_SUMMARY"
```

- [ ] **Step 2: Validate YAML syntax**

Run:
```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/docker-build-push-ecr.yml'))" && echo "YAML valid"
```
Expected: `YAML valid`

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/docker-build-push-ecr.yml
git commit -m "feat: add Docker build & push workflow for AWS ECR

Uses AWS Access Key credentials. ECR repository must already exist.
Supports multi-platform builds, caller-specified tags,
build args, and build-only validation mode."
```

---

### Task 4: Create the Azure ACR workflow

**Files:**
- Create: `.github/workflows/docker-build-push-acr.yml`

- [ ] **Step 1: Create the workflow file**

Create `.github/workflows/docker-build-push-acr.yml` with this exact content:

```yaml
name: Docker Build & Push (ACR)

on:
  workflow_call:
    inputs:
      registry:
        description: "ACR registry URL (e.g. myregistry.azurecr.io)"
        required: true
        type: string
      image-name:
        description: "Full image name (e.g. myregistry.azurecr.io/myapp)"
        required: true
        type: string
      tags:
        description: "Newline-separated list of tags (e.g. latest, abc1234, v1.0.0)"
        required: true
        type: string
      dockerfile:
        description: "Path to Dockerfile relative to context"
        required: false
        type: string
        default: "Dockerfile"
      context:
        description: "Docker build context path"
        required: false
        type: string
        default: "."
      platforms:
        description: "Comma-separated target platforms (e.g. linux/amd64,linux/arm64)"
        required: false
        type: string
        default: "linux/amd64"
      build-args:
        description: "Newline-separated build arguments (e.g. NODE_ENV=production)"
        required: false
        type: string
        default: ""
      push:
        description: "Whether to push the image after building (set false for build-only validation)"
        required: false
        type: boolean
        default: true
    secrets:
      client-id:
        description: "Azure Service Principal application (client) ID"
        required: true
      client-secret:
        description: "Azure Service Principal password (client secret)"
        required: true

jobs:
  build:
    name: Build & Push
    runs-on: ubuntu-latest
    steps:
      - name: Checkout caller repo
        uses: actions/checkout@v4

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Azure ACR
        if: ${{ inputs.push }}
        uses: docker/login-action@v3
        with:
          registry: ${{ inputs.registry }}
          username: ${{ secrets.client-id }}
          password: ${{ secrets.client-secret }}

      - name: Compute image tags
        id: tags
        env:
          IMAGE_NAME: ${{ inputs.image-name }}
          TAGS_INPUT: ${{ inputs.tags }}
        run: |
          FULL_TAGS=""
          while IFS= read -r tag; do
            tag=$(echo "$tag" | xargs)
            [ -z "$tag" ] && continue
            if [ -n "$FULL_TAGS" ]; then
              FULL_TAGS="${FULL_TAGS},"
            fi
            FULL_TAGS="${FULL_TAGS}${IMAGE_NAME}:${tag}"
          done <<< "$TAGS_INPUT"
          echo "tags=${FULL_TAGS}" >> "$GITHUB_OUTPUT"

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: ${{ inputs.context }}
          file: ${{ inputs.context }}/${{ inputs.dockerfile }}
          platforms: ${{ inputs.platforms }}
          push: ${{ inputs.push }}
          tags: ${{ steps.tags.outputs.tags }}
          build-args: ${{ inputs.build-args }}

      - name: Write job summary
        env:
          IMAGE_NAME: ${{ inputs.image-name }}
          TAGS_INPUT: ${{ inputs.tags }}
          PLATFORMS: ${{ inputs.platforms }}
          PUSH: ${{ inputs.push }}
        run: |
          {
            echo "## Docker Build & Push (ACR)"
            echo ""
            if [ "$PUSH" = "true" ]; then
              echo "**Status**: Pushed"
            else
              echo "**Status**: Built (not pushed)"
            fi
            echo "**Image**: \`${IMAGE_NAME}\`"
            echo "**Platforms**: \`${PLATFORMS}\`"
            echo ""
            echo "### Tags"
            echo ""
            while IFS= read -r tag; do
              tag=$(echo "$tag" | xargs)
              [ -z "$tag" ] && continue
              echo "- \`${IMAGE_NAME}:${tag}\`"
            done <<< "$TAGS_INPUT"
          } >> "$GITHUB_STEP_SUMMARY"
```

- [ ] **Step 2: Validate YAML syntax**

Run:
```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/docker-build-push-acr.yml'))" && echo "YAML valid"
```
Expected: `YAML valid`

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/docker-build-push-acr.yml
git commit -m "feat: add Docker build & push workflow for Azure ACR

Uses Service Principal (client-id/client-secret) authentication.
Supports multi-platform builds, caller-specified tags,
build args, and build-only validation mode."
```

---

### Task 5: Create documentation

**Files:**
- Create: `docs/DOCKER.md`

- [ ] **Step 1: Create the documentation file**

Create `docs/DOCKER.md` with this exact content:

````markdown
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
````

- [ ] **Step 2: Commit**

```bash
git add docs/DOCKER.md
git commit -m "docs: add Docker build & push workflow documentation

Covers all 4 registry workflows (GHCR, Docker Hub, ECR, ACR)
with usage examples, inputs, secrets, and multi-platform info."
```

---

### Task 6: Validate all workflows

- [ ] **Step 1: Validate all YAML files**

Run:
```bash
for f in .github/workflows/docker-build-push-*.yml; do
  python3 -c "import yaml; yaml.safe_load(open('$f'))" && echo "$f: YAML valid"
done
```
Expected: 4 lines, each ending with `YAML valid`.

- [ ] **Step 2: Verify all files exist**

Run:
```bash
ls -la .github/workflows/docker-build-push-ghcr.yml
ls -la .github/workflows/docker-build-push-dockerhub.yml
ls -la .github/workflows/docker-build-push-ecr.yml
ls -la .github/workflows/docker-build-push-acr.yml
ls -la docs/DOCKER.md
```
Expected: All 5 files listed.

- [ ] **Step 3: Review git log**

Run:
```bash
git log --oneline -7
```
Expected: 5 new commits visible — one per workflow file, one for docs.
