# Example Caller Workflows for a Go Service

Copy-paste starting points for onboarding a Go microservice onto the shared security stack.
Each file is a **caller** that lives in the service repository; the scanners themselves live in
this repository.

| Example | Copy to | Trigger |
|---------|---------|---------|
| [go-service-check.yml](go-service-check.yml) | `.github/workflows/check.yml` | `pull_request` |
| [go-service-build.yml](go-service-build.yml) | `.github/workflows/build.yml` | `push` to `main`, `v*` tags |
| [go-service-security-scheduled.yml](go-service-security-scheduled.yml) | `.github/workflows/security-scheduled.yml` | weekly `schedule` + `workflow_dispatch` |

Before copying, replace: `project-key`, `organization`, `image-name`, and `slack_channel_id`.

Contract and policy details: [../docs/go-security.md](../docs/go-security.md).
Inputs and outputs: [../USAGE.md](../USAGE.md).

---

## What one `go-security` call expands into

A single job in your caller fans out into four scanners and one gate:

```mermaid
flowchart LR
    CALL["go-security<br/>(1 job in your workflow)"] --> CQGO["codeql-go<br/>autobuild · /language:go"]
    CALL --> CQACT["codeql-actions<br/>none · /language:actions<br/>(scan-actions: true)"]
    CALL --> GV["govulncheck<br/>text gate + SARIF"]
    CALL --> DR["dependency-review<br/>pull_request only"]

    CQGO --> GATE["Security Gate"]
    CQACT --> GATE
    GV --> GATE
    DR --> GATE

    CQGO --> TAB["GitHub Security tab"]
    CQACT --> TAB
    GV --> TAB

    style GATE fill:#3b82f6,color:#fff
```

Check names that appear on the PR (require the gate, not the individual scanners):

```text
go-security / Security Gate
go-security / codeql-go / Analyze (go)
go-security / govulncheck / Govulncheck
go-security / dependency-review / Dependency Review
```

---

## After applying `check.yml` (pull request)

```mermaid
flowchart TD
    PR["Pull Request"] --> PRC["pr-checks"]
    PR --> GC["go-check<br/>test · lint"]
    PR --> GL["gitleaks"]
    PR --> SEC["go-security"]

    GC --> SONAR["sonar"]
    GL --> SONAR

    SEC --> GATE["Security Gate"]

    PRC --> NOTIFY["notify<br/>(never a required check)"]
    GC --> NOTIFY
    GL --> NOTIFY
    SONAR --> NOTIFY
    GATE --> NOTIFY

    GATE --> RULES["Ruleset gate"]
    SONAR --> RULES
    GC --> RULES
    GL --> RULES
    RULES --> MERGE["Merge"]

    style GATE fill:#3b82f6,color:#fff
    style RULES fill:#f59e0b,color:#fff
```

`go-security` runs in parallel with everything else — it is intentionally **not** chained behind
`sonar`, which would add its full duration to the PR critical path.

The ruleset gate is what actually blocks the merge:

| Required check | Source |
|----------------|--------|
| `go-check / Test` | `go-check.yml` |
| `gitleaks / Gitleaks Scan` | `gitleaks.yml` |
| `go-security / Security Gate` | `go-security.yml` |
| SonarCloud Quality Gate | SonarCloud app |
| Code Scanning merge protection (High+) | CodeQL alerts, **not** the job status |

---

## After applying `build.yml` (push to main)

```mermaid
flowchart TD
    PUSH["Push main / v* tag"] --> GC["go-check"]
    PUSH --> GL["gitleaks"]
    PUSH --> SEC["go-security<br/>dependency-review self-skips"]

    GC --> SONAR["sonar"]
    GL --> SONAR

    SONAR --> BUILD["docker-build<br/>--load → Trivy → push"]
    SEC --> BUILD
    GC --> BUILD
    GL --> BUILD

    BUILD -->|"scan pass"| SIGN["docker-sign<br/>Cosign keyless"]
    BUILD -.->|"optional"| RPT["trivy-report<br/>SARIF + Sheets"]
    SEC -.->|"Security Gate fail"| BLOCKED["Image NOT pushed<br/>→ Flux sees no new tag"]
    BUILD -.->|"scan fail"| BLOCKED

    SIGN --> PROMOTE["Promote deployable tag"]
    PROMOTE --> FLUX["Flux auto deploy"]

    style BLOCKED fill:#ef4444,color:#fff
    style SEC fill:#3b82f6,color:#fff
```

`docker-build` and `release-binary` list their dependencies explicitly
(`needs: [go-check, gitleaks, sonar, go-security]`) rather than inheriting `gitleaks` through the
`sonar` edge, so the gate graph is auditable at a glance.

> **Watch out:** a *skipped* `needs` dependency skips every downstream job. That is why the
> build example does **not** exclude tags from `gitleaks` — doing so would silently turn a tag
> build into a no-op.

---

## After applying `security-scheduled.yml` (weekly)

```mermaid
flowchart LR
    CRON["cron '17 2 * * 1'<br/>+ workflow_dispatch"] --> SEC["go-security"]
    SEC --> CQ["CodeQL (go)"]
    SEC --> GV["Govulncheck"]
    SEC -.->|"skipped: not a PR"| DR["Dependency Review"]
    CQ --> TAB["Security tab<br/>alert baseline refreshed"]
    GV --> TAB
    SEC -.-> NOBUILD["No image build<br/>No GHCR push<br/>No Flux deploy"]

    style NOBUILD fill:#6b7280,color:#fff
```

The schedule lives in its own file, never in `build.yml`, precisely so a scheduled run cannot
push an image or trigger a deployment.

---

## Onboarding order

```mermaid
flowchart LR
    A["1. Prerequisites<br/>Dependency Graph<br/>Code Security<br/>CodeQL Advanced Setup"] --> B["2. check.yml<br/>govulncheck-enforce: false"]
    B --> C["3. Fix the baseline<br/>reachable vulns"]
    C --> D["4. enforce: true<br/>+ required checks<br/>+ merge protection"]
    D --> E["5. build.yml<br/>docker-build needs go-security"]
    E --> F["6. security-scheduled.yml<br/>staggered weekly cron"]
```

Start with `govulncheck-enforce: false`. Turning enforcement on before the existing reachable
findings are fixed turns every PR red at once, and govulncheck has no stable per-finding ignore
mechanism to escape with. Keep `allowed-licenses` empty until Legal approves a policy — empty
skips the license check rather than enforcing someone else's allowlist.
