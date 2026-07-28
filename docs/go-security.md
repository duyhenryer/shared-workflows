# Go Security Contract

How Go microservices get source and dependency scanning from this shared repository: the
policy, the GitHub settings that enforce it, the rollout, and the exception process.

- Inputs and outputs — [workflows.md](workflows.md)
- Pipeline diagrams — [architecture.md](architecture.md)
- Secrets and prerequisites — [configuration.md](configuration.md)
- Copy-paste caller workflows — [../examples/](../examples/)

## Why one shared aggregate instead of a `security.yml` per service

Copying a scanner workflow into every microservice means every policy change becomes a
fan-out of N pull requests, and the policies drift. Instead there are two layers:

**Layer 1 — one reusable workflow per tool**

- `.github/workflows/codeql.yml`
- `.github/workflows/govulncheck.yml`
- `.github/workflows/dependency-review.yml`

**Layer 2 — one aggregate for Go services**

- `.github/workflows/go-security.yml`

A service calls only the aggregate:

```yaml
uses: duynhlab/gha-workflows/.github/workflows/go-security.yml@main
```

Triggers (`pull_request`, `push`, `schedule`) stay in the service repository; the shared
workflows are `workflow_call` only.

## How it runs

Diagrams for the pull request, push-to-main and scheduled flows are in
[architecture.md](architecture.md) — that is the single place they are maintained. The
behaviour that matters for policy:

| Event | What runs | What blocks |
|-------|-----------|-------------|
| `pull_request` | CodeQL, Govulncheck, Dependency Review | Govulncheck and Dependency Review fail the workflow directly; CodeQL findings are blocked by Code Scanning merge protection, **not** by the job's exit status |
| `push` to main | CodeQL, Govulncheck (Dependency Review self-skips) | `docker-build` waits for `go-security`, so a blocking finding prevents the image from being pushed and Flux never sees a deployable tag |
| `schedule` | CodeQL, Govulncheck | Nothing — refreshes the alert baseline only |

Keep the schedule in its own caller (`security-scheduled.yml`), never in `build.yml` — a
scheduled run must not build or push images, and must not trigger Flux.

## Policy matrix

| Tool | PR | Push main | Weekly | Blocking |
|------|----|-----------|--------|----------|
| Gitleaks | Yes | Yes | Optional | Yes |
| CodeQL Go | Yes | Yes | Yes | Via merge protection |
| CodeQL Actions | Optional in services | Mandatory in this repo | Yes (this repo) | Via merge protection |
| Govulncheck | Yes | Yes | Yes | Yes, after the baseline is clean |
| Dependency Review (vulns) | Yes | No | No | Critical, then tighten to High |
| Dependency Review (licenses) | Yes | No | No | Report, enforce after Legal approval |
| Trivy final image | No | Yes | Existing image | Yes, per image policy |
| Cosign | No | Yes | No | Must precede promotion |

These layers complement each other; none replaces another. Trivy sees the final OS packages
and app dependencies inside the built image, Govulncheck sees which Go vulnerabilities are
actually reachable from your code, Dependency Review sees what the PR is adding.

## Layering rationale

**Why the Security Gate is not a CodeQL severity gate.** A green CodeQL job means the analysis
ran, not that the repository is free of alerts. Severity thresholds are a Code Scanning merge
protection concern; treating `docker-build needs go-security` as a complete severity gate is a
misreading.

**Why Govulncheck runs twice.** `govulncheck -format sarif` always exits 0, even with findings
(only the text handler returns the "vulnerabilities found" error, exit code 3). So text mode
provides the exit code, SARIF mode provides the report, and a final step decides pass/fail. Any
exit code other than 0 or 3 is a tool/build error and fails the job even in report-only mode —
a scan that did not run is not a clean scan.

**Why the license check is a separate invocation.** Vulnerabilities can block today; a license
allowlist needs a baseline period and Legal sign-off. Splitting them means the license policy
can stay report-only without disarming the vulnerability gate.

**Why no Grype filesystem scan (yet).** Trivy already scans the final image, Govulncheck covers
Go dependencies by reachability, and Dependency Review covers new dependencies in PRs. A Grype
path scan adds some coverage but overlaps all three, so it is intentionally out of scope until
the noise level of the current stack is measured. See [Known gaps](#known-gaps).

**Why weekly, not daily.** A daily `0 2 * * *` across dozens of services is a thundering herd
on the runner pool. Weekly with a staggered minute is the baseline GitHub itself uses for
generated CodeQL workflows.

## Required GitHub settings

### Prerequisites

- Dependency Graph enabled.
- GitHub Code Security / Advanced Security enabled for private repositories (required for both
  Code Scanning and the Dependency Review Action).
- If CodeQL **Default Setup** is enabled, switch to **Advanced Setup** — otherwise you get
  duplicate scans and conflicting SARIF categories.
- Service repositories must be allowed to use reusable workflows from this repository
  (org setting: Actions → Policies).
- Self-hosted runners must be new enough for the Node 24 runtime used by
  `dependency-review-action` v5.

### Branch ruleset for `main`

- Require a pull request before merging.
- Block direct pushes (except an audited break-glass role).
- Require status checks:
  - `go-check / Test`
  - `go-check / Integration Test` (if the service runs them)
  - `gitleaks / Gitleaks Scan`
  - `go-security / Security Gate`
  - SonarCloud Quality Gate (if enforced)
- Require branches to be up to date, or use a merge queue.
- Require Code Scanning results: **High or higher** to start. Tighten to Moderate only once the
  team is comfortable — starting too strict pushes people toward bypassing the policy.

Do **not** require `notify` / Slack / Google Sheets jobs. A notification failure must never
block a merge.

If the repository uses a merge queue, add `merge_group` to the trigger list of the security
caller so CodeQL still analyzes the commit that will actually land.

## Rollout

### Phase 0 — Inventory (0.5–1 day)

List every Go service repo and record: public/private, Code Security licence, whether CodeQL
Default Setup is on, Dependency Graph status, root `go.mod` vs nested modules, Go version,
self-hosted runner version, merge-queue usage, whether direct push to `main` is already
blocked. Agree the initial severity policy, and the license policy with Legal/Compliance.

### Phase 1 — Pilot 2–3 services (3–5 days of observation)

Pick one simple service, one with integration tests / testcontainers, and one with many
dependencies or generated code.

Starting modes:

| Setting | Value |
|---------|-------|
| CodeQL Go | enabled, `security-extended` |
| `govulncheck-enforce` | `false` if the repo is not clean yet |
| `dependency-review-severity` | `critical` |
| `allowed-licenses` | empty (license check skipped) |

Watch: workflow duration p50/p95, noise/false-positive count, existing reachable Go
vulnerabilities, SARIF upload failures, runner cost, and Dependabot PR behaviour.

### Phase 2 — Enforce PR policy (1–2 days after the baseline is clean)

- Fix all existing reachable Govulncheck findings, then set `govulncheck-enforce: true`.
- Dependency Review: start at `critical`, then raise to `high`.
- Enable Code Scanning merge protection at High/Critical.
- Make `go-security / Security Gate` a required check.
- Block direct pushes to `main`.
- Enable `license-enforce` only after Legal approves the allowlist and the exception workflow.

### Phase 3 — Main and scheduled scans (1 day)

Add `go-security` to `build.yml`, make `docker-build` and `release-binary` depend on it, add
`security-scheduled.yml` with a staggered weekly cron, and add `go-security` to the `notify`
job's `needs` (without making `notify` a required check).

### Phase 4 — Full rollout

Roll out in batches of 5–10 repositories, watching Security Overview between batches. No
big-bang rollout.

## Version management

All third-party actions are pinned to a full commit SHA with a readable version comment. Never
`@main`, `@master`, or `@latest` for third-party actions.

| Ref | Version |
|-----|---------|
| `actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1` | v7.0.1 |
| `actions/setup-go@b7ad1dad31e06c5925ef5d2fc7ad053ef454303e` | v7.0.0 |
| `github/codeql-action/*@e4fba868fa4b1b91e1fdab776edc8cfbe6e9fb81` | v4.37.3 |
| `actions/dependency-review-action@a1d282b36b6f3519aa1f3fc636f609c47dddb294` | v5.0.0 |
| `golang.org/x/vuln/cmd/govulncheck` | `v1.6.0` |

Dependabot (`.github/dependabot.yml`) opens daily update PRs for the `github-actions`
ecosystem in this repository.

Inside `go-security.yml`, the nested workflows are referenced with relative paths
(`./.github/workflows/codeql.yml`) so they always resolve to the same commit as the aggregate.
Callers in service repositories currently use `@main`, matching the convention of the other
workflows here. If you later move to pinned callers, pin the aggregate to a full SHA with a
`# gha-workflows vX.Y.Z` comment, cut a release tag, never force-move a tag, and let
Dependabot/Renovate raise the update PRs.

## Exception process

Every exception must record: the vulnerability or license ID, the affected repository and
dependency, a business justification, the compensating control, an owner, an expiry date, and a
ticket link. Permanent, unowned ignores are not acceptable.

| Case | Mechanism |
|------|-----------|
| Dependency Review vulnerability | `allowed-ghsas` (GHSA IDs) |
| Dependency Review license | `allowed-dependencies-licenses` (purls) |
| Govulncheck finding | `govulncheck-enforce: false`, time-boxed |

Govulncheck has no stable native ignore/silence mechanism, which is why its only exception is
turning enforcement off for that repository — a blunt instrument, so it needs an expiry date and
a follow-up ticket. The preferred path is: report-only → inventory → fix the reachable findings
→ enforce.

Prefer managing exceptions centrally here rather than letting each caller widen the policy on
its own.

## Known gaps

- **No Grype filesystem scan.** Deliberate (see rationale above). If added later, start at
  report-only, then `block-critical`, and only then consider `block-high`.
- **This repository does not yet scan its own workflows with CodeQL.** CodeQL Actions analysis
  in a service repo does **not** cover the reusable workflows that live here, so the shared repo
  must eventually scan itself. It cannot call `go-security.yml` to do so — there is no `go.mod`
  here, so `codeql-go` and `govulncheck` would fail. The self-CI workflow must call
  `codeql.yml` directly with `language: actions` and `build-mode: none`.
- **Internal composite actions are still referenced as `@main`** in `go-check.yml`,
  `gitleaks.yml` and `status.yml`. The security workflows added here use no composite actions,
  so they are fully pinned; the older workflows are not.
