# Workflow reference

The public API is `check.yml`, `build.yml`, and `release.yml`. Callers pin the
workflow to an immutable commit SHA and grant only the permissions shown here.

## `check.yml`

Complete pre-merge gate for Go repositories. It only accepts pull requests into
`dev` or `main` and never publishes an artifact.

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `image-name` | Yes | — | Image path below the caller repository's GHCR path |
| `sonar-project-key` | Yes | — | SonarCloud project key |
| `sonar-organization` | Yes | — | SonarCloud organization |
| `command-test` | No | `go test -race ...` | Must produce `coverage.out` |
| `go-version-file` | No | `go.mod` | Toolchain source |
| `cache-dependency-path` | No | `go.sum` | `setup-go` cache dependency path |
| `dockerfile` / `context` | No | `Dockerfile` / `.` | Container build inputs |
| `build-args` | No | empty | Non-secret Docker build arguments |
| `scan-actions` | No | `false` | Also scan Actions code with CodeQL |
| `runs-on` | No | `ubuntu-latest` | Runner label |

Secret: `SONAR_TOKEN`. It is optional only because Dependabot cannot read normal
Actions secrets; human pull requests fail if Sonar cannot authenticate.

```yaml
name: Check
on:
  pull_request:
    branches: [dev, main]

jobs:
  check:
    uses: duynhlab/gha-workflows/.github/workflows/check.yml@<commit-sha>
    with:
      image-name: order-service
      sonar-project-key: duynhlab_order-service
      sonar-organization: duynhlab
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
    permissions:
      actions: read
      contents: read
      pull-requests: read
      security-events: write
```

With the supplied caller template, the stable required check is
`check / PR Gate`. CodeQL alert severity remains
a GitHub Code Scanning ruleset control rather than an exit-code interpretation.

## `build.yml`

Post-merge pipeline for `dev`. It reruns lint and tests, pushes a quarantine
image, scans its digest, and only then creates `sha-<short-sha>`.

| Input | Required | Default |
|-------|----------|---------|
| `image-name` | Yes | — |
| `command-test` | No | `go test -race ...` |
| `go-version-file` | No | `go.mod` |
| `cache-dependency-path` | No | `go.sum` |
| `dockerfile` / `context` | No | `Dockerfile` / `.` |
| `platforms` | No | `linux/amd64,linux/arm64` |
| `build-args` | No | empty |
| `runs-on` | No | `ubuntu-latest` |

Outputs: `image-ref` and `digest`.

```yaml
name: Build dev image
on:
  push:
    branches: [dev]

jobs:
  build:
    uses: duynhlab/gha-workflows/.github/workflows/build.yml@<commit-sha>
    with:
      image-name: order-service
    permissions:
      actions: read
      attestations: write
      contents: read
      id-token: write
      packages: write
```

The workflow uses the `cache-dev` registry cache. The cache is an optimization,
not a trusted artifact, and the resulting image digest is always scanned.

## `release.yml`

Production pipeline for an exact `vX.Y.Z` tag. The guard rejects tags whose
commit is not reachable from `main`. It reruns source gates and performs a
cache-free container build.

| Input | Required | Default |
|-------|----------|---------|
| `image-name` | Yes | — |
| `command-test` | No | `go test -race ...` |
| `go-version-file` | No | `go.mod` |
| `go-version` | No | `1.26` |
| `dockerfile` / `context` | No | `Dockerfile` / `.` |
| `platforms` | No | `linux/amd64,linux/arm64` |
| `build-args` | No | empty |
| `publish-binaries` | No | `true` |
| `scan-actions` | No | `false` |
| `runs-on` | No | `ubuntu-latest` |

Outputs: `image-ref` and `digest`.

```yaml
name: Release
on:
  push:
    tags: ['v*']

jobs:
  release:
    uses: duynhlab/gha-workflows/.github/workflows/release.yml@<commit-sha>
    with:
      image-name: order-service
    permissions:
      actions: read
      attestations: write
      contents: write
      id-token: write
      packages: write
      pull-requests: read
      security-events: write
```

The versioned image, signature, provenance, and SBOM all use the same subject
digest. GoReleaser runs only after the image gate passes and its checksum file
is used to attest the release assets.

## Container primitives

### `container-verify.yml`

- Pull-request only.
- Builds `linux/amd64` with `load: true` and `push: false`.
- Disables provenance and SBOM.
- Blocks fixable CRITICAL findings by default.
- Receives no package, OIDC, or attestation write permission from `check.yml`.

### `container-publish.yml`

- `channel: dev` requires `refs/heads/dev`.
- `channel: release` requires `vX.Y.Z`.
- Builds once to `unscanned-*`, scans `image@digest`, then promotes with crane.
- Verifies the promoted digest before Cosign signing and GitHub attestations.
- Release mode disables the registry build cache.

## Security primitives

`go-security.yml` aggregates CodeQL, govulncheck, and dependency review. It is
used by `check.yml` and `release.yml`, and it remains callable directly for a
scheduled full scan. See [go-security.md](go-security.md).

`gitleaks.yml`, `sonarqube.yml`, `go-check.yml`, and `goreleaser.yml` are nested
implementation workflows. Their direct inputs remain documented in the YAML
schema; new service callers should use the public entrypoints.

## Other workflows

- `tf-lint.yml`: Terraform/OpenTofu format and TFLint checks.
- `status.yml`: optional Slack and Google Sheets status reporting.
- `actionlint.yml`: self-CI for this repository, not reusable.

## Removed v1 contracts

Version 2 removes `pr-checks.yml`, the separate Go/Node Docker builders,
`trivy-scan.yml`, and `docker-sign.yml`. Existing consumers pinned to a v1
commit continue to resolve that historical workflow file. Upgrading to v2 is an
explicit caller change; there is no mixed v1/v2 compatibility mode.
