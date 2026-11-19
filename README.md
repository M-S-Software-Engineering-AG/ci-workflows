# ci-workflows

Shared GitHub Actions workflows and Renovate presets for M&S repos.

## Layout

GitHub only finds reusable workflows directly in `.github/workflows/` (no subfolders), so the repo type is a file-name prefix:

| Prefix | For | Files |
|---|---|---|
| `common-` | internal building blocks, **never referenced by a repo** | renovate, auto-tag, daily-scan |
| `go-` | Go code: tests, binaries | renovate, auto-tag, pr, release, daily-scan |
| `image-` | Docker image via `docker-bake.hcl`, any language | pr, release |
| `coredns-plugin-` | CoreDNS plugins | renovate, auto-tag, release, daily-scan |

A `<type>-*.yml` either holds its own logic or calls a `common-*.yml` with the type's settings.
Types are building blocks; a repo combines the ones it needs and never references `common-*`:

| Repo | pr | release | renovate, auto-tag, daily-scan | renovate.json |
|---|---|---|---|---|
| container-supervisor, env2yml | `go-pr` | `go-release` | `go-*` | `renovate/go` |
| docker-socket-proxy-go, corefile-gen | `go-pr` + `image-pr` | `image-release` | `go-*` | `renovate/go` |
| coredns-acmednschallenge, coredns-traefik | own `ci.yml` | `coredns-plugin-release` | `coredns-plugin-*` | `renovate/coredns-plugin` |

coredns-plugin repos keep their own `ci.yml` (integration tests are plugin-specific) and their `.github/ci/*.sh` scripts (`sbom.sh` is used by release and daily scan).

## Using it in a repo

No releases: repos reference `@main`, and this repo only gets dependency updates from Renovate.
Triggers, permissions and the `ci-ok` job stay in the repo: GitHub doesn't let a reusable workflow define its triggers, and `ci-ok` is the required check in the ruleset, so its name must not change.
Example for a Go CLI repo:

```yaml
# .github/workflows/ci.yml
name: CI
on:
  pull_request:
    branches: [main, 'release/**']
jobs:
  go:
    uses: M-S-Software-Engineering-AG/ci-workflows/.github/workflows/go-pr.yml@main
  # image repos add:
  # image:
  #   uses: M-S-Software-Engineering-AG/ci-workflows/.github/workflows/image-pr.yml@main
  # The only required check in the ruleset. Add new jobs to `needs`.
  ci-ok:
    if: always()
    needs: [go]
    runs-on: ubuntu-latest
    steps:
      - if: contains(needs.*.result, 'failure') || contains(needs.*.result, 'cancelled')
        run: exit 1
```

```yaml
# .github/workflows/release.yml
name: Release
on:
  push:
    tags: ['v*']
permissions:
  contents: write        # image-release also needs packages: write
jobs:
  release:
    uses: M-S-Software-Engineering-AG/ci-workflows/.github/workflows/go-release.yml@main
    with:
      binary-name: env2yml
      package: ./cmd/env2yml
```

```yaml
# .github/workflows/auto-tag.yml
name: Auto tag
on:
  push:
    branches: ['release/**']
permissions: {}
jobs:
  tag:
    uses: M-S-Software-Engineering-AG/ci-workflows/.github/workflows/go-auto-tag.yml@main
    secrets: inherit
```

```yaml
# .github/workflows/renovate.yml
name: Renovate
on:
  schedule:
    - cron: '0 */2 * * *'
      timezone: 'Europe/Zurich'
  push:
    branches: [main]
  workflow_dispatch:
  workflow_run:
    workflows: [CI]
    types: [completed]
permissions: {}
jobs:
  renovate:
    uses: M-S-Software-Engineering-AG/ci-workflows/.github/workflows/go-renovate.yml@main
    secrets: inherit
```

```yaml
# .github/workflows/daily-scan.yml
name: Daily scan
on:
  schedule:
    - cron: '0 8 * * *'
      timezone: Europe/Zurich
  workflow_dispatch:
jobs:
  scan:
    uses: M-S-Software-Engineering-AG/ci-workflows/.github/workflows/go-daily-scan.yml@main
```

```json
// renovate.json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>M-S-Software-Engineering-AG/ci-workflows//renovate/go"]
}
```

### Release inputs

| Workflow | Inputs |
|---|---|
| `go-release` | `binary-name` (required), `package` (default `.`) |
| `image-release` | none; `docker-bake.hcl` reads `VERSION`, `GIT_SHA`, `LATEST` (only add `:latest` when `LATEST == "true"`) |
| `coredns-plugin-release` | `plugin-name` (required, names `<plugin-name>.cdx.json`) |

## Requirements

- The automation GitHub App must be installed on ci-workflows: Renovate reads the presets from here.
- Org variable `AUTOMATION_APP_CLIENT_ID` and secret `AUTOMATION_APP_KEY` must be available to the calling repo.
- If ci-workflows is private, only repos in the M-S-Software-Engineering-AG org can call it.

