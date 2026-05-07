# Cloud Actions

Reusable GitHub Actions workflows and composite actions for common cloud infrastructure tasks. Workflows are standalone `workflow_call` entries; composite actions are step-level building blocks you compose in your own jobs.

## Workflows

| Domain | Workflow | Description |
|--------|----------|-------------|
| **Database** | `atlas-schema-plan.yml` | Preview Atlas schema diff on PR |
| **Database** | `atlas-schema-apply.yml` | Apply Atlas schema changes |
| **Docker** | `docker-build-push-ghcr.yml` | Build & push to GitHub Container Registry |
| **Docker** | `docker-build-push-dockerhub.yml` | Build & push to Docker Hub |
| **Docker** | `docker-build-push-ecr.yml` | Build & push to AWS ECR |
| **Docker** | `docker-build-push-acr.yml` | Build & push to Azure ACR |
| **K8s** | `k8s-deploy-manifest.yml` | Deploy raw K8s manifests via `kubectl apply` |
| **K8s** | `k8s-deploy-helm.yml` | Deploy Helm charts via `helm upgrade --install` |
| **Config** | `actions/config-fetch-s3` | Fetch config files from AWS S3 |
| **Config** | `actions/config-fetch-azure-blob` | Fetch config files from Azure Blob Storage |
| **Config** | `actions/config-fetch-gcs` | Fetch config files from Google Cloud Storage |
| **Config** | `actions/config-fetch-git` | Fetch config files from private Git repo |

## Quick Start

Reference any workflow from your repo:

```yaml
# .github/workflows/deploy.yml
name: Deploy

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

  deploy:
    needs: build
    uses: NFUChen/cloud-actions/.github/workflows/k8s-deploy-helm.yml@main
    with:
      chart-path: "charts/myapp"
      release-name: myapp
      namespace: production
      set-values: |
        image.tag=${{ github.sha }}
    secrets:
      kubeconfig: ${{ secrets.KUBECONFIG }}
```

See the detailed docs for each domain:

- [Atlas (Database Schema)](docs/ATLAS.md)
- [Docker (Build & Push)](docs/DOCKER.md)
- [K8s (Deployment)](docs/K8S.md)
- [Config Fetch](docs/CONFIG-FETCH.md)

## Installing Skills for AI Agents

This repo ships with [Claude Code skills](https://docs.anthropic.com/en/docs/claude-code) that help AI agents generate the correct caller workflow YAML. When installed, your agent understands all available workflows, inputs, secrets, and common patterns -- so you can describe what you need in natural language and get the right YAML.

### Available Skills

| Skill | Triggers on |
|-------|-------------|
| `cloud-actions-atlas` | Database schema, Atlas, migrations, schema diff/apply |
| `cloud-actions-docker` | Docker build, container image, push to registry |
| `cloud-actions-k8s` | K8s deploy, kubectl apply, Helm upgrade, cluster deployment |
| `cloud-actions-config-fetch` | Config fetch, download config, S3 config, Azure Blob config, GCS config, Git config |

### Installation

#### Option 1: Add as a git submodule (recommended)

This keeps the skills in sync with the workflows as they evolve.

```bash
# From your project root
git submodule add https://github.com/NFUChen/cloud-actions.git .cloud-actions

# Symlink the skills into your project's Claude skills directory
mkdir -p .claude/skills
ln -s ../../.cloud-actions/.claude/skills/cloud-actions-atlas .claude/skills/cloud-actions-atlas
ln -s ../../.cloud-actions/.claude/skills/cloud-actions-docker .claude/skills/cloud-actions-docker
ln -s ../../.cloud-actions/.claude/skills/cloud-actions-k8s .claude/skills/cloud-actions-k8s
ln -s ../../.cloud-actions/.claude/skills/cloud-actions-config-fetch .claude/skills/cloud-actions-config-fetch
```

To update later:

```bash
git submodule update --remote .cloud-actions
```

#### Option 2: Copy the skill files directly

```bash
# Clone and copy
git clone https://github.com/NFUChen/cloud-actions.git /tmp/cloud-actions

# Copy the skills you need into your project
mkdir -p .claude/skills
cp -r /tmp/cloud-actions/.claude/skills/cloud-actions-atlas .claude/skills/
cp -r /tmp/cloud-actions/.claude/skills/cloud-actions-docker .claude/skills/
cp -r /tmp/cloud-actions/.claude/skills/cloud-actions-k8s .claude/skills/
cp -r /tmp/cloud-actions/.claude/skills/cloud-actions-config-fetch .claude/skills/

rm -rf /tmp/cloud-actions
```

#### Option 3: Install globally (for personal use across all projects)

```bash
git clone https://github.com/NFUChen/cloud-actions.git /tmp/cloud-actions

# Copy to your global Claude skills directory
cp -r /tmp/cloud-actions/.claude/skills/cloud-actions-atlas ~/.claude/skills/
cp -r /tmp/cloud-actions/.claude/skills/cloud-actions-docker ~/.claude/skills/
cp -r /tmp/cloud-actions/.claude/skills/cloud-actions-k8s ~/.claude/skills/
cp -r /tmp/cloud-actions/.claude/skills/cloud-actions-config-fetch ~/.claude/skills/

rm -rf /tmp/cloud-actions
```

### Usage

Once installed, just describe what you need to your AI agent:

> "Set up Docker build to GHCR with multi-platform support"

> "I need a Helm deploy workflow for my charts/api chart to the staging namespace"

> "Set up Atlas schema plan on PR and manual apply for my PostgreSQL schema"

The agent will generate the complete caller workflow YAML with the correct inputs, secrets, and triggers.
