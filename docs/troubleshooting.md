# Troubleshooting

## PR branch policy fails

- PRs into `dev` need an approved prefix such as `feature/`, `fix/`, `hotfix/`,
  `docs/`, `ci/`, `dependabot/`, `renovate/`, or `sync/`.
- PRs into `main` must come from `dev` or `hotfix/*`.

## SonarCloud fails or cannot find coverage

`check.yml` waits for `go-check.yml`, downloads the `coverage-report` artifact,
and expects `coverage.out`. A custom `command-test` must write that file.

Do not set `fail-on-quality-gate: false` in a service caller. Fix the new-code
gate or the SonarCloud project configuration. Dependabot is the only documented
secret-related skip.

## Trivy blocks publication

The default gate blocks fixable CRITICAL findings and ignores vulnerabilities
without an available fix. Fix or update the affected package/base image.

An exception uses a narrowly scoped `.trivyignore` with an owner and expiry.
There is no public switch in `build.yml` or `release.yml` that disables the
scan.

When a publish scan fails, the `unscanned-*` tag remains for investigation but
the `sha-*` or version tag must not exist.

## Promoted digest differs

`container-publish.yml` fails after promotion if `crane digest` does not match
the Buildx digest. Do not retry signing manually. Inspect the registry tag and
rerun the entire workflow after the registry issue is resolved.

## Signing or attestation fails

Confirm the caller grants:

```yaml
permissions:
  attestations: write
  id-token: write
  packages: write
```

Also confirm the repository is eligible for GitHub artifact attestations under
its GitHub plan and visibility. A failure after promotion leaves an unsigned
tag; admission enforcement must reject it until a full rerun succeeds.

## Release tag is rejected

The tag must exactly match `vX.Y.Z`, and its commit must be reachable from
`origin/main`. Delete and recreate a mistaken unpublished tag locally; do not
move a tag that consumers may already have observed.

## Sync PR does not run checks

PRs created with `GITHUB_TOKEN` can require maintainer approval before their
workflows run. Use a GitHub App installation token or fine-grained PAT as
`SYNC_BRANCH_TOKEN` for unattended required checks.

If the action cannot enable requested auto-merge, the job fails deliberately.
Check token permissions, repository auto-merge settings, and branch rules.

## Govulncheck behavior

Exit code `3` means a reachable vulnerability. Other non-zero codes mean the
tool or build failed and always fail closed. Scheduled baselining may set
`govulncheck-enforce: false` temporarily, with an owner and expiry; PR and
release entrypoints enforce findings.

## Required check unexpectedly skips

A skipped `needs` dependency skips downstream jobs. The public entrypoints use
final aggregate gates to make this visible. Require `PR Gate / PR Gate`, not an
internal scanner job name that can change between v2 releases. With the supplied
caller template, the aggregate check is `check / PR Gate`.

## References

- [GitHub reusable workflows](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows)
- [GitHub artifact attestations](https://docs.github.com/en/actions/how-tos/secure-your-work/use-artifact-attestations/use-artifact-attestations)
- [Cosign](https://docs.sigstore.dev/cosign/)
- [Trivy](https://trivy.dev/latest/docs/)
- [SonarCloud Quality Gates](https://docs.sonarsource.com/sonarqube-cloud/standards/quality-gates/)
