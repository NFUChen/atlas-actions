# K8s Deploy Reusable Workflows - Design Spec

## Overview

Two separate reusable GitHub Actions workflows for deploying to Kubernetes clusters: one for raw manifests (`kubectl apply`) and one for Helm charts (`helm upgrade --install`). Both authenticate via a base64-encoded kubeconfig secret, with optional WireGuard VPN support. Follows the same standalone `workflow_call` pattern established by the Atlas and Docker workflows.

## Workflow Files

| File | Purpose |
|------|---------|
| `.github/workflows/k8s-deploy-manifest.yml` | Deploy raw K8s manifests via `kubectl apply` |
| `.github/workflows/k8s-deploy-helm.yml` | Deploy Helm chart via `helm upgrade --install` |

---

## Workflow 1: Manifest Deploy (`k8s-deploy-manifest.yml`)

### Inputs

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `manifest-path` | string | yes | -- | Path to manifest file or directory (relative to repo root) |
| `namespace` | string | no | `default` | Target K8s namespace |
| `kubectl-version` | string | no | `latest` | kubectl version to install |

### Secrets

| Name | Required | Description |
|------|----------|-------------|
| `kubeconfig` | yes | Full kubeconfig file content (base64 encoded) |
| `wg-config-file` | no | WireGuard configuration file content for VPN tunnel |

### Steps

1. **Checkout caller repo** -- `actions/checkout@v4`
2. **Set up WireGuard connection** -- `niklaskeerl/easy-wireguard-action@v2`, skipped if `wg-config-file` not provided
3. **Set up kubectl** -- `azure/setup-kubectl@v4` with specified version
4. **Write kubeconfig** -- decode base64 secret and write to `~/.kube/config`
5. **Verify cluster connectivity** -- `kubectl cluster-info`, fail fast if unreachable
6. **Apply manifests** -- `kubectl apply -f <manifest-path> -n <namespace>`
7. **Write job summary** -- displays namespace, manifest path, and applied output

### Caller Example

```yaml
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

---

## Workflow 2: Helm Deploy (`k8s-deploy-helm.yml`)

### Inputs

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `chart-path` | string | yes | -- | Path to Helm chart directory (relative to repo root) |
| `release-name` | string | yes | -- | Helm release name |
| `namespace` | string | no | `default` | Target K8s namespace |
| `values-file` | string | no | `""` | Path to values file (relative to repo root) |
| `set-values` | string | no | `""` | Newline-separated `key=value` pairs passed as `--set` flags |
| `helm-version` | string | no | `latest` | Helm version to install |

### Secrets

| Name | Required | Description |
|------|----------|-------------|
| `kubeconfig` | yes | Full kubeconfig file content (base64 encoded) |
| `wg-config-file` | no | WireGuard configuration file content for VPN tunnel |

### Steps

1. **Checkout caller repo** -- `actions/checkout@v4`
2. **Set up WireGuard connection** -- `niklaskeerl/easy-wireguard-action@v2`, skipped if `wg-config-file` not provided
3. **Set up Helm** -- `azure/setup-helm@v4` with specified version
4. **Write kubeconfig** -- decode base64 secret and write to `~/.kube/config`
5. **Verify cluster connectivity** -- `kubectl cluster-info` (kubectl is pre-installed on ubuntu-latest), fail fast if unreachable
6. **Deploy with Helm** -- `helm upgrade --install <release-name> <chart-path> -n <namespace> --create-namespace [-f values-file] [--set ...]`
7. **Write job summary** -- displays release name, namespace, chart path, revision, and status

### Caller Example

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

---

## Key Design Decisions

1. **Separate workflows per deploy method**: Manifest and Helm have different inputs and semantics. Separate files keep each workflow simple and explicit, consistent with the per-registry Docker pattern.
2. **kubeconfig as base64 secret**: Most universal authentication method. Works with any K8s cluster regardless of provider. The caller base64-encodes the kubeconfig and stores it as a repo secret.
3. **Optional WireGuard VPN**: Same pattern as Atlas workflows. Skipped when secret is not provided.
4. **Cluster connectivity check**: `kubectl cluster-info` runs before deploy to fail fast with a clear error if the cluster is unreachable.
5. **`--create-namespace` on Helm**: Automatically creates the target namespace if it doesn't exist, avoiding a common deployment failure.
6. **No `--wait` by default**: Keeps deploy fast. Callers who need to wait for rollout can add post-deploy steps in their own workflow.
7. **Caller-specified tool versions**: Allows pinning kubectl/helm versions for reproducibility. Defaults to `latest`.
8. **Newline-separated `set-values`**: Each line becomes a `--set` flag. Same pattern as Docker's `build-args` input.
9. **No dry-run step**: Per the "deploy only" requirement. Callers who need preview can run `kubectl diff` or `helm diff` in a separate workflow step before calling the deploy workflow.

## Actions Used

| Action | Version | Used In | Purpose |
|--------|---------|---------|---------|
| `actions/checkout` | v4 | Both | Checkout caller repo |
| `niklaskeerl/easy-wireguard-action` | v2 | Both | Optional VPN tunnel |
| `azure/setup-kubectl` | v4 | Manifest | Install kubectl |
| `azure/setup-helm` | v4 | Helm | Install Helm |
