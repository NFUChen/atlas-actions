---
name: cloud-actions-atlas
description: Generate GitHub Actions caller workflows for Atlas database schema management (plan & apply). Use this skill whenever the user mentions database schema migrations, Atlas HCL, schema diff, schema apply, database CI/CD, or wants to set up automated database schema management in GitHub Actions. Also use when the user asks about cloud-actions Atlas workflows, DB schema pipelines, or declarative database schema management.
---

# Atlas Schema Workflows

Generate caller workflow YAML for the `NFUChen/cloud-actions` Atlas reusable workflows. These workflows manage database schema changes using [Atlas](https://atlasgo.io/) declarative schema management.

## Available Workflows

| Workflow | File | Purpose |
|----------|------|---------|
| Schema Plan | `atlas-schema-plan.yml` | Preview schema diff on PR -- shows planned SQL changes in job summary and uploads artifact |
| Schema Apply | `atlas-schema-apply.yml` | Apply schema changes -- dry-run preview then auto-approve apply |

Both workflows are invoked via `workflow_call` from the caller repo.

## When to Generate What

- **PR-triggered plan**: User wants to preview DB changes when schema files change in a pull request
- **Manual apply**: User wants to apply schema changes via `workflow_dispatch` after reviewing the plan
- **Both**: Most common -- set up plan on PR + manual apply as a pair

## Caller Workflow Reference

### Base URL

All workflows are referenced as:
```
NFUChen/cloud-actions/.github/workflows/<workflow-file>@main
```

### Schema Plan -- Inputs & Secrets

```yaml
jobs:
  plan:
    uses: NFUChen/cloud-actions/.github/workflows/atlas-schema-plan.yml@main
    with:
      schema-path: "schema/schema.hcl"    # required -- HCL file or directory
      dev-url: "docker://postgres/15"      # optional -- default: docker://postgres/15
      exclude: ""                          # optional -- comma-separated glob patterns
    secrets:
      db-url: ${{ secrets.DB_URL }}        # required -- target database connection URL
      wg-config-file: ${{ secrets.WG_CONFIG_FILE }}  # optional -- WireGuard VPN config
```

### Schema Apply -- Inputs & Secrets

```yaml
jobs:
  apply:
    uses: NFUChen/cloud-actions/.github/workflows/atlas-schema-apply.yml@main
    with:
      schema-path: "schema/schema.hcl"    # required -- HCL file or directory
      dev-url: "docker://postgres/15"      # optional -- default: docker://postgres/15
      exclude: ""                          # optional -- comma-separated glob patterns
    secrets:
      db-url: ${{ secrets.DB_URL }}        # required -- target database connection URL
      wg-config-file: ${{ secrets.WG_CONFIG_FILE }}  # optional -- WireGuard VPN config
```

### Inputs Detail

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `schema-path` | string | yes | -- | HCL file or directory path relative to repo root |
| `dev-url` | string | no | `docker://postgres/15` | Dev database URL for diff calculation |
| `exclude` | string | no | `""` | Comma-separated glob patterns to exclude database objects |

### Secrets Detail

| Name | Required | Description |
|------|----------|-------------|
| `db-url` | yes | Target database connection URL (e.g. `postgres://user:pass@host:5432/dbname`) |
| `wg-config-file` | no | WireGuard config file content for VPN tunnel to database |

### Common dev-url Values

| Database | dev-url |
|----------|---------|
| PostgreSQL 15 | `docker://postgres/15` |
| MySQL 8 | `docker://mysql/8/mydb` |
| MariaDB | `docker://mariadb/latest/mydb` |
| SQLite | `sqlite://file?mode=memory` |

### Exclude Pattern Examples

| Pattern | Effect |
|---------|--------|
| `temp_*` | Exclude objects starting with `temp_` |
| `*.river_*` | Exclude objects matching `river_*` in any schema |
| `*[type=function]` | Exclude all functions |
| `*[type=policy\|trigger]` | Exclude all policies and triggers |

## Complete Example: Plan on PR + Manual Apply

```yaml
# .github/workflows/db-plan.yml
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
    secrets:
      db-url: ${{ secrets.DB_URL }}
      wg-config-file: ${{ secrets.WG_CONFIG_FILE }}
```

```yaml
# .github/workflows/db-apply.yml
name: DB Schema Apply

on:
  workflow_dispatch:

jobs:
  apply:
    uses: NFUChen/cloud-actions/.github/workflows/atlas-schema-apply.yml@main
    with:
      schema-path: "schema/schema.hcl"
    secrets:
      db-url: ${{ secrets.DB_URL }}
      wg-config-file: ${{ secrets.WG_CONFIG_FILE }}
```

## Prerequisites for the Caller Repo

1. Atlas HCL schema file(s) in the repo (e.g. `schema/schema.hcl`)
2. `DB_URL` secret configured with the target database connection string
3. (Optional) `WG_CONFIG_FILE` secret if the database is behind a VPN

## Generation Guidelines

When generating caller workflows:

- Always ask the user for their schema file path if not provided
- Default to `docker://postgres/15` for dev-url unless user specifies a different database
- Include `wg-config-file` secret only if the user mentions VPN or private network access
- Use `paths` filter on PR trigger to only run plan when schema files change
- Use `workflow_dispatch` for apply so it requires manual triggering
- The `exclude` input is rarely needed -- only include if user mentions objects to skip
