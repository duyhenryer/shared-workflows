# Shared GA - Reusable Workflows

> Centralized reusable workflows for consistent CI/CD across all repositories

## 🚀 Quick Start

Use any workflow by referencing it with the `uses` keyword:

```yaml
name: My Workflow
on:
  pull_request:
    branches: [main, dev]

jobs:
  check:
    uses: duyhenryer/shared-workflows/.github/workflows/go-check.yml@main
    with:
      command-test: 'go test ./...'
      lint: true
    secrets: inherit
```

## 📁 Available Workflows

| Workflow | Purpose | Features |
|----------|---------|----------|
| **[pr-checks.yml](.github/workflows/pr-checks.yml)** | PR handling | Branch validation, CODEOWNERS, Slack notifications |
| **[go-check.yml](.github/workflows/go-check.yml)** | Go code quality | Tests, linting, coverage artifacts |
| **[docker-build.yml](.github/workflows/docker-build.yml)** | Docker build & push | Orchestrates build → sign, GHCR, SBOM |
| **[docker-build-go.yml](.github/workflows/docker-build-go.yml)** | Docker build (Go) | Multi-platform, caching, provenance |
| **[docker-sign.yml](.github/workflows/docker-sign.yml)** | Cosign image signing | Keyless OIDC signing |
| **[sonarqube.yml](.github/workflows/sonarqube.yml)** | SonarCloud analysis | Go coverage, Quality Gate |
| **[tf-lint.yml](.github/workflows/tf-lint.yml)** | Terraform validation | Format check, TFLint analysis |
| **[status.yml](.github/workflows/status.yml)** | Build notifications | Slack, Google Sheets, job summaries |

---

> 📖 **For detailed workflow documentation, inputs, outputs, and complete examples**, see [USAGE.md](USAGE.md)

---

## 💡 Quick Examples

**Go Project:**
```yaml
jobs:
  go-check:
    uses: duyhenryer/shared-workflows/.github/workflows/go-check.yml@main
    with:
      command-test: 'go test ./...'
      lint: true
    secrets: inherit
```

**Terraform Project:**
```yaml
jobs:
  terraform-check:
    uses: duyhenryer/shared-workflows/.github/workflows/tf-lint.yml@main
    with:
      tflint_minimum_failure_severity: 'error'
```

**Status Notification:**
```yaml
jobs:
  notify-status:
    uses: duyhenryer/shared-workflows/.github/workflows/status.yml@main
    with:
      slack_channel_id: "#dev-notifications"
    secrets: inherit
```

---

## 🔐 Required Secrets

| Secret | Used By | Required | Description |
|--------|---------|----------|-------------|
| `SLACK_BOT_TOKEN` | pr-checks.yml, status.yml | **Yes** | Slack bot token for notifications |
| `SONAR_TOKEN` | sonarqube.yml | **Yes** | SonarCloud authentication token |
| `GSHEET_CLIENT_EMAIL` | status.yml | No | Google service account email (Sheets reporting) |
| `GSHEET_PRIVATE_KEY` | status.yml | No | Google service account private key (Sheets reporting) |

Add in repository: Settings → Secrets and variables → Actions

---

## 📚 Documentation

- **[USAGE.md](USAGE.md)** - Complete workflow documentation with examples
- **[.github/workflows/](.github/workflows/)** - Workflow source files

---