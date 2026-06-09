# Hardening Existing Workflows with zizmor

Use this reference when the user has an existing repo with GitHub Actions workflows and wants to apply security best practices. The tool `zizmor` (https://docs.zizmor.sh/) handles most fixes automatically.

## Prerequisites

- `zizmor` CLI installed (https://docs.zizmor.sh/installation/)
- `gh` CLI authenticated (for SHA pinning only)

## Quick Reference: The Complete Workflow

```bash
# Step 1: Create zizmor config (choose pinning policy)
cat > zizmor.yml << 'EOF'
rules:
  unpinned-uses:
    config:
      policies:
        '*': ref-pin    # accept version tags; use 'hash-pin' for SHA requirement
EOF

# Step 2: Run zizmor autofix (without SHA pinning)
zizmor --fix=all .github/workflows/

# Step 2 (alternative): With SHA pinning
zizmor --fix=all --gh-token=$(gh auth token) .github/workflows/

# Step 3: Verify
zizmor .github/workflows/
```

## What zizmor Fixes Automatically

| Fix | Mode | Description |
|-----|------|-------------|
| `persist-credentials: false` | `--fix=all` (unsafe) | Adds to all `actions/checkout` steps |
| Template injection | `--fix=all` (unsafe) | Moves dangerous `${{ }}` expressions to env vars with descriptive names |
| SHA pinning | `--fix=all --gh-token=...` | Resolves tags to commit SHAs (handles annotated tags correctly) |
| Mismatched version comments | `--fix=all --gh-token=...` | Corrects `# vX.Y.Z` comments that don't match the SHA |

**Safe vs unsafe:** `--fix=safe` applies only fixes that can't break anything. Currently most useful fixes (persist-credentials, template injection) are classified as "unsafe" because they can theoretically break workflows that depend on persisted credentials or exact expression formatting.

## What Requires Manual Addition

zizmor detects but cannot autofix these:

### 1. `permissions: {}` (Least Privilege)

Add before the `jobs:` block in workflows that don't have a top-level `permissions:`:

```yaml
permissions: {}

jobs:
  ...
```

### 2. zizmor CI Workflow

Create `.github/workflows/zizmor.yml`:

```yaml
name: GitHub Actions Security Analysis with zizmor

on:
  push:
    paths:
      - '.github/workflows/*'
  pull_request:
    paths:
      - '.github/workflows/*'

permissions: {}

jobs:
  zizmor:
    name: Run zizmor
    runs-on: ubuntu-latest
    permissions:
      contents: read
      actions: read
    steps:
      - name: Checkout repository
        uses: actions/checkout@v6
        with:
          persist-credentials: false

      - name: Run zizmor
        uses: zizmorcore/zizmor-action@5f14fd08f7cf1cb1609c1e344975f152c7ee938d # v0.5.6
```

### 3. Dependabot for GitHub Actions

Create or update `.github/dependabot.yml`:

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    cooldown:
      default-days: 7
```

## zizmor Config Options

### `zizmor.yml` — Pinning Policies

```yaml
rules:
  unpinned-uses:
    config:
      policies:
        '*': ref-pin          # Allow version tags (v6, v4.2.0)
        # '*': hash-pin       # Require full SHA pins
        # '*': any            # No pinning requirement
```

Use `ref-pin` when you rely on Dependabot to keep tags current. Use `hash-pin` for maximum supply-chain security (Galaxy, high-security repos).

### Suppressing Findings

For cases where a fix would break the workflow (e.g., template injection in a docker image tag expression):

```yaml
- run: |  # zizmor: ignore[template-injection]
    echo "FROM galaxy/galaxy-min:${{ steps.branch.outputs.name }}" > /tmp/Dockerfile
```

## Fix Mode Reference

| Command | What it does |
|---------|-------------|
| `zizmor .github/workflows/` | Audit only (no changes) |
| `zizmor --fix=safe ...` | Apply only safe fixes (very conservative) |
| `zizmor --fix=all ...` | Apply safe + unsafe fixes |
| `zizmor --fix=unsafe-only ...` | Apply only unsafe fixes |
| `zizmor --fix=all --gh-token=$(gh auth token) ...` | All fixes including SHA pinning |

## Template Injection: What zizmor Considers Dangerous

zizmor moves these to env vars automatically:
- `github.event.inputs.*` — workflow_dispatch inputs (user-supplied)
- `github.event.pull_request.title` / `.body` — attacker-controllable
- `github.event.issue.title` / `.body`
- `github.event.comment.body`
- `github.head_ref` — PR branch name

It leaves these alone (safe contexts):
- `matrix.*` — defined in the workflow, not user-controlled
- `runner.*` — system-provided
- `github.repository`, `github.ref` — metadata, not user input
- `steps.*.outputs.*` — generally safe (informational level only)

## Common Patterns from Real PRs

Based on galaxyproject/galaxy#22827 and similar hardening efforts:

1. Run `zizmor --fix=all` — handles persist-credentials + template injection
2. Add `permissions: {}` to each workflow (sed one-liner: can search for `^jobs:` and insert above)
3. Add zizmor CI workflow + config
4. Add dependabot
5. For cases zizmor can't fix cleanly, add `# zizmor: ignore[rule-name]` comments
6. Verify: `zizmor .github/workflows/` should report "No findings" or only suppressed ones
