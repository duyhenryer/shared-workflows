# Troubleshooting

Common failures when calling these workflows, and what to do about them.

## "SLACK_BOT_TOKEN not found"

Add the secret in the calling repository: Settings → Secrets and variables → Actions → New
repository secret, named `SLACK_BOT_TOKEN`. See [configuration.md](configuration.md#required-secrets).

## "Branch validation failed"

`pr-checks.yml` enforces gitflow-style prefixes. Use `hotfix/critical-bug`, `release/v1.2.3`,
`fix/something` — not a bare `fix-something`.

## "CODEOWNERS not found" or Slack shows no owners

Create a CODEOWNERS file with a global owners line (`* @user1 @user2`). The workflow looks in
`.github/CODEOWNERS`, then `CODEOWNERS` (root), then `docs/CODEOWNERS`. See
[configuration.md#codeowners](configuration.md#codeowners).

## "Lint timeout exceeded"

```yaml
with:
  lint-timeout: '20m'   # default is 10m
```

## "TFLint config not found"

Either create a `.tflint.hcl`, or point at yours:

```yaml
with:
  tflint_config_path: 'terraform/.tflint.hcl'
```

## "SonarCloud Quality Gate failed"

Fix what SonarCloud reports, or temporarily stop the gate from failing the job:

```yaml
with:
  fail-on-quality-gate: false
```

If the failure is coverage on code that is not meaningfully testable (bootstrap, wiring,
migrations, mocks), the right fix is usually `coverage-exclusions`, not disabling the gate.

## Trivy is blocking the image push

The push is gated by **`scan-block-severity`**, not by `scan-exit-code`. `scan-exit-code` is
deprecated and no longer affects the gate — setting it to `'0'` will not unblock anything.

By default `CRITICAL,HIGH` are reported and only `CRITICAL` blocks. Options, in order of
preference:

**1. Suppress a specific known upstream CVE** — keeps the gate intact:

```yaml
with:
  scan-trivyignores: '.trivyignore'   # path relative to the build context
```

**2. Narrow what blocks** (already the default; use this if you had widened it):

```yaml
with:
  scan-severity: 'CRITICAL,HIGH'    # reported in the summary
  scan-block-severity: 'CRITICAL'   # only these block the push
```

**3. Turn the scan off entirely** — last resort:

```yaml
with:
  scan-before-push: false
```

## Govulncheck fails every PR

Expected on a repository that has never been scanned. Baseline it first:

```yaml
with:
  govulncheck-enforce: false   # report-only
```

Fix the reachable findings, then set it back to `true`. Note that govulncheck has no stable
per-finding ignore mechanism, so `enforce: false` is the only escape hatch — give it an expiry
date and a ticket. See [go-security.md](go-security.md#exception-process).

## Govulncheck fails even with `enforce: false`

That is by design for **tool and build errors**. Exit code `3` means "vulnerabilities found"
and respects `enforce`; any other non-zero exit means govulncheck could not complete (usually a
build failure in the scanned module), and that always fails the job — a scan that did not run is
not a clean scan. Check the job log for the compile error.

## Security Gate fails but every scanner looks green

The gate also rejects `cancelled` results, and it requires `dependency-review` to have actually
run on a pull request. If `dependency-review` shows as `skipped` on a PR, the usual cause is a
missing prerequisite — Dependency Graph disabled, or GitHub Code Security not enabled on a
private repository. See [configuration.md](configuration.md#security-prerequisites).

## CodeQL job is green but alerts still exist

That is expected. A green CodeQL job means the analysis ran, not that the repository is clean.
Alert severity is enforced by a **Code Scanning merge protection** rule in the branch ruleset,
not by the job status. See [go-security.md](go-security.md#branch-ruleset-for-main).

## A downstream job was skipped for no obvious reason

A **skipped** `needs` dependency skips everything downstream. If you add an `if:` guard to a job
that others depend on — for example excluding tags from `gitleaks` — the whole chain below it
silently disappears instead of failing. Either drop the guard or add `if: always()` plus an
explicit result check on the dependent job.

## Additional resources

- [GitHub Actions documentation](https://docs.github.com/en/actions)
- [Reusing workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
- [CodeQL code scanning](https://docs.github.com/en/code-security/code-scanning)
- [govulncheck](https://pkg.go.dev/golang.org/x/vuln/cmd/govulncheck)
- [TFLint](https://github.com/terraform-linters/tflint)
- [SonarCloud](https://docs.sonarsource.com/sonarcloud/)
- [Cosign](https://docs.sigstore.dev/cosign/overview/)
- [Trivy](https://trivy.dev/latest/docs/)
- [Slack API](https://api.slack.com/)

---

**Need help?** [Open an issue](https://github.com/duynhlab/gha-workflows/issues)
