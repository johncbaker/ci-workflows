# ci-workflows

Reusable GitHub Actions workflows shared across all my projects.

## semgrep

Runs Semgrep on every PR and push to `main`, uploads SARIF as an artifact, and on `main` pushes (and manual dispatches) ingests findings as Stakker tasks via `stakker-cli`.

### Caller example

```yaml
name: Semgrep

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
  workflow_dispatch:

concurrency:
  group: semgrep-${{ github.ref }}
  cancel-in-progress: true

jobs:
  call:
    uses: johncbaker/ci-workflows/.github/workflows/semgrep.yml@main
    secrets:
      STAKKER_CLI_PAT: ${{ secrets.STAKKER_CLI_PAT }}
      STAKKER_API_TOKEN: ${{ secrets.STAKKER_API_TOKEN }}
      STAKKER_API_URL: ${{ secrets.STAKKER_API_URL }}
```

### Required secrets (set at org or repo level)

| Secret | Purpose |
|---|---|
| `STAKKER_CLI_PAT` | Fine-grained PAT with Contents: Read on `omnixi/stakker-cli` |
| `STAKKER_API_TOKEN` | Stakker API token with `tasks:write` |
| `STAKKER_API_URL` | Base URL of the Stakker API |

### Inputs

| Input | Default | Description |
|---|---|---|
| `min-severity` | `warning` | Minimum SARIF level to create a task for (`error`\|`warning`\|`note`\|`none`) |
| `stakker-cli-ref` | `main` | Git ref of `stakker-cli` to install |

## promote

Force-pushes a tagged commit onto a long-lived deploy branch (default `production`). Use this to trigger a deploy pipeline that watches the `production` branch. The caller owns the tag-pattern trigger and the concurrency group.

### Caller example

```yaml
name: Deploy

on:
  push:
    tags: ['v*']

concurrency:
  group: promote
  cancel-in-progress: false

jobs:
  call:
    uses: johncbaker/ci-workflows/.github/workflows/promote.yml@v1
```

### Inputs

| Input | Default | Description |
|---|---|---|
| `target-branch` | `production` | Branch to force-push the tagged commit to. |
| `runs-on` | `'"ubuntu-latest"'` | Runner label(s) as a JSON-encoded string. |
