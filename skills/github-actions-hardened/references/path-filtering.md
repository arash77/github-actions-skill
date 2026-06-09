# Path Filtering

Reduce wasted CI minutes by only triggering workflows when relevant files change.

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

**Rules:**
- If `paths` and `paths-ignore` both match the same file, `paths-ignore` wins.
- For documentation-only repos, omit path filtering entirely — it would prevent the workflow from ever running.
- Path filtering doesn't apply to `workflow_dispatch` or `schedule` triggers.

**Monorepo pattern** — trigger different jobs based on which package changed:

```yaml
on:
  push:
    branches: [main]

jobs:
  build-api:
    if: contains(github.event.commits[0].modified, 'packages/api/')
    ...
  build-web:
    if: contains(github.event.commits[0].modified, 'packages/web/')
    ...
```

For a more robust monorepo approach, use `dorny/paths-filter@v3` which handles merge commits and multiple-commit pushes correctly.
