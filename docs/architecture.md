# Architecture

How the reusable workflows in this repository fit together. This is the single source for
repository-level pipeline diagrams — per-example diagrams live in
[../examples/README.md](../examples/README.md).

- Workflow inputs and outputs — [workflows.md](workflows.md)
- Security policy and rollout — [go-security.md](go-security.md)

## Builder interface

`docker-build-go.yml` is the Go-specific builder with integrated Trivy scanning. Other stacks
(`docker-build-node.yml`, and any future `docker-build-python.yml`) implement the **same output
interface** — `tags` + `digest` + `scan-status` — so they share the downstream sign and report
workflows.

```mermaid
flowchart TD
    subgraph builders ["Stack-specific builders (scan-before-push)"]
        GO["docker-build-go.yml<br/>(--load → Trivy → push)"]
        NODE["docker-build-node.yml<br/>(--load → Trivy → push)"]
        PYTHON["docker-build-python.yml (future)"]
    end
    subgraph shared ["Shared downstream"]
        SIGN["docker-sign.yml"]
        TRIVY_RPT["trivy-scan.yml<br/>(SARIF reporting, optional)"]
    end
    GO -->|"outputs: tags, digest, scan-status"| SIGN
    NODE -->|"outputs: tags, digest, scan-status"| SIGN
    PYTHON -->|"outputs: tags, digest, scan-status"| SIGN
    GO -.->|"optional"| TRIVY_RPT
    NODE -.->|"optional"| TRIVY_RPT
```

The builders differ in tagging: the Go builder emits **immutable tags only**, the Node builder
also publishes `:latest` and branch tags. See
[workflows.md — Container images](workflows.md#container-images).

## Overall CI flow

Canonical high-level flow with scan-before-push.

```mermaid
flowchart TD
    subgraph prPush ["PR / Push (excluding tags)"]
        PRCHECKS["pr-checks.yml"]
        GOCHECK["go-check.yml"]
        GITLEAKS["gitleaks.yml"]
        SEC["go-security.yml<br/>(CodeQL + Govulncheck + Dep Review)"]
        SONAR["sonarqube.yml"]
    end

    subgraph buildOnly ["Push to main/dev only"]
        BUILD["docker-build-go.yml<br/>(--load → Trivy → push)"]
        SIGN["docker-sign.yml"]
        TRIVY_RPT["trivy-scan.yml<br/>(optional report)"]
    end

    NOTIFY["status.yml"]

    GOCHECK --> SONAR
    GITLEAKS --> SONAR
    SONAR --> BUILD
    SEC --> BUILD
    BUILD --> SIGN
    BUILD -.-> TRIVY_RPT

    PRCHECKS --> NOTIFY
    GOCHECK --> NOTIFY
    GITLEAKS --> NOTIFY
    SEC --> NOTIFY
    SONAR --> NOTIFY
    BUILD --> NOTIFY
    SIGN --> NOTIFY
    TRIVY_RPT --> NOTIFY
```

> `go-security.yml` runs **in parallel** with `go-check` / `gitleaks` / `sonarqube` — it is not
> chained behind Sonar, which would add its full duration to the PR critical path. But
> `docker-build` waits for it, so a blocking Govulncheck finding prevents the image from ever
> being pushed.

## Pull request

Code quality, secret scanning and security scanning run in parallel. Docker jobs are skipped.

```mermaid
flowchart TD
    subgraph pr_flow ["PR Flow"]
        PR["pr-checks.yml"]
        GOCHECK["go-check.yml"]
        GITLEAKS["gitleaks.yml"]
        SONAR["sonarqube.yml"]
        NOTIFY["status.yml"]

        subgraph sec ["go-security.yml"]
            CQGO["codeql-go"]
            CQACT["codeql-actions<br/>(optional)"]
            GOVULN["govulncheck"]
            DEPREV["dependency-review"]
            GATE["Security Gate"]

            CQGO --> GATE
            CQACT --> GATE
            GOVULN --> GATE
            DEPREV --> GATE
        end

        GOCHECK --> SONAR
        GITLEAKS --> SONAR
        PR --> NOTIFY
        GOCHECK --> NOTIFY
        GITLEAKS --> NOTIFY
        SONAR --> NOTIFY
        GATE --> NOTIFY
        GATE --> MERGE["Merge<br/>(ruleset: required checks<br/>+ Code Scanning merge protection)"]
    end

    style GATE fill:#3b82f6,color:#fff
```

Required status checks to configure on `main`: `go-check / Test`, `gitleaks / Gitleaks Scan`,
`go-security / Security Gate`, the SonarCloud Quality Gate, plus a Code Scanning merge
protection rule for CodeQL alerts. Do **not** require `notify`.

## Push to main

Full pipeline: build with integrated scan, then sign. If the pre-push scan fails the image is
never pushed and signing is skipped automatically.

```mermaid
flowchart TD
    subgraph main_flow ["Main Flow"]
        GOCHECK2["go-check.yml"]
        GITLEAKS2["gitleaks.yml"]
        SONAR2["sonarqube.yml"]

        BUILD["docker-build-go.yml<br/>(--load → Trivy → push)"]
        SIGN["docker-sign.yml"]
        TRIVY_RPT["trivy-scan.yml<br/>(SARIF report, optional)"]

        DBINIT["docker-build-go.yml (migration)<br/>(--load → Trivy → push)"]

        NOTIFY2["status.yml"]

        SEC2["go-security.yml<br/>(Dependency Review self-skips on push)"]

        GOCHECK2 --> SONAR2
        GITLEAKS2 --> SONAR2
        SONAR2 --> BUILD
        SEC2 --> BUILD
        SONAR2 --> DBINIT
        SEC2 --> DBINIT
        BUILD -->|"scan pass → pushed"| SIGN
        BUILD -.->|"scan fail"| BUILD_FAIL["Image NOT pushed"]
        SEC2 -.->|"Security Gate fail"| BUILD_FAIL
        BUILD -.->|"optional"| TRIVY_RPT

        BUILD --> NOTIFY2
        SIGN --> NOTIFY2
        TRIVY_RPT --> NOTIFY2
        GITLEAKS2 --> NOTIFY2
        SEC2 --> NOTIFY2
        DBINIT --> NOTIFY2
    end

    style BUILD fill:#3b82f6,color:#fff
    style BUILD_FAIL fill:#ef4444,color:#fff
```

> Because `docker-build` waits for `go-security`, a Govulncheck gate failure means no new image
> reaches GHCR — so Flux never sees a deployable tag. This is a source and dependency gate; the
> final-image Trivy gate, SBOM and Cosign remain a separate supply-chain layer.

## Scheduled security scan

Weekly source and dependency re-scan, decoupled from the build so it can never push an image or
trigger a deployment.

```mermaid
flowchart LR
    CRON["schedule: '17 2 * * 1'<br/>+ workflow_dispatch"] --> SEC3["go-security.yml"]
    SEC3 --> CQ3["CodeQL (go)"]
    SEC3 --> GV3["Govulncheck"]
    SEC3 -.->|"skipped: not a PR"| DR3["Dependency Review"]
    CQ3 --> TAB["GitHub Security tab"]
    GV3 --> TAB
```

Use a weekly cron with a staggered minute (17 / 29 / 41) across repository groups — a daily
`0 2 * * *` across dozens of services creates a thundering herd on the runners.
