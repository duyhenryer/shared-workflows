# Shared GA - Reusable Workflows

> Centralized reusable workflows for consistent CI/CD across all repositories

## Quick Start

Use any workflow by referencing it with the `uses` keyword:

```yaml
name: My Workflow
on:
  pull_request:
    branches: [main, dev]

jobs:
  check:
    uses: duynhlab/gha-workflows/.github/workflows/go-check.yml@main
    with:
      command-test: 'go test ./...'
      lint: true
    secrets: inherit
```

## Available Workflows

| Workflow | Purpose | Features |
|----------|---------|----------|
| **[pr-checks.yml](.github/workflows/pr-checks.yml)** | PR handling | Branch validation, CODEOWNERS, Slack notifications |
| **[go-check.yml](.github/workflows/go-check.yml)** | Go code quality | Tests, linting, coverage artifacts |
| **[gitleaks.yml](.github/workflows/gitleaks.yml)** | Secret scanning (source) | Gitleaks CLI, PR diff / full scan, SARIF, job summary |
| **[go-security.yml](.github/workflows/go-security.yml)** | **Security entrypoint for Go services** | CodeQL + Govulncheck + Dependency Review behind one stable `Security Gate` check |
| **[codeql.yml](.github/workflows/codeql.yml)** | CodeQL analysis (one language per call) | `go` (autobuild/manual) or `actions` (none), `security-extended`, per-language SARIF category |
| **[govulncheck.yml](.github/workflows/govulncheck.yml)** | Go dependency vulnerabilities | Reachability analysis, pinned tool version, real blocking gate, SARIF |
| **[dependency-review.yml](.github/workflows/dependency-review.yml)** | PR dependency + license policy | Blocks new vulnerable dependencies, license audit split from the vuln gate |
| **[docker-build-go.yml](.github/workflows/docker-build-go.yml)** | Docker build (Go) | **Scan-before-push**, multi-platform, caching, provenance, outputs `tags` + `digest` + `scan-status` |
| **[docker-build-node.yml](.github/workflows/docker-build-node.yml)** | Docker build (Node) | **Scan-before-push**, multi-platform, caching, provenance, outputs `tags` + `digest` + `scan-status` |
| **[trivy-scan.yml](.github/workflows/trivy-scan.yml)** | Image vulnerability report | Trivy post-push scan, SARIF, Google Sheets reporting |
| **[docker-sign.yml](.github/workflows/docker-sign.yml)** | Cosign image signing | Keyless OIDC signing |
| **[goreleaser.yml](.github/workflows/goreleaser.yml)** | Go binary release (on `v*` tags) | GoReleaser → GitHub Release; tarball + `.sha256` + `build-info.env` matching the `packages` contract |
| **[sonarqube.yml](.github/workflows/sonarqube.yml)** | SonarCloud analysis | Go coverage, Quality Gate |
| **[tf-lint.yml](.github/workflows/tf-lint.yml)** | Terraform validation | Format check, TFLint analysis |
| **[status.yml](.github/workflows/status.yml)** | Build notifications | Slack, Google Sheets, job summaries |

> **Pipeline pattern:** Builder workflows (`docker-build-go.yml`, `docker-build-node.yml`) now include integrated Trivy scanning **before push**. Images are only pushed to GHCR if the scan passes. `trivy-scan.yml` remains available for optional post-push SARIF reporting.

---

> For detailed workflow documentation, inputs, outputs, and complete examples, see [USAGE.md](USAGE.md)

---

## Quick Examples

**Go Project:**
```yaml
jobs:
  go-check:
    uses: duynhlab/gha-workflows/.github/workflows/go-check.yml@main
    with:
      command-test: 'go test ./...'
      lint: true
    secrets: inherit
```

**Go Security Scanning (CodeQL + Govulncheck + Dependency Review):**
```yaml
jobs:
  go-security:
    uses: duynhlab/gha-workflows/.github/workflows/go-security.yml@main
    with:
      go-version-file: go.mod
      govulncheck-enforce: true
      dependency-review-severity: critical
    # No `secrets: inherit` — this workflow needs no secrets.
    permissions:
      actions: read
      contents: read
      pull-requests: read
      security-events: write
```

Require the `go-security / Security Gate` check in the branch ruleset.
See [docs/go-security.md](docs/go-security.md) for rollout, policy and exceptions, and [examples/](examples/) for full caller workflows.

**Secret Scanning (PR block, push warn):**
```yaml
jobs:
  gitleaks:
    uses: duynhlab/gha-workflows/.github/workflows/gitleaks.yml@main
    continue-on-error: ${{ github.event_name != 'pull_request' }}
    with:
      config-path: '.gitleaks.toml'  # optional
    permissions:
      contents: read
      security-events: write
```

**Docker Build with Scan-Before-Push (recommended):**
```yaml
jobs:
  build:
    uses: duynhlab/gha-workflows/.github/workflows/docker-build-go.yml@main
    with:
      image-name: my-service
      push: true
      scan-before-push: true        # default: true
      scan-severity: 'CRITICAL,HIGH'
      scan-exit-code: '1'
    secrets: inherit
    permissions:
      contents: read
      packages: write
      actions: read

  sign:
    needs: [build]
    uses: duynhlab/gha-workflows/.github/workflows/docker-sign.yml@main
    with:
      tags: ${{ needs.build.outputs.tags }}
      digest: ${{ needs.build.outputs.digest }}
    secrets: inherit
    permissions:
      contents: read
      packages: write
      id-token: write
```

**Docker Build with Post-Push Reporting (optional):**
```yaml
jobs:
  build:
    uses: duynhlab/gha-workflows/.github/workflows/docker-build-go.yml@main
    with:
      image-name: my-service
      push: true
    secrets: inherit
    permissions:
      contents: read
      packages: write
      actions: read

  # Optional: detailed SARIF + Google Sheets reporting
  trivy-report:
    needs: [build]
    if: needs.build.outputs.scan-status == 'pass'
    uses: duynhlab/gha-workflows/.github/workflows/trivy-scan.yml@main
    with:
      image-ref: ghcr.io/${{ github.repository }}/my-service@${{ needs.build.outputs.digest }}
      severity: 'CRITICAL,HIGH,MEDIUM'
      exit-code: '0'
      ignore-unfixed: true
    secrets: inherit
    permissions:
      contents: read
      packages: read
      security-events: write

  sign:
    needs: [build]
    uses: duynhlab/gha-workflows/.github/workflows/docker-sign.yml@main
    with:
      tags: ${{ needs.build.outputs.tags }}
      digest: ${{ needs.build.outputs.digest }}
    secrets: inherit
    permissions:
      contents: read
      packages: write
      id-token: write
```

**Terraform Project:**
```yaml
jobs:
  terraform-check:
    uses: duynhlab/gha-workflows/.github/workflows/tf-lint.yml@main
    with:
      tflint_minimum_failure_severity: 'error'
```

**Status Notification:**
```yaml
jobs:
  notify-status:
    uses: duynhlab/gha-workflows/.github/workflows/status.yml@main
    with:
      slack_channel_id: "#dev-notifications"
    secrets: inherit
```

---

## Required Secrets

| Secret | Used By | Required | Description |
|--------|---------|----------|-------------|
| `SLACK_BOT_TOKEN` | pr-checks.yml, status.yml | **Yes** | Slack bot token for notifications |
| `SONAR_TOKEN` | sonarqube.yml | **Yes** | SonarCloud authentication token |
| `GSHEET_CLIENT_EMAIL` | status.yml, trivy-scan.yml | No | Google service account email (Sheets reporting) |
| `GSHEET_PRIVATE_KEY` | status.yml, trivy-scan.yml | No | Google service account private key (Sheets reporting) |

Add in repository: Settings > Secrets and variables > Actions

---

## Documentation

- **[USAGE.md](USAGE.md)** - Complete workflow documentation with examples
- **[docs/go-security.md](docs/go-security.md)** - Go security contract: architecture, policy matrix, ruleset settings, rollout phases, exception process
- **[examples/](examples/)** - Copy-paste caller workflows for a Go service (`check` / `build` / scheduled security scan)
- **[.github/workflows/](.github/workflows/)** - Workflow source files

---
