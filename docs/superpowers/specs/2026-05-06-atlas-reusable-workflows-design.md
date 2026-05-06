# Atlas Declarative Schema Reusable Workflows

## Overview

Two cross-repo reusable GitHub Actions workflows for Atlas declarative schema management:

- **schema-plan.yml** — Generates and displays schema migration plan
- **schema-apply.yml** — Applies schema changes to the target database

## Architecture

```
atlas-actions/                      (this repo)
└── .github/workflows/
    ├── schema-plan.yml             # Reusable: plan
    └── schema-apply.yml            # Reusable: apply

caller-repo/                        (any consuming repo)
├── schema/schema.hcl               # Atlas HCL desired state
└── .github/workflows/
    ├── db-plan.yml                  # Calls schema-plan on PR
    └── db-apply.yml                 # Calls schema-apply via workflow_dispatch
```

## Workflow: schema-plan.yml

**Trigger**: `workflow_call`

**Inputs**:

| Name          | Type   | Required | Default              | Description                          |
|---------------|--------|----------|----------------------|--------------------------------------|
| `schema-file` | string | yes      | —                    | HCL file path relative to repo root |
| `dev-url`     | string | no       | `docker://postgres/15` | Atlas dev database URL             |

**Secrets**:

| Name     | Required | Description                |
|----------|----------|----------------------------|
| `db-url` | yes      | Target database connection |

**Steps**:

1. `actions/checkout@v4` — checkout caller repo
2. `ariga/setup-atlas@v0` — install Atlas CLI
3. Run `atlas schema diff` with `--from "$db-url"`, `--to "file://$schema-file"`, `--dev-url "$dev-url"` to generate the plan
4. Write plan output to `$GITHUB_STEP_SUMMARY`
5. Save plan to file, upload as artifact via `actions/upload-artifact@v4`

## Workflow: schema-apply.yml

**Trigger**: `workflow_call`

**Inputs**: Same as plan (`schema-file`, `dev-url`)

**Secrets**: Same as plan (`db-url`)

**Steps**:

1. `actions/checkout@v4` — checkout caller repo
2. `ariga/setup-atlas@v0` — install Atlas CLI
3. Run `atlas schema apply --dry-run` to preview changes, write to `$GITHUB_STEP_SUMMARY`
4. Run `atlas schema apply --auto-approve` to execute

## Security

- `db-url` passed exclusively via `secrets`, never exposed in logs
- `dev-url` defaults to `docker://postgres/15` (ephemeral container)
- Apply runs dry-run first, then auto-approve (human gate is the manual `workflow_dispatch`)

## Caller Usage Example

```yaml
# Plan on PR
on:
  pull_request:
    paths: ['schema/**']
jobs:
  plan:
    uses: NFUChen/atlas-actions/.github/workflows/schema-plan.yml@main
    with:
      schema-file: "schema/schema.hcl"
    secrets:
      db-url: ${{ secrets.DB_URL }}

# Apply manually
on:
  workflow_dispatch:
jobs:
  apply:
    uses: NFUChen/atlas-actions/.github/workflows/schema-apply.yml@main
    with:
      schema-file: "schema/schema.hcl"
    secrets:
      db-url: ${{ secrets.DB_URL }}
```
