# Faultline GitHub Action

Structural risk analysis for Go codebases - on every PR and push.

Faultline scans your Go repository, posts a risk advisory comment on pull requests,
uploads SARIF results to GitHub code scanning, and optionally sends a governance
snapshot to Faultline Enterprise for portfolio-level tracking.

## Quick start

Add `.github/workflows/faultline.yml` to your repository:

```yaml
name: Faultline

on:
  pull_request:
  push:
    branches: [main]

jobs:
  faultline:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
      security-events: write
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0      # Required: Faultline needs git history

      - uses: faultline-go/action@v1
```

That's it. On pull requests, Faultline posts a risk advisory comment and uploads
SARIF to GitHub code scanning. On pushes to main, it scans, uploads SARIF, and
exposes a compact scan summary output for downstream workflow steps.

## With test coverage

```yaml
      - uses: actions/setup-go@v6
        with:
          go-version: stable

      - name: Run tests
        run: go test ./... -coverprofile=coverage.out

      - uses: faultline-go/action@v1
        with:
          coverage: coverage.out
```

Coverage data significantly improves risk score accuracy by replacing the neutral
`FL-COV-002` default with real per-package coverage gaps.

## Fail on high-severity findings

```yaml
      - uses: faultline-go/action@v1
        with:
          fail-on: high
```

The workflow step exits non-zero if any new high-severity findings appear in the PR.
Existing findings are not flagged - only regressions introduced by the PR.

## With Faultline Enterprise

Store your API token as a repository secret named `FAULTLINE_API_TOKEN`. Existing
pilot installs that use a repository or organization Actions variable with the
same name can use the fallback expression below:

```yaml
      - uses: faultline-go/action@v1
        with:
          enterprise-url: https://api.gofaultline.dev
          enterprise-token: ${{ secrets.FAULTLINE_API_TOKEN || vars.FAULTLINE_API_TOKEN }}
          enterprise-org-id: YOUR_ORG_ID
```

When Enterprise credentials are set, Faultline automatically uploads a governance
snapshot from pull request reviews and push scans. The Enterprise dashboard
tracks portfolio-level risk trends, owner scorecards, policy compliance, and
audit evidence across all repos.

Get your org ID at [app.gofaultline.dev](https://app.gofaultline.dev) -> Settings.
Create an API token at Settings -> API Tokens.

## 15-minute Enterprise activation path

1. Start a trial at [app.gofaultline.dev](https://app.gofaultline.dev).
2. Create an API token and save it as `FAULTLINE_API_TOKEN`, preferably as a GitHub secret.
3. Add the Action with `enterprise-url`, `enterprise-token`, and `enterprise-org-id`.
4. Merge or run the workflow on one production Go repository.
5. Open the Enterprise dashboard and review the first non-demo snapshot.

That first snapshot is the activation milestone. From there, teams can enable
weekly digests, invite owners, review suppression debt, export audit evidence,
and expand coverage to the repositories that carry production risk.

## With architecture boundaries

```yaml
      - uses: faultline-go/action@v1
        with:
          config: .faultline.yaml
```

With a `faultline.yaml` at the repo root, Faultline enforces architecture boundary
rules and custom suppression policies. The PR comment includes policy violation
details alongside ownership and risk findings.

```yaml
# .faultline.yaml
version: 1
boundaries:
  - name: handlers-must-not-import-storage
    from: "*/internal/handlers/*"
    deny:
      - "*/internal/storage/*"
```

## Inputs

| Input | Default | Description |
|---|---|---|
| `version` | `latest` | Faultline CLI version |
| `go-version` | `stable` | Go version for installation |
| `patterns` | `./...` | Go package patterns to scan |
| `coverage` | | Path to coverage profile |
| `config` | | Path to faultline.yaml |
| `fail-on` | `none` | Fail on: `none`, `high`, `critical` |
| `post-comment` | `auto` | Post PR comment (`true`, `false`, `auto`) |
| `sarif-upload` | `true` | Upload SARIF to code scanning |
| `compare-mode` | `auto` | PR diff mode: `auto`, `worktree`, `history` |
| `enterprise-url` | | Enterprise API URL |
| `enterprise-token` | | Enterprise API token (use a secret) |
| `enterprise-org-id` | | Enterprise organization ID |

## Outputs

| Output | Description |
|---|---|
| `risk-summary` | Compact `.summary` JSON from the full `faultline-report.json` scan report on non-PR runs |
| `snapshot-id` | Enterprise snapshot ID (when uploaded) |

## Permissions

The Action requires these GitHub token permissions:

```yaml
permissions:
  contents: read
  pull-requests: write
  security-events: write
```

## How it works

On pull requests, Faultline runs `faultline pr review` which diffs the changed
packages against the base branch, generates a risk advisory comment, and produces
SARIF for inline code annotations. Only packages changed in the PR are highlighted.

On pushes to main, Faultline runs `faultline scan ./...` which produces a full
snapshot of the repository's risk posture. If Enterprise credentials are set,
pull request review snapshots and push snapshots are uploaded to track trend data
over time.

Faultline is local-first and source-free. No source code leaves your runner.
The scanner reads git history, go.mod, and CODEOWNERS. It does not upload source
files, ASTs, or compiled artifacts.

## OSS

The Faultline scanner is open source under Apache 2.0:
[github.com/faultline-go/faultline](https://github.com/faultline-go/faultline)

The GitHub Action is open source under Apache 2.0:
[github.com/faultline-go/action](https://github.com/faultline-go/action)

Faultline Enterprise adds multi-repo dashboards, policy packs, suppression
governance, and signed audit exports:
[gofaultline.dev](https://gofaultline.dev)
