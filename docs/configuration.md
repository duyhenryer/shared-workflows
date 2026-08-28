# Repository configuration

Calling repositories configure branch rules, SonarCloud, workflow permissions,
and the token used for automatic `main → dev` synchronization.

## Secrets

| Secret | Used by | Requirement |
|--------|---------|-------------|
| `SONAR_TOKEN` | `check.yml` | Required for human PRs; Dependabot runs skip Sonar |
| `SYNC_BRANCH_TOKEN` | `sync-branches` | GitHub App token or fine-grained PAT for unattended sync PR checks |
| `SLACK_BOT_TOKEN` | `status.yml` | Optional; only when status notifications are enabled |
| `GSHEET_CLIENT_EMAIL` / `GSHEET_PRIVATE_KEY` | `status.yml` | Optional reporting integration |

Image publication uses `GITHUB_TOKEN` plus OIDC. Do not store registry passwords
or Cosign private keys.

## Caller permissions

PR callers grant read-only source permissions plus Security tab upload:

```yaml
permissions:
  actions: read
  contents: read
  pull-requests: read
  security-events: write
```

Dev and release callers additionally grant only the trusted artifact jobs what
they need:

```yaml
permissions:
  actions: read
  attestations: write
  contents: write       # release; dev only needs read
  id-token: write
  packages: write
  pull-requests: read
  security-events: write
```

Reusable workflows cannot elevate permissions omitted by the caller.

## Branch rules

Protect both long-lived branches from direct pushes and force pushes.

### `dev`

- Require a pull request and at least one approval.
- Require `check / PR Gate` and Code Scanning merge protection.
- Accept approved branch prefixes and `sync/main-to-dev`.
- A successful push after merge invokes `build.yml`.

### `main`

- Require a pull request and at least one approval.
- Require the same PR and Code Scanning gates.
- Normal promotions must come from `dev`.
- Emergency changes use `hotfix/*` branched from `main`.
- Production release requires a reviewed `vX.Y.Z` tag.

## SonarCloud

The workflow always waits for the Quality Gate and fails a human PR when it
fails. Configure the project Quality Gate in SonarCloud to evaluate new code;
that policy is not encoded in workflow YAML.

## Security prerequisites

- Enable Dependency Graph.
- Enable GitHub Code Security/Advanced Security where the repository visibility
  requires it.
- Use CodeQL Advanced Setup and disable duplicate Default Setup scans.
- Configure Code Scanning merge protection for the alert severities the team
  blocks.
- Allow callers to use workflows and actions from `duynhlab/gha-workflows`.

See [go-security.md](go-security.md) for the complete scanner and exception
policy.

## Image automation prerequisite

Before adopting v2, Flux/Helm image automation must exclude `unscanned-*` and
OCI signature/attestation helper artifacts. Only `sha-*` may reach dev/staging;
only exact semantic versions may reach production.
