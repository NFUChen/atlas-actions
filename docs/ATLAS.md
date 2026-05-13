# Atlas Actions

Reusable GitHub Actions workflows for [Atlas](https://atlasgo.io/) declarative schema management.

## Workflows

| Workflow | Purpose |
|----------|---------|
| `schema-plan.yml` | Generate and preview schema migration plan |
| `schema-apply.yml` | Apply schema changes to the target database |
| `schema-inspect.yml` | Inspect live database schema, optionally commit HCL to repo |

## Usage

### Prerequisites

- An Atlas HCL schema file or directory in your repo (e.g. `schema/schema.hcl` or `schema/`)
- A `DB_URL` secret configured in your repo with the target database connection string
- (Optional) A `WG_CONFIG_FILE` secret with WireGuard config if the database is behind a VPN

### 1. Plan on Pull Request

Create `.github/workflows/db-plan.yml` in your repo:

```yaml
name: DB Schema Plan

on:
  pull_request:
    paths:
      - 'schema/**'

jobs:
  plan:
    uses: NFUChen/atlas-actions/.github/workflows/schema-plan.yml@main
    with:
      schema-path: "schema/schema.hcl"
    secrets:
      db-url: ${{ secrets.DB_URL }}
      wg-config-file: ${{ secrets.WG_CONFIG_FILE }}
```

The plan output will appear in the GitHub Actions **job summary** and be uploaded as an **artifact**.

### 2. Apply via Manual Trigger

Create `.github/workflows/db-apply.yml` in your repo:

```yaml
name: DB Schema Apply

on:
  workflow_dispatch:

jobs:
  apply:
    uses: NFUChen/atlas-actions/.github/workflows/schema-apply.yml@main
    with:
      schema-path: "schema/schema.hcl"
    secrets:
      db-url: ${{ secrets.DB_URL }}
      wg-config-file: ${{ secrets.WG_CONFIG_FILE }}
```

Go to **Actions > DB Schema Apply > Run workflow** to trigger manually.

### 3. Inspect via Manual Trigger (Audit)

Create `.github/workflows/db-inspect.yml` in your repo:

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

The inspect output will appear in the GitHub Actions **job summary** and be uploaded as an **artifact**.

### 4. Inspect and Commit Schema to Repo (Bootstrap)

Create `.github/workflows/db-bootstrap.yml` in your repo:

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

This extracts the current database schema as HCL and commits it to the specified path and branch.

## Inputs

| Name | Type | Required | Default | Used by | Description |
|------|------|----------|---------|---------|-------------|
| `schema-path` | string | yes | — | plan, apply | HCL file or directory path relative to repo root |
| `dev-url` | string | no | `docker://postgres/15` | all | Dev database URL for diff/normalization |
| `exclude` | string | no | `""` | all | Comma-separated glob patterns to exclude database objects (e.g. `*.audit_logs,temp_*`) |
| `atlas-version` | string | no | `v2.0.0` | all | Atlas binary version tag from NFUChen/atlas releases |
| `schemas` | string | no | `""` | inspect | Comma-separated schema names to inspect (empty = all) |
| `output-path` | string | no | `""` | inspect | If set, commits inspected HCL to this path in the repo |
| `target-branch` | string | no | `main` | inspect | Branch to commit to when output-path is set |

## Secrets

| Name | Required | Description |
|------|----------|-------------|
| `db-url` | yes | Target database connection URL |
| `wg-config-file` | no | WireGuard config file content for VPN tunnel to database |
| `pat` | no | Personal access token for pushing commits (required for inspect with `output-path`) |

## How It Works

```
PR opened/updated
  └─► schema-plan runs
        ├─ atlas schema diff (current DB vs desired HCL)
        ├─ writes result to job summary
        └─ uploads plan as artifact

Review plan ─► Merge PR ─► Manually trigger apply
                              └─► schema-apply runs
                                    ├─ atlas schema apply --dry-run (preview in summary)
                                    └─ atlas schema apply --auto-approve (execute)

Manually trigger inspect
  └─► schema-inspect runs
        ├─ atlas schema inspect (reads live DB)
        ├─ writes HCL to job summary
        ├─ uploads HCL as artifact
        └─ (optional) commits HCL to repo at output-path
```

## Customizing dev-url

The `dev-url` defaults to `docker://postgres/15`. Override it for other databases:

```yaml
# Single file
with:
  schema-path: "schema/schema.hcl"
  dev-url: "docker://mysql/8/mydb"

# Directory of HCL files
with:
  schema-path: "schema/"
  dev-url: "docker://mysql/8/mydb"
```

Common dev-url values:

| Database | dev-url |
|----------|---------|
| PostgreSQL 15 | `docker://postgres/15` |
| MySQL 8 | `docker://mysql/8/mydb` |
| MariaDB | `docker://mariadb/latest/mydb` |
| SQLite | `sqlite://file?mode=memory` |

## Excluding Database Objects

By default Atlas scans all database objects. Use the `exclude` input to skip objects you don't want Atlas to manage. Patterns are comma-separated and each becomes a separate `--exclude` flag.

```yaml
with:
  schema-path: "schema/schema.hcl"
  exclude: "*.audit_logs,temp_*"
```

### Pattern Examples

| Pattern | Effect |
|---------|--------|
| `temp_*` | Exclude all objects starting with `temp_` |
| `*.river_*` | Exclude objects matching `river_*` in any schema |
| `*[type=function]` | Exclude all functions |
| `*[type=policy\|trigger]` | Exclude all policies and triggers |
| `atlas_schema_revisions` | Exclude the `atlas_schema_revisions` schema itself |
| `atlas_schema_revisions.*` | Exclude objects inside `atlas_schema_revisions` schema |

See the [Atlas CLI Reference](https://atlasgo.io/cli-reference) for full glob pattern syntax.
