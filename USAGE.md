# 📖 Workflow Usage Guide

Detailed documentation for all available workflows with inputs, outputs, and examples.

## 🔄 ci-common.yml

**Purpose:** Main entry point for standard CI checks  
**Trigger:** Manual or PR events  
**Features:** PR validation, notifications, CODEOWNERS integration

### Usage

```yaml
jobs:
  ci:
    uses: duyhenryer/workflows/.github/workflows/ci-common.yml@main
    secrets: inherit
```

### What It Does

- Runs on pull request events
- Validates branch naming (hotfix/*, release/*)
- Sends Slack notifications to CODEOWNERS
- Integrates with PR checks workflow

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
    uses: duyhenryer/workflows/.github/workflows/go-check.yml@main
    with:
      command-test: 'go test ./...'
    secrets: inherit
```

**With Linting:**
```yaml
jobs:
  go-check:
    uses: duyhenryer/workflows/.github/workflows/go-check.yml@main
    with:
      command-test: 'go test -v -race ./...'
      lint: true
      lint-path: '.golangci.yml'
      lint-timeout: '15m'
    secrets: inherit
```

**Custom Go Version:**
```yaml
jobs:
  go-check:
    uses: duyhenryer/workflows/.github/workflows/go-check.yml@main
    with:
      command-test: 'go test ./...'
      gomod-path: 'cmd/service/go.mod'
      setup-go: true
    secrets: inherit
```

---

## 🏗️ tf-lint.yaml

**Purpose:** Terraform infrastructure validation  
**Trigger:** Manual or on terraform changes  
**Features:** Format check, TFLint analysis, plugin caching

### Inputs

| Parameter | Type | Default | Required | Description |
|-----------|------|---------|----------|-------------|
| `tflint_config_path` | string | - | No | Custom TFLint config file path |
| `tflint_minimum_failure_severity` | string | `"warning"` | No | Minimum failure severity (error, warning) |
| `tflint_force` | boolean | `false` | No | Force TFLint execution |

### Usage

**Basic:**
```yaml
jobs:
  terraform-check:
    uses: duyhenryer/workflows/.github/workflows/tf-lint.yaml@main
```

**With Custom Config:**
```yaml
jobs:
  terraform-check:
    uses: duyhenryer/workflows/.github/workflows/tf-lint.yaml@main
    with:
      tflint_config_path: 'terraform/.tflint.hcl'
      tflint_minimum_failure_severity: 'error'
```

**Strict Mode:**
```yaml
jobs:
  terraform-check:
    uses: duyhenryer/workflows/.github/workflows/tf-lint.yaml@main
    with:
      tflint_minimum_failure_severity: 'error'
      tflint_force: true
```

### What It Checks

- **terraform-fmt**: Validates code formatting
- **tflint**: Runs comprehensive lint analysis
  - Configuration issues
  - Best practices
  - Security vulnerabilities
  - Provider-specific rules

---

## 🐳 docker-build.yml

**Purpose:** Docker image build, push to GHCR, and Cosign signing  
**Trigger:** workflow_call  
**Features:** Multi-platform builds, Cosign keyless signing, SBOM generation, GitHub Cache

### Inputs

| Parameter | Type | Default | Required | Description |
|-----------|------|---------|----------|-------------|
| `image-name` | string | - | **Yes** | Name of the Docker image |
| `dockerfile` | string | `"Dockerfile"` | No | Path to Dockerfile |
| `context` | string | `"."` | No | Docker build context |
| `push` | boolean | `false` | No | Push image to registry |
| `platforms` | string | `"linux/amd64"` | No | Target platforms |
| `build-args` | string | `""` | No | Build-time variables |
| `tags` | string | `""` | No | Custom tags (comma-separated) |
| `cosign` | boolean | `true` | No | Sign with Cosign |
| `sbom` | boolean | `false` | No | Generate SBOM attestation |
| `runs-on` | string | `"ubuntu-latest"` | No | Runner type |

### Outputs

| Output | Description |
|--------|-------------|
| `image` | Full image tags |
| `digest` | Image digest |

### Usage

**Basic (PR validation):**
```yaml
jobs:
  docker:
    uses: duyhenryer/shared-workflows/.github/workflows/docker-build.yml@main
    with:
      image-name: 'my-service'
    secrets: inherit
```

**Push to GHCR:**
```yaml
jobs:
  docker:
    uses: duyhenryer/shared-workflows/.github/workflows/docker-build.yml@main
    with:
      image-name: 'my-service'
      push: true
      cosign: true
    secrets: inherit
```

**Multi-platform with SBOM:**
```yaml
jobs:
  docker:
    uses: duyhenryer/shared-workflows/.github/workflows/docker-build.yml@main
    with:
      image-name: 'my-service'
      push: true
      platforms: 'linux/amd64,linux/arm64'
      sbom: true
    secrets: inherit
```

---

## 🔬 sonarqube.yml

**Purpose:** SonarCloud code analysis with Go coverage  
**Trigger:** workflow_call  
**Features:** Go test coverage, SonarCloud scan, Quality Gate check

### Inputs

| Parameter | Type | Default | Required | Description |
|-----------|------|---------|----------|-------------|
| `project-key` | string | - | **Yes** | SonarCloud project key |
| `organization` | string | - | **Yes** | SonarCloud organization |
| `sources` | string | `"."` | No | Source directories |
| `exclusions` | string | `"**/vendor/**,..."` | No | Patterns to exclude |
| `go-version` | string | `"1.25"` | No | Go version |
| `coverage-path` | string | `"coverage.out"` | No | Coverage file path |
| `test-command` | string | `"go test ..."` | No | Test command |
| `quality-gate-wait` | boolean | `true` | No | Wait for Quality Gate |
| `quality-gate-timeout` | number | `300` | No | QG timeout (seconds) |
| `runs-on` | string | `"ubuntu-latest"` | No | Runner type |

### Required Secrets

| Secret | Description |
|--------|-------------|
| `SONAR_TOKEN` | SonarCloud authentication token |

### Usage

**Basic:**
```yaml
jobs:
  sonar:
    uses: duyhenryer/shared-workflows/.github/workflows/sonarqube.yml@main
    with:
      project-key: 'duynhne_my-service'
      organization: 'duynhne'
    secrets: inherit
```

**Custom test command:**
```yaml
jobs:
  sonar:
    uses: duyhenryer/shared-workflows/.github/workflows/sonarqube.yml@main
    with:
      project-key: 'duynhne_my-service'
      organization: 'duynhne'
      test-command: 'go test -race -coverprofile=coverage.out ./...'
      go-version: '1.25.5'
    secrets: inherit
```

---

## 📢 status.yml

**Purpose:** Build status notifications with detailed reporting  
**Trigger:** Always runs (workflow_call)  
**Features:** Slack notifications, job summaries, smart status detection

### Inputs

| Parameter | Type | Default | Required | Description |
|-----------|------|---------|----------|-------------|
| `slack_channel_id` | string | `""` | No | Target Slack channel |
| `slack_msg` | string | Auto-generated | No | Custom Slack message |
| `runs-on` | string | `"ubuntu-latest"` | No | Runner type |

### Usage

**Basic:**
```yaml
jobs:
  notify-status:
    needs: [test, build]
    uses: duyhenryer/workflows/.github/workflows/status.yml@main
    secrets: inherit
```

**With Custom Message:**
```yaml
jobs:
  notify-status:
    needs: [test, build]
    uses: duyhenryer/workflows/.github/workflows/status.yml@main
    with:
      slack_channel_id: "#dev-notifications"
      slack_msg: "Build #${{ github.run_number }} completed for ${{ github.repository }}"
    secrets: inherit
```

### Required Secret

- `SLACK_BOT_TOKEN` - Slack bot authentication token

### Features

- Creates detailed job summary table
- Handles failed, cancelled, and successful states
- Configurable allowed failure jobs
- Automatic workflow status detection
- Clickable job links in notifications

---

## 🔍 pr-checks.yml

**Purpose:** Pull request validation and notifications  
**Trigger:** PR events only  
**Features:** Branch validation, CODEOWNERS tagging, event notifications

### Inputs

| Parameter | Type | Default | Required | Description |
|-----------|------|---------|----------|-------------|
| `runs-on` | string | `"ubuntu-latest"` | No | Runner type |

### Usage

```yaml
jobs:
  pr-validation:
    uses: duyhenryer/workflows/.github/workflows/pr-checks.yml@main
    secrets:
      SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

### Required Secret

- `SLACK_BOT_TOKEN` - Slack bot authentication token

### What It Does

**validate-pr Job:**
- Only runs on PRs targeting `main` branch
- Validates source branch naming:
  - ✅ `hotfix/*`
  - ✅ `release/*`
  - ❌ Other patterns

**notify-pr-events Job:**
- Reads CODEOWNERS file
- Extracts reviewer list
- Sends Slack notifications to `#pull-request-{base-branch}`
- Notifies on: created, updated, review_requested, etc.

### Setup

Create `.github/CODEOWNERS` file:
```gitignore
# Global code owners
* @duyhenryer

# Specific paths
/terraform @team-infra
/cmd @team-backend
```

---

## 💡 Complete Examples

### Go ProjectPipeline

```yaml
name: CI Pipeline

on:
  push:
    branches: [main, dev]
  pull_request:
    branches: [main, dev]

jobs:
  # Common checks (PR validation, notifications)
  common-checks:
    uses: duyhenryer/workflows/.github/workflows/ci-common.yml@main
    secrets: inherit

  # Go quality checks
  go-quality:
    uses: duyhenryer/workflows/.github/workflows/go-check.yml@main
    with:
      command-test: 'go test -race -coverprofile=coverage.out -covermode=atomic ./...'
      lint: true
      lint-path: '.golangci.yml'
      lint-timeout: '15m'
    secrets: inherit

  # Terraform checks (if applicable)
  terraform-check:
    if: contains(github.event.head_commit.modified, '*.tf')
    uses: duyhenryer/workflows/.github/workflows/tf-lint.yaml@main
    with:
      tflint_config_path: 'terraform/.tflint.hcl'
      tflint_minimum_failure_severity: 'error'

  # Notify status
  notify-status:
    needs: [go-quality, terraform-check]
    if: always()
    uses: duyhenryer/workflows/.github/workflows/status.yml@main
    with:
      slack_channel_id: "#dev-notifications"
    secrets: inherit
```

### Terraform-Only Project

```yaml
name: Terraform CI

on:
  pull_request:
    paths: 
      - 'terraform/**'
      - '*.tf'
      - '.github/workflows/terraform-*.yml'

jobs:
  terraform-validation:
    uses: duyhenryer/workflows/.github/workflows/tf-lint.yaml@main
    with:
      tflint_config_path: '.tflint.hcl'
      tflint_minimum_failure_severity: 'warning'

  notify-status:
    needs: [terraform-validation]
    if: always()
    uses: duyhenryer/workflows/.github/workflows/status.yml@main
    with:
      slack_channel_id: "#infra-notifications"
    secrets: inherit
```

### Multi-Language Project

```yaml
name: Multi-Language CI

on:
  pull_request:
    branches: [main]

jobs:
  # Common checks
  common:
    uses: duyhenryer/workflows/.github/workflows/ci-common.yml@main
    secrets: inherit

  # Go checks
  go:
    if: contains(github.event.head_commit.modified, '*.go') || contains(github.event.head_commit.modified, 'go.mod')
    uses: duyhenryer/workflows/.github/workflows/go-check.yml@main
    with:
      command-test: 'go test ./...'
      lint: true
    secrets: inherit

  # Terraform checks
  terraform:
    if: contains(github.event.head_commit.modified, '*.tf')
    uses: duyhenryer/workflows/.github/workflows/tf-lint.yaml@main
    with:
      tflint_minimum_failure_severity: 'error'

  # Final notification
  notify:
    needs: [common, go, terraform]
    if: always()
    uses: duyhenryer/workflows/.github/workflows/status.yml@main
    with:
      slack_channel_id: "#ci-alerts"
    secrets: inherit
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
