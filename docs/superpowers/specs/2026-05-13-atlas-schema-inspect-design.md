# Atlas Schema Inspect Workflow + Custom Binary Migration

**Date:** 2026-05-13
**Status:** Draft

## Overview

Add an `atlas-schema-inspect.yml` reusable workflow that connects to a live database, extracts the current schema as HCL, and optionally commits it back to the repo. Additionally, migrate all Atlas workflows (plan, apply, inspect) from `ariga/setup-atlas@v0` to a custom binary from `NFUChen/atlas/releases`.

## Scope

1. **New workflow:** `atlas-schema-inspect.yml` — inspect live DB schema, output to summary + artifact, optionally commit HCL back to a target branch
2. **Custom binary migration:** Replace `ariga/setup-atlas@v0` in plan, apply, and inspect with a download step for the `NFUChen/atlas` binary, controlled by an `atlas-version` input
3. **Documentation:** Update `docs/ATLAS.md` and the `cloud-actions-atlas` skill
4. **README:** Add inspect workflow to the README workflow table

## Custom Atlas Binary Setup

All three Atlas workflows (plan, apply, inspect) share this setup pattern:

- **New input:** `atlas-version` (string, optional, default: `"v0.1.2"`)
- **Step:** Download `atlas-linux-amd64` from `https://github.com/NFUChen/atlas/releases/download/{atlas-version}/atlas-linux-amd64`, `chmod +x`, add to `$PATH`
- Replaces `ariga/setup-atlas@v0` in plan and apply workflows
- Runners are `ubuntu-latest` so always `linux-amd64`

## Inspect Workflow Design

### File

`.github/workflows/atlas-schema-inspect.yml`

### Trigger

`workflow_call` — called from caller repos, same pattern as plan/apply.

### Inputs

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `atlas-version` | string | no | `v0.1.2` | Atlas binary version tag from NFUChen/atlas releases |
| `dev-url` | string | no | `docker://postgres/15` | Dev database URL for schema normalization |
| `exclude` | string | no | `""` | Comma-separated glob patterns to exclude database objects |
| `schemas` | string | no | `""` | Comma-separated schema names to inspect (empty = all schemas) |
| `output-path` | string | no | `""` | If set, commits inspected HCL to this path in the repo |
| `target-branch` | string | no | `main` | Branch to commit to when output-path is set |

### Secrets

| Name | Required | Description |
|------|----------|-------------|
| `db-url` | yes | Target database connection URL |
| `wg-config-file` | no | WireGuard config file content for VPN tunnel |
| `pat` | no | Personal access token for pushing commits (required when output-path is set) |

### Steps

1. **Checkout caller repo** — `actions/checkout@v4`
2. **Set up WireGuard** — conditional on `wg-config-file` secret being provided
3. **Check database connectivity** — parse host/port from `db-url`, `nc` connectivity test (same as plan/apply)
4. **Download custom Atlas binary** — curl from `NFUChen/atlas/releases/download/{atlas-version}/atlas-linux-amd64`, chmod +x, add to PATH
5. **Run `atlas schema inspect`** — connect to live DB, output HCL
   - `--url "$DB_URL"`
   - `--dev-url "$DEV_URL"`
   - `--exclude` flags (parsed from comma-separated input)
   - `--schema` flags (parsed from comma-separated schemas input)
   - Capture HCL output to step output variable (use `|| true` to capture errors in output, consistent with plan workflow)
6. **Write job summary** — schema HCL in a fenced `hcl` code block with metadata header
7. **Save to file + upload artifact** — save as `atlas-schema-inspect.hcl`, upload with `actions/upload-artifact@v4`, 30-day retention
8. **Commit-back** — conditional: only runs when `output-path != ""`
   - Configure git remote with PAT for push authentication
   - Checkout `target-branch`
   - Write inspected HCL to `output-path`
   - `git add`, `git commit`, `git push` to `target-branch`

### Commit-back Details

When `output-path` is provided:
- The `pat` secret is required — workflow fails with a clear error if `output-path` is set but `pat` is missing
- Git remote is reconfigured to use the PAT: `https://x-access-token:{PAT}@github.com/{repo}.git`
- Commit message: `chore: update schema from atlas inspect`
- Push targets `target-branch` (default: `main`)
- If no schema changes are detected (file identical), the step skips the commit

## Changes to Existing Workflows

### atlas-schema-plan.yml

- Add `atlas-version` input (string, optional, default: `v0.1.2`)
- Replace `ariga/setup-atlas@v0` step with custom binary download step

### atlas-schema-apply.yml

- Add `atlas-version` input (string, optional, default: `v0.1.2`)
- Replace `ariga/setup-atlas@v0` step with custom binary download step

## Documentation Updates

### docs/ATLAS.md

- Add inspect workflow to the workflow table
- Add usage section for inspect (audit mode and bootstrap mode)
- Add `atlas-version` input to the inputs table
- Add `pat` secret to the secrets table
- Update "How It Works" diagram to include inspect

### cloud-actions-atlas skill

- Add Schema Inspect section with inputs, secrets, and caller workflow reference
- Add `atlas-version` input documentation
- Update generation guidelines for inspect workflows

### README.md

- Add `atlas-schema-inspect.yml` row to the workflows table

## Caller Workflow Examples

### Audit — Manual trigger to view current schema

```yaml
name: DB Schema Inspect

on:
  workflow_dispatch:

jobs:
  inspect:
    uses: NFUChen/cloud-actions/.github/workflows/atlas-schema-inspect.yml@main
    with:
      schemas: "public"
    secrets:
      db-url: ${{ secrets.DB_URL }}
      wg-config-file: ${{ secrets.WG_CONFIG_FILE }}
```

### Bootstrap — Inspect and commit HCL to repo

```yaml
name: DB Schema Bootstrap

on:
  workflow_dispatch:

jobs:
  bootstrap:
    uses: NFUChen/cloud-actions/.github/workflows/atlas-schema-inspect.yml@main
    with:
      schemas: "public"
      output-path: "schema/schema.hcl"
      target-branch: "main"
    secrets:
      db-url: ${{ secrets.DB_URL }}
      pat: ${{ secrets.PAT }}
      wg-config-file: ${{ secrets.WG_CONFIG_FILE }}
```
