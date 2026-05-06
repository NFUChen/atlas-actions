---
name: cloud-actions-k8s
description: Generate GitHub Actions caller workflows for Kubernetes deployment via kubectl manifests or Helm charts. Use this skill whenever the user mentions K8s deployment, Kubernetes CI/CD, kubectl apply, Helm deploy, Helm upgrade, deploying to a cluster, or wants to set up automated Kubernetes deployment pipelines in GitHub Actions. Also use when the user asks about cloud-actions K8s workflows, container orchestration CI/CD, or deploying apps to Kubernetes.
---

# K8s Deploy Workflows

Generate caller workflow YAML for the `NFUChen/cloud-actions` K8s reusable workflows. Two workflows: one for raw manifests, one for Helm charts.

## Available Workflows

| Workflow | File | Method |
|----------|------|--------|
| Manifest | `k8s-deploy-manifest.yml` | `kubectl apply` -- deploy raw YAML manifests |
| Helm | `k8s-deploy-helm.yml` | `helm upgrade --install` -- deploy Helm charts |

## Choosing the Right Workflow

- **Manifest**: User has raw K8s YAML files (deployments, services, configmaps, etc.)
- **Helm**: User has a Helm chart directory with `Chart.yaml`, templates, and values files

If the user isn't sure, ask what's in their repo -- a `charts/` directory points to Helm, a `k8s/` or `manifests/` directory with plain YAML points to manifest.

## Base URL

```
NFUChen/cloud-actions/.github/workflows/<workflow-file>@main
```

## Shared Secrets (both workflows)

| Name | Required | Description |
|------|----------|-------------|
| `kubeconfig` | yes | Full kubeconfig file content (base64 encoded) |
| `wg-config-file` | no | WireGuard config file content for VPN tunnel |

The caller must base64-encode their kubeconfig:
```bash
cat ~/.kube/config | base64 | pbcopy  # macOS
cat ~/.kube/config | base64 -w 0      # Linux
```

## Manifest Workflow Reference

### Inputs

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `manifest-path` | string | yes | -- | Path to manifest file or directory |
| `namespace` | string | no | `default` | Target K8s namespace |
| `kubectl-version` | string | no | `latest` | kubectl version to install |

### Example

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    uses: NFUChen/cloud-actions/.github/workflows/k8s-deploy-manifest.yml@main
    with:
      manifest-path: "k8s/"
      namespace: production
    secrets:
      kubeconfig: ${{ secrets.KUBECONFIG }}
      wg-config-file: ${{ secrets.WG_CONFIG_FILE }}
```

## Helm Workflow Reference

### Inputs

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `chart-path` | string | yes | -- | Path to Helm chart directory |
| `release-name` | string | yes | -- | Helm release name |
| `namespace` | string | no | `default` | Target K8s namespace |
| `values-file` | string | no | `""` | Path to values file |
| `set-values` | string | no | `""` | Newline-separated `key=value` pairs for `--set` |
| `helm-version` | string | no | `latest` | Helm version to install |

### Example -- Full

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    uses: NFUChen/cloud-actions/.github/workflows/k8s-deploy-helm.yml@main
    with:
      chart-path: "charts/myapp"
      release-name: myapp
      namespace: production
      values-file: "charts/myapp/values-prod.yaml"
      set-values: |
        image.tag=${{ github.sha }}
        replicas=3
    secrets:
      kubeconfig: ${{ secrets.KUBECONFIG }}
      wg-config-file: ${{ secrets.WG_CONFIG_FILE }}
```

### Example -- Minimal

```yaml
jobs:
  deploy:
    uses: NFUChen/cloud-actions/.github/workflows/k8s-deploy-helm.yml@main
    with:
      chart-path: "charts/myapp"
      release-name: myapp
    secrets:
      kubeconfig: ${{ secrets.KUBECONFIG }}
```

## Common Patterns

### Helm Values Override

Values file + `--set` overrides (set takes precedence):

```yaml
with:
  values-file: "charts/myapp/values-prod.yaml"
  set-values: |
    image.tag=${{ github.sha }}
```

### Pinning Tool Versions

```yaml
with:
  kubectl-version: "v1.31.0"   # manifest workflow
  helm-version: "v3.16.0"      # helm workflow
```

### Docker Build + K8s Deploy Pipeline

A common pattern: build the image first, then deploy with the new tag.

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

## Generation Guidelines

When generating caller workflows:

- Ask whether the user has raw manifests or a Helm chart if not clear
- Always ask for the namespace -- don't assume `default`
- Include `wg-config-file` only if user mentions VPN or private cluster
- For Helm, always ask for the release name -- it's required and must be meaningful
- If user mentions Docker build + deploy, generate the combined pipeline pattern with `needs: build`
- Remind user to base64-encode their kubeconfig and store it as `KUBECONFIG` secret
