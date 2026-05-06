# K8s Actions

Reusable GitHub Actions workflows for deploying to Kubernetes clusters. One workflow per deployment method.

## Workflows

| Workflow | Method | Description |
|----------|--------|-------------|
| `k8s-deploy-manifest.yml` | kubectl apply | Deploy raw K8s manifest files or directories |
| `k8s-deploy-helm.yml` | helm upgrade --install | Deploy Helm charts with values and overrides |

## Prerequisites

- A Kubernetes cluster accessible from GitHub Actions runners (or via WireGuard VPN)
- A base64-encoded kubeconfig stored as a repo secret
- Manifest files or a Helm chart in your repo

### Encoding kubeconfig

```bash
cat ~/.kube/config | base64 | pbcopy  # macOS
cat ~/.kube/config | base64 -w 0      # Linux
```

Store the output as a repo secret named `KUBECONFIG`.

## Usage

### Manifest Deploy

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
```

### Manifest Deploy with VPN

```yaml
jobs:
  deploy:
    uses: NFUChen/cloud-actions/.github/workflows/k8s-deploy-manifest.yml@main
    with:
      manifest-path: "k8s/deployment.yaml"
      namespace: staging
    secrets:
      kubeconfig: ${{ secrets.KUBECONFIG }}
      wg-config-file: ${{ secrets.WG_CONFIG_FILE }}
```

### Helm Deploy

```yaml
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

### Helm Deploy — Minimal

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

## Inputs

### Manifest Workflow

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `manifest-path` | string | yes | — | Path to manifest file or directory |
| `namespace` | string | no | `default` | Target K8s namespace |
| `kubectl-version` | string | no | `latest` | kubectl version to install |

### Helm Workflow

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `chart-path` | string | yes | — | Path to Helm chart directory |
| `release-name` | string | yes | — | Helm release name |
| `namespace` | string | no | `default` | Target K8s namespace |
| `values-file` | string | no | `""` | Path to values file |
| `set-values` | string | no | `""` | Newline-separated `key=value` pairs for `--set` |
| `helm-version` | string | no | `latest` | Helm version to install |

## Secrets

Both workflows use the same secrets:

| Name | Required | Description |
|------|----------|-------------|
| `kubeconfig` | yes | Full kubeconfig file content (base64 encoded) |
| `wg-config-file` | no | WireGuard config file content for VPN tunnel |

## How It Works

```
Push to main
  └─► k8s-deploy-manifest runs
        ├─ WireGuard VPN (if configured)
        ├─ kubectl cluster-info (connectivity check)
        ├─ kubectl apply -f <path> -n <namespace>
        └─ writes result to job summary

  └─► k8s-deploy-helm runs
        ├─ WireGuard VPN (if configured)
        ├─ kubectl cluster-info (connectivity check)
        ├─ helm upgrade --install <release> <chart> -n <namespace>
        └─ writes result to job summary
```

## Pinning Tool Versions

Both workflows accept a version input for reproducibility:

```yaml
with:
  kubectl-version: "v1.31.0"   # manifest workflow
  helm-version: "v3.16.0"      # helm workflow
```

Default is `latest` for both.

## Helm Values

### Via values file

```yaml
with:
  values-file: "charts/myapp/values-prod.yaml"
```

### Via --set overrides

One `key=value` pair per line:

```yaml
with:
  set-values: |
    image.tag=abc1234
    replicas=3
    ingress.enabled=true
```

### Both together

```yaml
with:
  values-file: "charts/myapp/values-prod.yaml"
  set-values: |
    image.tag=${{ github.sha }}
```

`--set` values override values file entries, matching standard Helm behavior.
