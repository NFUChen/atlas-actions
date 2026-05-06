# Atlas Actions

Reusable GitHub Actions workflows for [Atlas](https://atlasgo.io/) declarative schema management.

## Workflows

| Workflow | Purpose |
|----------|---------|
| `schema-plan.yml` | Generate and preview schema migration plan |
| `schema-apply.yml` | Apply schema changes to the target database |

## Usage

### Prerequisites

- An Atlas HCL schema file in your repo (e.g. `schema/schema.hcl`)
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
    uses: <org>/atlas-actions/.github/workflows/schema-plan.yml@main
    with:
      schema-file: "schema/schema.hcl"
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
    uses: <org>/atlas-actions/.github/workflows/schema-apply.yml@main
    with:
      schema-file: "schema/schema.hcl"
    secrets:
      db-url: ${{ secrets.DB_URL }}
      wg-config-file: ${{ secrets.WG_CONFIG_FILE }}
```

Go to **Actions > DB Schema Apply > Run workflow** to trigger manually.

## Inputs

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `schema-file` | string | yes | — | HCL file path relative to repo root |
| `dev-url` | string | no | `docker://postgres/15` | Dev database URL for diff calculation |

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
with:
  schema-file: "schema/schema.hcl"
  dev-url: "docker://mysql/8/mydb"
```

Common dev-url values:

| Database | dev-url |
|----------|---------|
| PostgreSQL 15 | `docker://postgres/15` |
| MySQL 8 | `docker://mysql/8/mydb` |
| MariaDB | `docker://mariadb/latest/mydb` |
| SQLite | `sqlite://file?mode=memory` |
