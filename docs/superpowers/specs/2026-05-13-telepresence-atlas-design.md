# Telepresence Support for Atlas Workflows

**Date:** 2026-05-13
**Status:** Draft

## Overview

Add Telepresence as a network connectivity option alongside WireGuard in all 3 Atlas workflows (plan, apply, inspect). This enables accessing databases that are only reachable inside a Kubernetes cluster. The two tunnel options are mutually exclusive.

## Scope

1. **New composite action:** `.github/actions/telepresence/action.yml` — install Telepresence CLI, write kubeconfig, connect with optional `--also-proxy` CIDRs
2. **Workflow changes:** Add `kubeconfig` secret, `also-proxy` input, Telepresence step, and mutual exclusion validation to all 3 Atlas workflows
3. **Documentation:** Update `docs/ATLAS.md` and the `cloud-actions-atlas` skill

## Telepresence Composite Action

### File

`.github/actions/telepresence/action.yml`

### Inputs

| Name | Required | Description |
|------|----------|-------------|
| `kubeconfig` | yes | Kubeconfig file content as a string |
| `also-proxy` | no | Comma-separated CIDR ranges to also proxy through the Telepresence tunnel (e.g. `10.0.0.0/8,172.16.0.0/12`). Each CIDR becomes a separate `--also-proxy` flag. |

### Steps

1. Install Telepresence CLI — download via the official install script (`curl -fL https://app.getambassador.io/download/tel2oss/releases/download/<version>/telepresence-linux-amd64 -o /usr/local/bin/telepresence`)
2. Write `kubeconfig` input content to `~/.kube/config`, set permissions to `600`
3. Run `telepresence connect` with `--also-proxy` flags parsed from the comma-separated input
4. Verify connection with `telepresence status`

## Atlas Workflow Changes

All 3 workflows (`atlas-schema-plan.yml`, `atlas-schema-apply.yml`, `atlas-schema-inspect.yml`) get identical changes:

### New Secret

| Name | Required | Description |
|------|----------|-------------|
| `kubeconfig` | no | Kubeconfig file content for Telepresence connection to K8s cluster |

### New Input

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `also-proxy` | string | no | `""` | Comma-separated CIDR ranges to also proxy through Telepresence |

### New Steps

**1. Validate network config** (first step after checkout, before any VPN setup):

Checks that `wg-config-file` and `kubeconfig` are not both provided. Fails with a clear error if both are set.

**2. Set up Telepresence connection** (after WireGuard step, before connectivity check):

Conditional on `kubeconfig` secret being provided. Calls the new `.github/actions/telepresence` composite action with `kubeconfig` and `also-proxy` inputs.

### Step Order (updated)

1. Checkout caller repo
2. **Validate network config** (new — mutual exclusion check)
3. Set up WireGuard connection (conditional)
4. **Set up Telepresence connection** (new — conditional)
5. Check database connectivity
6. Download Atlas binary
7. (remaining steps unchanged)

## Documentation Updates

### docs/ATLAS.md

- Add `kubeconfig` to secrets table
- Add `also-proxy` to inputs table
- Add Telepresence usage example in a new "Connecting via Telepresence" section
- Note mutual exclusion between WireGuard and Telepresence

### cloud-actions-atlas skill (SKILL.md)

- Add `kubeconfig` to secrets detail table
- Add `also-proxy` to inputs detail table
- Add Telepresence to caller workflow reference examples
- Update generation guidelines to mention Telepresence option
- Update prerequisites to mention `KUBECONFIG` secret option

## Caller Workflow Example

### Using Telepresence to access a cluster-internal database

```yaml
name: DB Schema Plan

on:
  pull_request:
    paths:
      - 'schema/**'

jobs:
  plan:
    uses: NFUChen/cloud-actions/.github/workflows/atlas-schema-plan.yml@main
    with:
      schema-path: "schema/schema.hcl"
      also-proxy: "10.0.0.0/8"
    secrets:
      db-url: ${{ secrets.DB_URL }}
      kubeconfig: ${{ secrets.KUBECONFIG }}
```

## Mutual Exclusion

- If only `wg-config-file` is provided → WireGuard tunnel
- If only `kubeconfig` is provided → Telepresence tunnel
- If both are provided → fail with error
- If neither is provided → direct connection (no tunnel)
