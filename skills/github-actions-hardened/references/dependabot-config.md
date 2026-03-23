# Dependabot Configuration Variants

## Minimal (GitHub Actions only)

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    groups:
      github-actions:
        patterns:
          - "*"
    commit-message:
      prefix: "chore(ci)"
```

## With Reviewers and Labels

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    reviewers:
      - "platform-team"
    labels:
      - "dependencies"
      - "ci"
    groups:
      github-actions:
        patterns:
          - "*"
    commit-message:
      prefix: "chore(ci)"
      include: "scope"
```

## Multi-Ecosystem (Actions + npm + Docker)

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    groups:
      github-actions:
        patterns: ["*"]
    commit-message:
      prefix: "chore(ci)"

  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 5
    groups:
      npm-dependencies:
        patterns: ["*"]
    commit-message:
      prefix: "chore(deps)"

  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
    commit-message:
      prefix: "chore(docker)"
```

## With Ignore Patterns

Use `ignore` to suppress updates for pinned internal actions or actions you manage manually:

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    ignore:
      # Ignore patch updates for stable actions (still get minor/major)
      - dependency-name: "actions/checkout"
        update-types: ["version-update:semver-patch"]
      # Skip entirely for a specific action
      - dependency-name: "my-org/internal-action"
```

## Notes
