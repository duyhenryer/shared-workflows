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
    uses: duyhenryer/workflows/.github/workflows/go-check.yml@main
    with:
      command-test: 'go test ./...'
      lint: true
    secrets: inherit
```

## 📁 Available Workflows

| Workflow | Purpose | Features |
|----------|---------|----------|
| **[ci-common.yml](.github/workflows/ci-common.yml)** | Common CI checks | PR validation, notifications |
| **[go-check.yml](.github/workflows/go-check.yml)** | Go code quality | Tests, linting, status reporting |
| **[tf-lint.yaml](.github/workflows/tf-lint.yaml)** | Terraform validation | Format check, TFLint analysis |
| **[status.yml](.github/workflows/status.yml)** | Build notifications | Slack notifications, job summaries |
| **[pr-checks.yml](.github/workflows/pr-checks.yml)** | PR handling | Branch validation, CODEOWNERS |

---

> 📖 **For detailed workflow documentation, inputs, outputs, and complete examples**, see [USAGE.md](USAGE.md)

---

## 💡 Quick Examples

**Go Project:**
```yaml
jobs:
  go-check:
    uses: duyhenryer/workflows/.github/workflows/go-check.yml@main
    with:
      command-test: 'go test ./...'
      lint: true
    secrets: inherit
```

**Terraform Project:**
```yaml
jobs:
  terraform-check:
    uses: duyhenryer/workflows/.github/workflows/tf-lint.yaml@main
    with:
      tflint_minimum_failure_severity: 'error'
```

**Status Notification:**
```yaml
jobs:
  notify-status:
    uses: duyhenryer/workflows/.github/workflows/status.yml@main
    with:
      slack_channel_id: "#dev-notifications"
    secrets: inherit
```

---

## 🔐 Required Secrets

- `SLACK_BOT_TOKEN` - For pr-checks.yml and status.yml workflows

Add in repository: Settings → Secrets and variables → Actions

---

## 📚 Documentation

- **[USAGE.md](USAGE.md)** - Complete workflow documentation with examples
- **[.github/workflows/](.github/workflows/)** - Workflow source files

---