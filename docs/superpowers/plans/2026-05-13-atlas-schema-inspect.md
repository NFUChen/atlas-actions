# Atlas Schema Inspect + Custom Binary Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an `atlas-schema-inspect.yml` reusable workflow and migrate all Atlas workflows from `ariga/setup-atlas@v0` to a custom `NFUChen/atlas` binary with a configurable version input.

**Architecture:** All three Atlas workflows (plan, apply, inspect) share a common setup pattern: checkout, optional WireGuard, connectivity check, and custom Atlas binary download. The inspect workflow adds schema inspection with HCL output to summary + artifact, plus an optional commit-back flow using a PAT and target branch.

**Tech Stack:** GitHub Actions `workflow_call`, shell scripts, `atlas schema inspect` CLI, `actions/checkout@v4`, `actions/upload-artifact@v4`

---

### Task 1: Migrate atlas-schema-plan.yml to custom Atlas binary

**Files:**
- Modify: `.github/workflows/atlas-schema-plan.yml`

- [ ] **Step 1: Add `atlas-version` input to the workflow**

Add the new input after `exclude` in the `on.workflow_call.inputs` section of `.github/workflows/atlas-schema-plan.yml`:

```yaml
      atlas-version:
        description: "Atlas binary version tag from NFUChen/atlas releases"
        required: false
        type: string
        default: "v0.1.2"
```

- [ ] **Step 2: Replace `ariga/setup-atlas@v0` step with custom binary download**

Replace this step in `.github/workflows/atlas-schema-plan.yml`:

```yaml
      - name: Setup Atlas
        uses: ariga/setup-atlas@v0
```

With:

```yaml
      - name: Download Atlas binary
        env:
          ATLAS_VERSION: ${{ inputs.atlas-version }}
        run: |
          curl -fsSL "https://github.com/NFUChen/atlas/releases/download/${ATLAS_VERSION}/atlas-linux-amd64" -o /usr/local/bin/atlas
          chmod +x /usr/local/bin/atlas
          atlas version
```

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/atlas-schema-plan.yml
git commit -m "feat: migrate atlas-schema-plan to custom NFUChen/atlas binary"
```

---

### Task 2: Migrate atlas-schema-apply.yml to custom Atlas binary

**Files:**
- Modify: `.github/workflows/atlas-schema-apply.yml`

- [ ] **Step 1: Add `atlas-version` input to the workflow**

Add the new input after `exclude` in the `on.workflow_call.inputs` section of `.github/workflows/atlas-schema-apply.yml`:

```yaml
      atlas-version:
        description: "Atlas binary version tag from NFUChen/atlas releases"
        required: false
        type: string
        default: "v0.1.2"
```

- [ ] **Step 2: Replace `ariga/setup-atlas@v0` step with custom binary download**

Replace this step in `.github/workflows/atlas-schema-apply.yml`:

```yaml
      - name: Setup Atlas
        uses: ariga/setup-atlas@v0
```

With:

```yaml
      - name: Download Atlas binary
        env:
          ATLAS_VERSION: ${{ inputs.atlas-version }}
        run: |
          curl -fsSL "https://github.com/NFUChen/atlas/releases/download/${ATLAS_VERSION}/atlas-linux-amd64" -o /usr/local/bin/atlas
          chmod +x /usr/local/bin/atlas
          atlas version
```

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/atlas-schema-apply.yml
git commit -m "feat: migrate atlas-schema-apply to custom NFUChen/atlas binary"
```

---

### Task 3: Create atlas-schema-inspect.yml workflow

**Files:**
- Create: `.github/workflows/atlas-schema-inspect.yml`

- [ ] **Step 1: Create the full inspect workflow file**

Create `.github/workflows/atlas-schema-inspect.yml` with the following content:

```yaml
name: Atlas Schema Inspect

on:
  workflow_call:
    inputs:
      atlas-version:
        description: "Atlas binary version tag from NFUChen/atlas releases"
        required: false
        type: string
        default: "v0.1.2"
      dev-url:
        description: "Atlas dev database URL for schema normalization"
        required: false
        type: string
        default: "docker://postgres/15"
      exclude:
        description: "Comma-separated glob patterns to exclude database objects from inspect (e.g. '*.audit_logs,temp_*'). Supports type filters like '*[type=function]'. Each pattern is passed as a separate --exclude flag."
        required: false
        type: string
        default: ""
      schemas:
        description: "Comma-separated schema names to inspect (e.g. 'public,auth'). Empty means all schemas."
        required: false
        type: string
        default: ""
      output-path:
        description: "If set, commits the inspected HCL to this path in the repo (e.g. 'schema/schema.hcl')"
        required: false
        type: string
        default: ""
      target-branch:
        description: "Branch to commit to when output-path is set"
        required: false
        type: string
        default: "main"
    secrets:
      db-url:
        description: "Target database connection URL"
        required: true
      wg-config-file:
        description: "WireGuard configuration file content for VPN tunnel"
        required: false
      pat:
        description: "Personal access token for pushing commits (required when output-path is set)"
        required: false

jobs:
  inspect:
    name: Schema Inspect
    runs-on: ubuntu-latest
    steps:
      - name: Checkout caller repo
        uses: actions/checkout@v4

      - name: Set up WireGuard connection
        if: ${{ env.WG_CONFIG_FILE != '' }}
        uses: ./.github/actions/wireguard
        env:
          WG_CONFIG_FILE: ${{ secrets.wg-config-file }}
        with:
          WG_CONFIG_FILE: ${{ secrets.wg-config-file }}

      - name: Check database connectivity
        env:
          DB_URL: ${{ secrets.db-url }}
        run: |
          DB_HOST=$(echo "$DB_URL" | sed -E 's|.*@([^:/]+).*|\1|')
          DB_PORT=$(echo "$DB_URL" | sed -E 's|.*:([0-9]+)/.*|\1|')
          if ! echo "$DB_PORT" | grep -qE '^[0-9]+$'; then
            DB_PORT=5432
          fi
          echo "Testing connectivity to ${DB_HOST}:${DB_PORT}..."
          if ! nc -zv -w 10 "$DB_HOST" "$DB_PORT" 2>&1; then
            echo "::error::Cannot reach database at ${DB_HOST}:${DB_PORT}. Aborting inspect."
            exit 1
          fi
          echo "Database connectivity check passed."

      - name: Download Atlas binary
        env:
          ATLAS_VERSION: ${{ inputs.atlas-version }}
        run: |
          curl -fsSL "https://github.com/NFUChen/atlas/releases/download/${ATLAS_VERSION}/atlas-linux-amd64" -o /usr/local/bin/atlas
          chmod +x /usr/local/bin/atlas
          atlas version

      - name: Inspect database schema
        id: inspect
        env:
          DB_URL: ${{ secrets.db-url }}
          DEV_URL: ${{ inputs.dev-url }}
          EXCLUDE: ${{ inputs.exclude }}
          SCHEMAS: ${{ inputs.schemas }}
        run: |
          EXCLUDE_ARGS=()
          if [ -n "$EXCLUDE" ]; then
            IFS=',' read -ra PATTERNS <<< "$EXCLUDE"
            for pattern in "${PATTERNS[@]}"; do
              trimmed=$(echo "$pattern" | xargs)
              [ -n "$trimmed" ] && EXCLUDE_ARGS+=(--exclude "$trimmed")
            done
          fi
          SCHEMA_ARGS=()
          if [ -n "$SCHEMAS" ]; then
            IFS=',' read -ra SCHEMA_LIST <<< "$SCHEMAS"
            for schema in "${SCHEMA_LIST[@]}"; do
              trimmed=$(echo "$schema" | xargs)
              [ -n "$trimmed" ] && SCHEMA_ARGS+=(--schema "$trimmed")
            done
          fi
          OUTPUT=$(atlas schema inspect \
            --url "$DB_URL" \
            --dev-url "$DEV_URL" \
            "${EXCLUDE_ARGS[@]}" \
            "${SCHEMA_ARGS[@]}" 2>&1) || true
          {
            echo "schema<<EOF"
            echo "$OUTPUT"
            echo "EOF"
          } >> "$GITHUB_OUTPUT"

      - name: Write job summary
        env:
          DEV_URL: ${{ inputs.dev-url }}
          SCHEMAS: ${{ inputs.schemas }}
          SCHEMA_OUTPUT: ${{ steps.inspect.outputs.schema }}
        run: |
          {
            echo "## Atlas Schema Inspect"
            echo ""
            echo "**Dev URL**: \`${DEV_URL}\`"
            if [ -n "$SCHEMAS" ]; then
              echo "**Schemas**: \`${SCHEMAS}\`"
            fi
            echo ""
            echo "### Current Schema"
            echo ""
            echo '```hcl'
            echo "$SCHEMA_OUTPUT"
            echo '```'
          } >> "$GITHUB_STEP_SUMMARY"

      - name: Save schema to file
        env:
          SCHEMA_OUTPUT: ${{ steps.inspect.outputs.schema }}
        run: |
          mkdir -p inspect-output
          echo "$SCHEMA_OUTPUT" > inspect-output/schema.hcl

      - name: Upload schema artifact
        uses: actions/upload-artifact@v4
        with:
          name: atlas-schema-inspect
          path: inspect-output/schema.hcl
          retention-days: 30

      - name: Commit schema to repo
        if: ${{ inputs.output-path != '' }}
        env:
          OUTPUT_PATH: ${{ inputs.output-path }}
          TARGET_BRANCH: ${{ inputs.target-branch }}
          PAT: ${{ secrets.pat }}
          SCHEMA_OUTPUT: ${{ steps.inspect.outputs.schema }}
        run: |
          if [ -z "$PAT" ]; then
            echo "::error::output-path is set but pat secret is missing. A personal access token is required to push commits."
            exit 1
          fi
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git remote set-url origin "https://x-access-token:${PAT}@github.com/${{ github.repository }}.git"
          git fetch origin "$TARGET_BRANCH"
          git checkout "$TARGET_BRANCH"
          mkdir -p "$(dirname "$OUTPUT_PATH")"
          echo "$SCHEMA_OUTPUT" > "$OUTPUT_PATH"
          if git diff --quiet "$OUTPUT_PATH" 2>/dev/null; then
            echo "No schema changes detected. Skipping commit."
          else
            git add "$OUTPUT_PATH"
            git commit -m "chore: update schema from atlas inspect"
            git push origin "$TARGET_BRANCH"
          fi
```

- [ ] **Step 2: Commit**

```bash
git add .github/workflows/atlas-schema-inspect.yml
git commit -m "feat: add atlas-schema-inspect workflow with commit-back support"
```

---

### Task 4: Update docs/ATLAS.md

**Files:**
- Modify: `docs/ATLAS.md`

- [ ] **Step 1: Add inspect to workflow table**

In `docs/ATLAS.md`, replace the workflow table:

```markdown
| Workflow | Purpose |
|----------|---------|
| `schema-plan.yml` | Generate and preview schema migration plan |
| `schema-apply.yml` | Apply schema changes to the target database |
```

With:

```markdown
| Workflow | Purpose |
|----------|---------|
| `schema-plan.yml` | Generate and preview schema migration plan |
| `schema-apply.yml` | Apply schema changes to the target database |
| `schema-inspect.yml` | Inspect live database schema, optionally commit HCL to repo |
```

- [ ] **Step 2: Add inspect usage sections**

In `docs/ATLAS.md`, after the "2. Apply via Manual Trigger" section (after the "Go to **Actions > DB Schema Apply > Run workflow** to trigger manually." line), add:

```markdown

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
```

- [ ] **Step 3: Add `atlas-version` and inspect-specific inputs to the inputs table**

In `docs/ATLAS.md`, replace the inputs table:

```markdown
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `schema-path` | string | yes | — | HCL file or directory path relative to repo root |
| `dev-url` | string | no | `docker://postgres/15` | Dev database URL for diff calculation |
| `exclude` | string | no | `""` | Comma-separated glob patterns to exclude database objects (e.g. `*.audit_logs,temp_*`) |
```

With:

```markdown
| Name | Type | Required | Default | Used by | Description |
|------|------|----------|---------|---------|-------------|
| `schema-path` | string | yes | — | plan, apply | HCL file or directory path relative to repo root |
| `dev-url` | string | no | `docker://postgres/15` | all | Dev database URL for diff/normalization |
| `exclude` | string | no | `""` | all | Comma-separated glob patterns to exclude database objects (e.g. `*.audit_logs,temp_*`) |
| `atlas-version` | string | no | `v0.1.2` | all | Atlas binary version tag from NFUChen/atlas releases |
| `schemas` | string | no | `""` | inspect | Comma-separated schema names to inspect (empty = all) |
| `output-path` | string | no | `""` | inspect | If set, commits inspected HCL to this path in the repo |
| `target-branch` | string | no | `main` | inspect | Branch to commit to when output-path is set |
```

- [ ] **Step 4: Add `pat` to the secrets table**

In `docs/ATLAS.md`, replace the secrets table:

```markdown
| Name | Required | Description |
|------|----------|-------------|
| `db-url` | yes | Target database connection URL |
| `wg-config-file` | no | WireGuard config file content for VPN tunnel to database |
```

With:

```markdown
| Name | Required | Description |
|------|----------|-------------|
| `db-url` | yes | Target database connection URL |
| `wg-config-file` | no | WireGuard config file content for VPN tunnel to database |
| `pat` | no | Personal access token for pushing commits (required for inspect with `output-path`) |
```

- [ ] **Step 5: Update "How It Works" diagram**

In `docs/ATLAS.md`, replace the How It Works diagram:

````markdown
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
````

With:

````markdown
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
````

- [ ] **Step 6: Commit**

```bash
git add docs/ATLAS.md
git commit -m "docs: add inspect workflow and atlas-version input to Atlas documentation"
```

---

### Task 5: Update README.md

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Add inspect workflow to the README workflows table**

In `README.md`, after the line:

```markdown
| **Database** | `atlas-schema-apply.yml` | Apply Atlas schema changes |
```

Add:

```markdown
| **Database** | `atlas-schema-inspect.yml` | Inspect live DB schema, optionally commit HCL to repo |
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add atlas-schema-inspect to README workflow table"
```

---

### Task 6: Update cloud-actions-atlas skill

**Files:**
- Modify: `.claude/skills/cloud-actions-atlas/SKILL.md`

- [ ] **Step 1: Update skill description in frontmatter**

In `.claude/skills/cloud-actions-atlas/SKILL.md`, replace the frontmatter:

```yaml
---
name: cloud-actions-atlas
description: Generate GitHub Actions caller workflows for Atlas database schema management (plan & apply). Use this skill whenever the user mentions database schema migrations, Atlas HCL, schema diff, schema apply, database CI/CD, or wants to set up automated database schema management in GitHub Actions. Also use when the user asks about cloud-actions Atlas workflows, DB schema pipelines, or declarative database schema management.
---
```

With:

```yaml
---
name: cloud-actions-atlas
description: Generate GitHub Actions caller workflows for Atlas database schema management (plan, apply & inspect). Use this skill whenever the user mentions database schema migrations, Atlas HCL, schema diff, schema apply, schema inspect, database CI/CD, or wants to set up automated database schema management in GitHub Actions. Also use when the user asks about cloud-actions Atlas workflows, DB schema pipelines, declarative database schema management, or database schema bootstrapping.
---
```

- [ ] **Step 2: Add inspect to the available workflows table**

In `.claude/skills/cloud-actions-atlas/SKILL.md`, replace the available workflows table:

```markdown
| Workflow | File | Purpose |
|----------|------|---------|
| Schema Plan | `atlas-schema-plan.yml` | Preview schema diff on PR -- shows planned SQL changes in job summary and uploads artifact |
| Schema Apply | `atlas-schema-apply.yml` | Apply schema changes -- dry-run preview then auto-approve apply |
```

With:

```markdown
| Workflow | File | Purpose |
|----------|------|---------|
| Schema Plan | `atlas-schema-plan.yml` | Preview schema diff on PR -- shows planned SQL changes in job summary and uploads artifact |
| Schema Apply | `atlas-schema-apply.yml` | Apply schema changes -- dry-run preview then auto-approve apply |
| Schema Inspect | `atlas-schema-inspect.yml` | Inspect live database schema -- outputs HCL to job summary and artifact, optionally commits to repo |
```

- [ ] **Step 3: Update "When to Generate What" section**

In `.claude/skills/cloud-actions-atlas/SKILL.md`, replace the "When to Generate What" section:

```markdown
## When to Generate What

- **PR-triggered plan**: User wants to preview DB changes when schema files change in a pull request
- **Manual apply**: User wants to apply schema changes via `workflow_dispatch` after reviewing the plan
- **Both**: Most common -- set up plan on PR + manual apply as a pair
```

With:

```markdown
## When to Generate What

- **PR-triggered plan**: User wants to preview DB changes when schema files change in a pull request
- **Manual apply**: User wants to apply schema changes via `workflow_dispatch` after reviewing the plan
- **Plan + apply pair**: Most common -- set up plan on PR + manual apply as a pair
- **Inspect (audit)**: User wants to view the current live database schema on demand
- **Inspect (bootstrap)**: User wants to extract the current DB schema and commit it as HCL to the repo
```

- [ ] **Step 4: Add `atlas-version` input to existing plan/apply caller workflow references**

In `.claude/skills/cloud-actions-atlas/SKILL.md`, replace the Schema Plan caller workflow reference:

```yaml
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

With:

```yaml
### Schema Plan -- Inputs & Secrets

```yaml
jobs:
  plan:
    uses: NFUChen/cloud-actions/.github/workflows/atlas-schema-plan.yml@main
    with:
      schema-path: "schema/schema.hcl"    # required -- HCL file or directory
      dev-url: "docker://postgres/15"      # optional -- default: docker://postgres/15
      exclude: ""                          # optional -- comma-separated glob patterns
      atlas-version: "v0.1.2"             # optional -- default: v0.1.2
    secrets:
      db-url: ${{ secrets.DB_URL }}        # required -- target database connection URL
      wg-config-file: ${{ secrets.WG_CONFIG_FILE }}  # optional -- WireGuard VPN config
```

Do the same for the Schema Apply caller workflow reference — add the `atlas-version` line after `exclude`.

- [ ] **Step 5: Add Schema Inspect caller workflow reference section**

In `.claude/skills/cloud-actions-atlas/SKILL.md`, after the Schema Apply caller workflow reference section, add:

```markdown
### Schema Inspect -- Inputs & Secrets

```yaml
jobs:
  inspect:
    uses: NFUChen/cloud-actions/.github/workflows/atlas-schema-inspect.yml@main
    with:
      dev-url: "docker://postgres/15"      # optional -- default: docker://postgres/15
      exclude: ""                          # optional -- comma-separated glob patterns
      schemas: "public"                    # optional -- comma-separated schema names
      atlas-version: "v0.1.2"             # optional -- default: v0.1.2
      output-path: ""                      # optional -- commit HCL to this path
      target-branch: "main"               # optional -- branch to commit to
    secrets:
      db-url: ${{ secrets.DB_URL }}        # required -- target database connection URL
      wg-config-file: ${{ secrets.WG_CONFIG_FILE }}  # optional -- WireGuard VPN config
      pat: ${{ secrets.PAT }}              # optional -- required when output-path is set
```
```

- [ ] **Step 6: Add inspect-specific inputs and secrets to the detail tables**

In `.claude/skills/cloud-actions-atlas/SKILL.md`, replace the Inputs Detail table:

```markdown
### Inputs Detail

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `schema-path` | string | yes | -- | HCL file or directory path relative to repo root |
| `dev-url` | string | no | `docker://postgres/15` | Dev database URL for diff calculation |
| `exclude` | string | no | `""` | Comma-separated glob patterns to exclude database objects |
```

With:

```markdown
### Inputs Detail

| Name | Type | Required | Default | Used by | Description |
|------|------|----------|---------|---------|-------------|
| `schema-path` | string | yes | -- | plan, apply | HCL file or directory path relative to repo root |
| `dev-url` | string | no | `docker://postgres/15` | all | Dev database URL for diff/normalization |
| `exclude` | string | no | `""` | all | Comma-separated glob patterns to exclude database objects |
| `atlas-version` | string | no | `v0.1.2` | all | Atlas binary version tag from NFUChen/atlas releases |
| `schemas` | string | no | `""` | inspect | Comma-separated schema names to inspect (empty = all) |
| `output-path` | string | no | `""` | inspect | If set, commits inspected HCL to this path in the repo |
| `target-branch` | string | no | `main` | inspect | Branch to commit to when output-path is set |
```

Replace the Secrets Detail table:

```markdown
### Secrets Detail

| Name | Required | Description |
|------|----------|-------------|
| `db-url` | yes | Target database connection URL (e.g. `postgres://user:pass@host:5432/dbname`) |
| `wg-config-file` | no | WireGuard config file content for VPN tunnel to database |
```

With:

```markdown
### Secrets Detail

| Name | Required | Description |
|------|----------|-------------|
| `db-url` | yes | Target database connection URL (e.g. `postgres://user:pass@host:5432/dbname`) |
| `wg-config-file` | no | WireGuard config file content for VPN tunnel to database |
| `pat` | no | Personal access token for pushing commits (required for inspect with `output-path`) |
```

- [ ] **Step 7: Add inspect examples to the complete example section**

In `.claude/skills/cloud-actions-atlas/SKILL.md`, after the existing "Complete Example: Plan on PR + Manual Apply" section, add:

```markdown
## Complete Example: Inspect (Audit)

```yaml
# .github/workflows/db-inspect.yml
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

## Complete Example: Inspect (Bootstrap)

```yaml
# .github/workflows/db-bootstrap.yml
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
```

- [ ] **Step 8: Update generation guidelines**

In `.claude/skills/cloud-actions-atlas/SKILL.md`, replace the Generation Guidelines section:

```markdown
## Generation Guidelines

When generating caller workflows:

- Always ask the user for their schema file path if not provided
- Default to `docker://postgres/15` for dev-url unless user specifies a different database
- Include `wg-config-file` secret only if the user mentions VPN or private network access
- Use `paths` filter on PR trigger to only run plan when schema files change
- Use `workflow_dispatch` for apply so it requires manual triggering
- The `exclude` input is rarely needed -- only include if user mentions objects to skip
```

With:

```markdown
## Generation Guidelines

When generating caller workflows:

- Always ask the user for their schema file path if not provided (for plan/apply)
- Default to `docker://postgres/15` for dev-url unless user specifies a different database
- Include `wg-config-file` secret only if the user mentions VPN or private network access
- Use `paths` filter on PR trigger to only run plan when schema files change
- Use `workflow_dispatch` for apply and inspect so they require manual triggering
- The `exclude` input is rarely needed -- only include if user mentions objects to skip
- For inspect: ask the user which schemas to inspect if not specified
- For inspect with commit-back: require `output-path`, `target-branch`, and `pat` secret
- The `atlas-version` input is rarely needed -- only include if user needs a specific version
```

- [ ] **Step 9: Update prerequisites section**

In `.claude/skills/cloud-actions-atlas/SKILL.md`, replace the prerequisites section:

```markdown
## Prerequisites for the Caller Repo

1. Atlas HCL schema file(s) in the repo (e.g. `schema/schema.hcl`)
2. `DB_URL` secret configured with the target database connection string
3. (Optional) `WG_CONFIG_FILE` secret if the database is behind a VPN
```

With:

```markdown
## Prerequisites for the Caller Repo

1. Atlas HCL schema file(s) in the repo (e.g. `schema/schema.hcl`) — required for plan/apply, not needed for inspect
2. `DB_URL` secret configured with the target database connection string
3. (Optional) `WG_CONFIG_FILE` secret if the database is behind a VPN
4. (Optional) `PAT` secret with repo write permissions — required for inspect with `output-path`
```

- [ ] **Step 10: Commit**

```bash
git add .claude/skills/cloud-actions-atlas/SKILL.md
git commit -m "docs: add inspect workflow and atlas-version input to cloud-actions-atlas skill"
```
