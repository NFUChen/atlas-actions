# Atlas Actions

Reusable GitHub Actions workflows for [Atlas](https://atlasgo.io/) declarative schema management.

## Workflows

| Workflow | Purpose |
|----------|---------|
| `schema-plan.yml` | Generate and preview schema migration plan |
| `schema-apply.yml` | Apply schema changes to the target database |

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

## Inputs

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `schema-path` | string | yes | — | HCL file or directory path relative to repo root |
| `dev-url` | string | no | `docker://postgres/15` | Dev database URL for diff calculation |
| `exclude` | string | no | `""` | Comma-separated glob patterns to exclude database objects (e.g. `*.audit_logs,temp_*`) |

## Secrets

| Name | Required | Description |
|------|----------|-------------|
| `db-url` | yes | Target database connection URL |
| `wg-config-file` | no | WireGuard config file content for VPN tunnel to database |

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
