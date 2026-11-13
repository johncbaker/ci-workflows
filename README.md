# ci-workflows

Reusable GitHub Actions workflows shared across all my projects.

## semgrep

Runs Semgrep (and optionally Gitleaks) on every PR and push to `main` in a single job, uploads SARIF as an artifact, and on `main` pushes (and manual dispatches) ingests findings as Stakker tasks via `stakker-cli` (set `stakker-ingest: false` to skip that, and pass no Stakker secrets).

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
| `STAKKER_CLI_PAT` | Fine-grained PAT with Contents: Read on `johncbaker/stakker-cli` (only with `stakker-ingest`) |
| `STAKKER_API_TOKEN` | Stakker API token with `tasks:write` |
| `STAKKER_API_URL` | Base URL of the Stakker API |

### Inputs

| Input | Default | Description |
|---|---|---|
| `min-severity` | `warning` | Minimum SARIF level to create a task for (`error`\|`warning`\|`note`\|`none`) |
| `stakker-ingest` | `true` | Send `main` findings to Stakker; `false` needs no Stakker secrets |
| `stakker-cli-ref` | `main` | Git ref of `stakker-cli` to install |
| `semgrep-version` | `1.183.0` | Pinned Semgrep release, installed into a cached virtualenv |
| `gitleaks` | `false` | Also run Gitleaks over the new commits (PR or push range) in the same job |
| `runs-on` | Blacksmith 2 vCPU | Runner label(s) as a JSON string |

One job covers both scanners, so repos that also used a separate `gitleaks.yml` should pass `gitleaks: true` here and delete that file. On pull requests Semgrep is diff-aware (only new findings); pushes to `main` run a full scan for ingest.

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
    permissions:
      contents: write
```

Caller **must** grant `contents: write` — the reusable workflow force-pushes to the target branch and cannot escalate beyond the caller's permissions.

### Inputs

| Input | Default | Description |
|---|---|---|
| `target-branch` | `production` | Branch to force-push the tagged commit to. |
| `runs-on` | `'"ubuntu-latest"'` | Runner label(s) as a JSON-encoded string. |

## Stakker task transitions

The former `stakker-transition`, `stakker-conflict-check` and `stakker-ci-failure` reusable workflows are gone. Stakker now receives these events from its GitHub App webhook (`POST /api/v1/webhooks/github` in stakker-api), so they cost no runner minutes. See `docs/github-app-webhooks.md` in stakker-api.
