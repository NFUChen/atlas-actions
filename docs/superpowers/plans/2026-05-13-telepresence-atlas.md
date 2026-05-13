# Telepresence Support for Atlas Workflows Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add Telepresence as a network connectivity option alongside WireGuard in all 3 Atlas workflows, with mutual exclusion validation.

**Architecture:** A new `.github/actions/telepresence/action.yml` composite action handles install + connect (mirroring the existing wireguard action pattern). Each Atlas workflow gets a `kubeconfig` secret, `also-proxy` input, a mutual exclusion validation step, and a conditional Telepresence step.

**Tech Stack:** GitHub Actions composite actions, Telepresence CLI, shell scripts

---

### Task 1: Create Telepresence composite action

**Files:**
- Create: `.github/actions/telepresence/action.yml`

- [ ] **Step 1: Create the composite action file**

Create `.github/actions/telepresence/action.yml` with the following content:

```yaml
name: Telepresence Connection
description: Connect to a Kubernetes cluster via Telepresence for accessing cluster-internal resources

inputs:
  kubeconfig:
    description: Kubeconfig file content
    required: true
  also-proxy:
    description: "Comma-separated CIDR ranges to also proxy through the Telepresence tunnel (e.g. '10.0.0.0/8,172.16.0.0/12')"
    required: false
    default: ""

runs:
  using: composite
  steps:
    - run: |
        curl -fsSL https://app.getambassador.io/download/tel2oss/releases/download/v2.22.0/telepresence-linux-amd64 -o /usr/local/bin/telepresence
        chmod +x /usr/local/bin/telepresence
        telepresence version
      shell: bash
    - run: |
        mkdir -p ~/.kube
        echo "${{ inputs.kubeconfig }}" > ~/.kube/config
        chmod 600 ~/.kube/config
      shell: bash
    - run: |
        ALSO_PROXY_ARGS=()
        if [ -n "$ALSO_PROXY" ]; then
          IFS=',' read -ra CIDRS <<< "$ALSO_PROXY"
          for cidr in "${CIDRS[@]}"; do
            trimmed=$(echo "$cidr" | xargs)
            [ -n "$trimmed" ] && ALSO_PROXY_ARGS+=(--also-proxy "$trimmed")
          done
        fi
        telepresence connect "${ALSO_PROXY_ARGS[@]}"
        telepresence status
      shell: bash
      env:
        ALSO_PROXY: ${{ inputs.also-proxy }}

branding:
  icon: globe
  color: blue
```

- [ ] **Step 2: Commit**

```bash
git add .github/actions/telepresence/action.yml
git commit -m "feat: add Telepresence composite action for K8s cluster connectivity"
```

---

### Task 2: Add Telepresence support to atlas-schema-plan.yml

**Files:**
- Modify: `.github/workflows/atlas-schema-plan.yml`

- [ ] **Step 1: Add `also-proxy` input and `kubeconfig` secret**

In `.github/workflows/atlas-schema-plan.yml`, add the `also-proxy` input after `atlas-version`:

```yaml
      also-proxy:
        description: "Comma-separated CIDR ranges to also proxy through Telepresence (e.g. '10.0.0.0/8')"
        required: false
        type: string
        default: ""
```

Add the `kubeconfig` secret after `wg-config-file`:

```yaml
      kubeconfig:
        description: "Kubeconfig file content for Telepresence connection to K8s cluster"
        required: false
```

- [ ] **Step 2: Add validation and Telepresence steps**

In `.github/workflows/atlas-schema-plan.yml`, after the "Checkout caller repo" step and before the "Set up WireGuard connection" step, add the validation step:

```yaml
      - name: Validate network config
        env:
          WG: ${{ secrets.wg-config-file }}
          KC: ${{ secrets.kubeconfig }}
        run: |
          if [ -n "$WG" ] && [ -n "$KC" ]; then
            echo "::error::Both wg-config-file and kubeconfig are set. Only one network tunnel is allowed."
            exit 1
          fi
```

After the "Set up WireGuard connection" step and before the "Check database connectivity" step, add the Telepresence step:

```yaml
      - name: Set up Telepresence connection
        if: ${{ env.KUBECONFIG_CONTENT != '' }}
        uses: ./.github/actions/telepresence
        env:
          KUBECONFIG_CONTENT: ${{ secrets.kubeconfig }}
        with:
          kubeconfig: ${{ secrets.kubeconfig }}
          also-proxy: ${{ inputs.also-proxy }}
```

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/atlas-schema-plan.yml
git commit -m "feat: add Telepresence support to atlas-schema-plan workflow"
```

---

### Task 3: Add Telepresence support to atlas-schema-apply.yml

**Files:**
- Modify: `.github/workflows/atlas-schema-apply.yml`

- [ ] **Step 1: Add `also-proxy` input and `kubeconfig` secret**

In `.github/workflows/atlas-schema-apply.yml`, add the `also-proxy` input after `atlas-version`:

```yaml
      also-proxy:
        description: "Comma-separated CIDR ranges to also proxy through Telepresence (e.g. '10.0.0.0/8')"
        required: false
        type: string
        default: ""
```

Add the `kubeconfig` secret after `wg-config-file`:

```yaml
      kubeconfig:
        description: "Kubeconfig file content for Telepresence connection to K8s cluster"
        required: false
```

- [ ] **Step 2: Add validation and Telepresence steps**

In `.github/workflows/atlas-schema-apply.yml`, after the "Checkout caller repo" step and before the "Set up WireGuard connection" step, add:

```yaml
      - name: Validate network config
        env:
          WG: ${{ secrets.wg-config-file }}
          KC: ${{ secrets.kubeconfig }}
        run: |
          if [ -n "$WG" ] && [ -n "$KC" ]; then
            echo "::error::Both wg-config-file and kubeconfig are set. Only one network tunnel is allowed."
            exit 1
          fi
```

After the "Set up WireGuard connection" step and before the "Check database connectivity" step, add:

```yaml
      - name: Set up Telepresence connection
        if: ${{ env.KUBECONFIG_CONTENT != '' }}
        uses: ./.github/actions/telepresence
        env:
          KUBECONFIG_CONTENT: ${{ secrets.kubeconfig }}
        with:
          kubeconfig: ${{ secrets.kubeconfig }}
          also-proxy: ${{ inputs.also-proxy }}
```

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/atlas-schema-apply.yml
git commit -m "feat: add Telepresence support to atlas-schema-apply workflow"
```

---

### Task 4: Add Telepresence support to atlas-schema-inspect.yml

**Files:**
- Modify: `.github/workflows/atlas-schema-inspect.yml`

- [ ] **Step 1: Add `also-proxy` input and `kubeconfig` secret**

In `.github/workflows/atlas-schema-inspect.yml`, add the `also-proxy` input after `target-branch`:

```yaml
      also-proxy:
        description: "Comma-separated CIDR ranges to also proxy through Telepresence (e.g. '10.0.0.0/8')"
        required: false
        type: string
        default: ""
```

Add the `kubeconfig` secret after `wg-config-file` (before `pat`):

```yaml
      kubeconfig:
        description: "Kubeconfig file content for Telepresence connection to K8s cluster"
        required: false
```

- [ ] **Step 2: Add validation and Telepresence steps**

In `.github/workflows/atlas-schema-inspect.yml`, after the "Checkout caller repo" step and before the "Set up WireGuard connection" step, add:

```yaml
      - name: Validate network config
        env:
          WG: ${{ secrets.wg-config-file }}
          KC: ${{ secrets.kubeconfig }}
        run: |
          if [ -n "$WG" ] && [ -n "$KC" ]; then
            echo "::error::Both wg-config-file and kubeconfig are set. Only one network tunnel is allowed."
            exit 1
          fi
```

After the "Set up WireGuard connection" step and before the "Check database connectivity" step, add:

```yaml
      - name: Set up Telepresence connection
        if: ${{ env.KUBECONFIG_CONTENT != '' }}
        uses: ./.github/actions/telepresence
        env:
          KUBECONFIG_CONTENT: ${{ secrets.kubeconfig }}
        with:
          kubeconfig: ${{ secrets.kubeconfig }}
          also-proxy: ${{ inputs.also-proxy }}
```

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/atlas-schema-inspect.yml
git commit -m "feat: add Telepresence support to atlas-schema-inspect workflow"
```

---

### Task 5: Update docs/ATLAS.md

**Files:**
- Modify: `docs/ATLAS.md`

- [ ] **Step 1: Add `also-proxy` to inputs table**

In `docs/ATLAS.md`, add a new row after `target-branch` in the inputs table:

```markdown
| `also-proxy` | string | no | `""` | all | Comma-separated CIDR ranges to also proxy through Telepresence |
```

- [ ] **Step 2: Add `kubeconfig` to secrets table**

In `docs/ATLAS.md`, add a new row after `wg-config-file` in the secrets table:

```markdown
| `kubeconfig` | no | Kubeconfig file content for Telepresence connection to K8s cluster (mutually exclusive with `wg-config-file`) |
```

- [ ] **Step 3: Update prerequisites**

In `docs/ATLAS.md`, replace the prerequisites section:

```markdown
### Prerequisites

- An Atlas HCL schema file or directory in your repo (e.g. `schema/schema.hcl` or `schema/`)
- A `DB_URL` secret configured in your repo with the target database connection string
- (Optional) A `WG_CONFIG_FILE` secret with WireGuard config if the database is behind a VPN
```

With:

```markdown
### Prerequisites

- An Atlas HCL schema file or directory in your repo (e.g. `schema/schema.hcl` or `schema/`)
- A `DB_URL` secret configured in your repo with the target database connection string
- (Optional) A `WG_CONFIG_FILE` secret with WireGuard config if the database is behind a VPN
- (Optional) A `KUBECONFIG` secret if the database is inside a Kubernetes cluster (use Telepresence; mutually exclusive with WireGuard)
```

- [ ] **Step 4: Add "Connecting via Telepresence" section**

In `docs/ATLAS.md`, after the "Excluding Database Objects" section (before the end of the file), add:

```markdown

## Connecting via Telepresence

If your database is only accessible inside a Kubernetes cluster, use [Telepresence](https://www.telepresence.io/) instead of WireGuard. Provide a `KUBECONFIG` secret and optionally specify CIDR ranges to also proxy.

> **Note:** WireGuard and Telepresence are mutually exclusive. If both `wg-config-file` and `kubeconfig` secrets are provided, the workflow will fail.

```yaml
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
```

- [ ] **Step 5: Commit**

```bash
git add docs/ATLAS.md
git commit -m "docs: add Telepresence connectivity option to Atlas documentation"
```

---

### Task 6: Update cloud-actions-atlas skill

**Files:**
- Modify: `.claude/skills/cloud-actions-atlas/SKILL.md`

- [ ] **Step 1: Add `kubeconfig` to all caller workflow reference sections**

In `.claude/skills/cloud-actions-atlas/SKILL.md`, add the `kubeconfig` secret line to the Schema Plan, Schema Apply, and Schema Inspect caller workflow references. For each section, add after the `wg-config-file` line:

```yaml
      kubeconfig: ${{ secrets.KUBECONFIG }}   # optional -- Telepresence K8s tunnel (mutually exclusive with wg-config-file)
```

Also add the `also-proxy` input to each `with:` block:

```yaml
      also-proxy: ""                          # optional -- CIDR ranges for Telepresence
```

- [ ] **Step 2: Add `also-proxy` to inputs detail table and `kubeconfig` to secrets detail table**

In `.claude/skills/cloud-actions-atlas/SKILL.md`, add a new row to the Inputs Detail table after `target-branch`:

```markdown
| `also-proxy` | string | no | `""` | all | Comma-separated CIDR ranges to also proxy through Telepresence |
```

Add a new row to the Secrets Detail table after `wg-config-file`:

```markdown
| `kubeconfig` | no | Kubeconfig file content for Telepresence connection to K8s cluster (mutually exclusive with `wg-config-file`) |
```

- [ ] **Step 3: Add Telepresence caller workflow example**

In `.claude/skills/cloud-actions-atlas/SKILL.md`, after the "Complete Example: Inspect (Bootstrap)" section, add:

```markdown
## Complete Example: Plan via Telepresence

```yaml
# .github/workflows/db-plan-k8s.yml
name: DB Schema Plan (K8s)

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
```

- [ ] **Step 4: Update generation guidelines**

In `.claude/skills/cloud-actions-atlas/SKILL.md`, replace the Generation Guidelines section:

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

With:

```markdown
## Generation Guidelines

When generating caller workflows:

- Always ask the user for their schema file path if not provided (for plan/apply)
- Default to `docker://postgres/15` for dev-url unless user specifies a different database
- Include `wg-config-file` secret only if the user mentions VPN or private network access
- Include `kubeconfig` secret if the user mentions Kubernetes, cluster-internal, or Telepresence
- WireGuard and Telepresence are mutually exclusive -- never include both in the same caller workflow
- If using Telepresence, ask if the user needs `also-proxy` for additional CIDR ranges
- Use `paths` filter on PR trigger to only run plan when schema files change
- Use `workflow_dispatch` for apply and inspect so they require manual triggering
- The `exclude` input is rarely needed -- only include if user mentions objects to skip
- For inspect: ask the user which schemas to inspect if not specified
- For inspect with commit-back: require `output-path`, `target-branch`, and `pat` secret
- The `atlas-version` input is rarely needed -- only include if user needs a specific version
```

- [ ] **Step 5: Update prerequisites**

In `.claude/skills/cloud-actions-atlas/SKILL.md`, replace the prerequisites section:

```markdown
## Prerequisites for the Caller Repo

1. Atlas HCL schema file(s) in the repo (e.g. `schema/schema.hcl`) — required for plan/apply, not needed for inspect
2. `DB_URL` secret configured with the target database connection string
3. (Optional) `WG_CONFIG_FILE` secret if the database is behind a VPN
4. (Optional) `PAT` secret with repo write permissions — required for inspect with `output-path`
```

With:

```markdown
## Prerequisites for the Caller Repo

1. Atlas HCL schema file(s) in the repo (e.g. `schema/schema.hcl`) — required for plan/apply, not needed for inspect
2. `DB_URL` secret configured with the target database connection string
3. (Optional) `WG_CONFIG_FILE` secret if the database is behind a VPN
4. (Optional) `KUBECONFIG` secret if the database is inside a K8s cluster (Telepresence; mutually exclusive with WireGuard)
5. (Optional) `PAT` secret with repo write permissions — required for inspect with `output-path`
```

- [ ] **Step 6: Commit**

```bash
git add .claude/skills/cloud-actions-atlas/SKILL.md
git commit -m "docs: add Telepresence support to cloud-actions-atlas skill"
```
