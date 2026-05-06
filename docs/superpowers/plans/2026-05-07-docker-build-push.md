# Docker Build & Push Reusable Workflow — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a reusable GitHub Actions workflow that builds and optionally pushes Docker images to any container registry, with GHCR auto-auth, multi-platform support, and caller-specified tags.

**Architecture:** Single `workflow_call` workflow using the official Docker GitHub Actions (`setup-qemu-action`, `setup-buildx-action`, `login-action`, `build-push-action`). Auth conditionally uses `GITHUB_TOKEN` for GHCR or caller-provided secrets for other registries. A `push` boolean input controls whether images are pushed or just built for validation.

**Tech Stack:** GitHub Actions YAML, Docker Buildx, QEMU

**Spec:** `docs/superpowers/specs/2026-05-07-docker-build-push-design.md`

---

## File Structure

| File | Action | Responsibility |
|------|--------|----------------|
| `.github/workflows/docker-build-push.yml` | Create | The reusable workflow |
| `docs/DOCKER.md` | Create | Usage documentation (mirrors `docs/ATLAS.md` pattern) |

---

### Task 1: Create the reusable workflow

**Files:**
- Create: `.github/workflows/docker-build-push.yml`

- [ ] **Step 1: Create the workflow file with inputs, secrets, and the build job**

Create `.github/workflows/docker-build-push.yml` with this exact content:

```yaml
name: Docker Build & Push

on:
  workflow_call:
    inputs:
      registry:
        description: "Container registry URL (e.g. ghcr.io, docker.io)"
        required: true
        type: string
      image-name:
        description: "Full image name (e.g. ghcr.io/org/app or myorg/myapp)"
        required: true
        type: string
      tags:
        description: "Newline-separated list of tags to apply (e.g. latest, abc1234, v1.0.0)"
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
      registry-username:
        description: "Registry username (not needed for GHCR)"
        required: false
      registry-password:
        description: "Registry password or access token (not needed for GHCR)"
        required: false

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
        if: ${{ inputs.push && inputs.registry == 'ghcr.io' }}
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Login to registry
        if: ${{ inputs.push && inputs.registry != 'ghcr.io' }}
        uses: docker/login-action@v3
        with:
          registry: ${{ inputs.registry }}
          username: ${{ secrets.registry-username }}
          password: ${{ secrets.registry-password }}

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
            echo "## Docker Build & Push"
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

- [ ] **Step 2: Validate the YAML syntax**

Run:
```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/docker-build-push.yml'))" && echo "YAML valid"
```
Expected: `YAML valid`

- [ ] **Step 3: Commit the workflow**

```bash
git add .github/workflows/docker-build-push.yml
git commit -m "feat: add Docker build & push reusable workflow

Supports any container registry with GHCR auto-auth,
multi-platform builds, caller-specified tags, build args,
and build-only validation mode via push boolean."
```

---

### Task 2: Create documentation

**Files:**
- Create: `docs/DOCKER.md`

- [ ] **Step 1: Create the documentation file**

Create `docs/DOCKER.md` with this exact content:

```markdown
# Docker Actions

Reusable GitHub Actions workflow for building and pushing Docker images to any container registry.

## Workflow

| Workflow | Purpose |
|----------|---------|
| `docker-build-push.yml` | Build and optionally push Docker images |

## Usage

### Prerequisites

- A `Dockerfile` in your repo
- (For push) Registry credentials configured as secrets, unless using GHCR with `GITHUB_TOKEN`

### GHCR — Build on PR, Push on Merge

```yaml
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

No secrets needed — GHCR uses `GITHUB_TOKEN` automatically.

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

### Build-Only Validation (No Push)

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

## Inputs

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `registry` | string | yes | — | Registry URL (e.g. `ghcr.io`, `docker.io`) |
| `image-name` | string | yes | — | Full image name (e.g. `ghcr.io/org/app`) |
| `tags` | string | yes | — | Newline-separated list of tags |
| `dockerfile` | string | no | `Dockerfile` | Path to Dockerfile relative to context |
| `context` | string | no | `.` | Docker build context path |
| `platforms` | string | no | `linux/amd64` | Comma-separated target platforms |
| `build-args` | string | no | `""` | Newline-separated build arguments |
| `push` | boolean | no | `true` | Push image after building |

## Secrets

| Name | Required | Description |
|------|----------|-------------|
| `registry-username` | no | Registry username (not needed for GHCR) |
| `registry-password` | no | Registry password/token (not needed for GHCR) |

## Registry Authentication

- **GHCR (`ghcr.io`)**: Automatic via `GITHUB_TOKEN`. No secrets needed from the caller. The calling repo must have `packages: write` permission.
- **All other registries**: Pass `registry-username` and `registry-password` secrets.
- **Build-only mode** (`push: false`): No login is performed regardless of registry.

## Multi-Platform Builds

Defaults to `linux/amd64`. Add platforms via the `platforms` input:

```yaml
with:
  platforms: linux/amd64,linux/arm64
```

Uses QEMU emulation under the hood. Multi-platform builds are slower than single-platform.

## Build Arguments

Pass build-time variables via `build-args`, one per line:

```yaml
with:
  build-args: |
    NODE_ENV=production
    APP_VERSION=1.0.0
    GIT_SHA=${{ github.sha }}
```
```

- [ ] **Step 2: Commit the documentation**

```bash
git add docs/DOCKER.md
git commit -m "docs: add Docker build & push workflow documentation"
```

---

### Task 3: Verify everything

- [ ] **Step 1: Verify all files exist and are valid**

```bash
ls -la .github/workflows/docker-build-push.yml
ls -la docs/DOCKER.md
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/docker-build-push.yml'))" && echo "YAML valid"
```
Expected: Both files listed, `YAML valid` printed.

- [ ] **Step 2: Review git log**

```bash
git log --oneline -5
```
Expected: Two new commits visible — one for the workflow, one for the docs.
