# 📖 Workflow Usage Guide

Detailed documentation for all available workflows with inputs, outputs, and examples.

## 🔍 pr-checks.yml

**Purpose:** Pull request validation and notifications  
**Trigger:** PR events only  
**Features:** Branch validation, CODEOWNERS tagging, event notifications

### Usage

```yaml
jobs:
  pr-checks:
    if: github.event_name == 'pull_request'
    uses: duyhenryer/shared-workflows/.github/workflows/pr-checks.yml@main
    secrets:
      SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

### Required Secret

- `SLACK_BOT_TOKEN` - Slack bot authentication token

---

## 🐹 go-check.yml

**Purpose:** Go code quality assurance  
**Trigger:** PR events  
**Features:** Unit testing, linting, status reporting

### Inputs

| Parameter | Type | Default | Required | Description |
|-----------|------|---------|----------|-------------|
| `command-test` | string | - | **Yes** | Test command to execute |
| `setup-go` | boolean | `true` | No | Install Go automatically |
| `gomod-path` | string | `"go.mod"` | No | Path to go.mod file |
| `lint` | boolean | `false` | No | Enable linting |
| `lint-path` | string | `".golangci.yml"` | No | Lint config file path |
| `lint-timeout` | string | `"10m"` | No | Lint timeout duration |
| `runs-on` | string | `"ubuntu-latest"` | No | Runner type |

### Usage

**Basic:**
```yaml
jobs:
  go-check:
    uses: duyhenryer/shared-workflows/.github/workflows/go-check.yml@main
    with:
      command-test: 'go test ./...'
    secrets: inherit
```

---

## 💡 Complete Examples

### Go Project Pipeline

```yaml
name: CI Pipeline

on:
  push:
    branches: [main, dev]
  pull_request:
    branches: [main, dev]

jobs:
  # PR Validation
  pr-checks:
    if: github.event_name == 'pull_request'
    uses: duyhenryer/shared-workflows/.github/workflows/pr-checks.yml@main
    secrets:
      SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}

  # Go quality checks
  go-check:
    uses: duyhenryer/shared-workflows/.github/workflows/go-check.yml@main
    with:
      command-test: 'go test -race -coverprofile=coverage.out -covermode=atomic ./...'
      lint: true
    secrets: inherit

  # Notify status
  notify:
    needs: [go-check]
    if: always()
    uses: duyhenryer/shared-workflows/.github/workflows/status.yml@main
    with:
      slack_channel_id: "#dev-notifications"
    secrets:
      SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

---

## ⚙️ Configuration

### Required Secrets

Set these in your repository settings (Settings → Secrets and variables → Actions):

| Secret | Used By | Description |
|--------|---------|-------------|
| `SLACK_BOT_TOKEN` | pr-checks.yml, status.yml | Slack bot token for notifications |

### Repository Setup

1. **Create `.github/CODEOWNERS` file:**
```gitignore
# Global owners
* @duyhenryer @team-lead

# Language-specific
*.go @team-backend
*.tf @team-infra

# Directory-specific
/cmd @team-backend
/terraform @team-infra
```

2. **Configure Slack channels** (optional):
   - `#ci-alert` - General CI notifications
   - `#pull-request-main` - PR notifications for main branch
   - `#pull-request-dev` - PR notifications for dev branch
   - `#dev-notifications` - Development updates

---

## 🐛 Troubleshooting

### "SLACK_BOT_TOKEN not found"

**Solution:** Add the secret in repository settings:
1. Go to Settings → Secrets and variables → Actions
2. Click "New repository secret"
3. Name: `SLACK_BOT_TOKEN`
4. Value: Your Slack bot token

### "Branch validation failed"

**Solution:** Use correct branch naming:
- ✅ `hotfix/critical-bug`
- ✅ `release/v1.2.3`
- ❌ `fix-something` (not allowed)

### "CODEOWNERS not found"

**Solution:** Create `.github/CODEOWNERS` file in your repository root.

### "Lint timeout exceeded"

**Solution:** Increase timeout in workflow:
```yaml
with:
  lint-timeout: '20m'  # Increase from default 10m
```

### "TFLint config not found"

**Solution:** Either:
1. Create a `.tflint.hcl` file, or
2. Specify custom path:
```yaml
with:
  tflint_config_path: 'terraform/.tflint.hcl'
```

---

## 📚 Additional Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Go Testing](https://golang.org/doc/code.html#Testing)
- [TFLint Documentation](https://github.com/terraform-linters/tflint)
- [Slack API](https://api.slack.com/)

---

<div align="center">

**Need help?** [Open an issue](https://github.com/duyhenryer/workflows/issues)

[⬆ Back to Top](#-workflow-usage-guide)

</div>
