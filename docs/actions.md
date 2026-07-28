# Composite Actions

The composite actions in `.github/actions/` are building blocks used internally by the reusable
workflows. You normally consume them indirectly — call the workflow, not the action. They are
documented here for the cases where you are building a custom workflow, or debugging one.

Reference them as `duynhlab/gha-workflows/.github/actions/<name>@main`. A relative path
(`./.github/actions/...`) does **not** work from another repository, because the checkout in a
reusable workflow is the caller's repository, not this one.

| Action | Used by | Purpose |
|--------|---------|---------|
| [`coverage-summary`](#coverage-summary) | `go-check.yml` | Go coverage job summary |
| [`gitleaks`](#gitleaks) | `gitleaks.yml` | Secret scanning via the gitleaks CLI |
| [`slack-notification`](#slack-notification) | `status.yml` | Slack CI status message |

---

## coverage-summary

**Location:** `.github/actions/coverage-summary/action.yml`
**Purpose:** Parse a Go coverage profile and write a job summary with total coverage and a
collapsible per-package breakdown.

### Inputs

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `coverage-file` | **Yes** | - | Path to the Go coverage profile (`.out` file) |
| `title` | **Yes** | - | Summary heading, e.g. `Unit Test Coverage` |
| `artifact-name` | **Yes** | - | Name of the uploaded coverage artifact |

### Outputs

| Output | Description |
|--------|-------------|
| `total-coverage` | Total statement coverage percentage (e.g. `72.5%`) |

### Usage

```yaml
- name: Coverage Report Summary
  uses: duynhlab/gha-workflows/.github/actions/coverage-summary@main
  with:
    coverage-file: coverage.out
    title: Unit Test Coverage
    artifact-name: coverage-report
```

---

## gitleaks

**Location:** `.github/actions/gitleaks/action.yml`
**Purpose:** Download the gitleaks CLI binary and scan source code / git history for leaked
secrets. Produces a SARIF report and a job summary. No license key required — the CLI is
MIT-licensed.

### Inputs

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `version` | No | `8.21.2` | Gitleaks CLI version to download (without the `v` prefix) |
| `config-path` | No | `""` | Path to a custom `.gitleaks.toml` |
| `scan-mode` | No | `auto` | `auto` (PR=diff, push=full), `full`, or `diff` |
| `fail-on-leak` | No | `true` | Fail the step when leaks are found |

### Outputs

| Output | Description |
|--------|-------------|
| `exit-code` | Gitleaks exit code (`0`=clean, `1`=leaks found) |
| `sarif` | Path to the SARIF report file |
| `summary` | Number of findings |

### Usage

The action does not check out the repository — do that first, with full history so `full` mode
can walk the git log.

```yaml
- name: Checkout
  uses: actions/checkout@v7
  with:
    fetch-depth: 0

- name: Run Gitleaks
  id: gitleaks
  uses: duynhlab/gha-workflows/.github/actions/gitleaks@main
  with:
    scan-mode: auto
```

> `fail-on-leak: false` is useful when you want the SARIF report uploaded but the decision made
> by a later step.

---

## slack-notification

**Location:** `.github/actions/slack-notification/action.yml`
**Purpose:** Send a CI/CD status notification to Slack, with thread support.

- `thread_ts` empty → sends a standalone message (push-to-main flow).
- `thread_ts` provided → replies in that thread (PR flow).
- `update_ts` provided → updates an existing message instead of sending a new one.

### Inputs

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `channel_id` | **Yes** | - | Slack channel ID |
| `status` | **Yes** | - | CI status: `success`, `failed`, `cancelled` |
| `thread_ts` | No | `""` | Slack thread timestamp to reply to |
| `update_ts` | No | `""` | Slack message timestamp to update |
| `reply_broadcast` | No | `false` | Whether the reply is broadcast to the channel |
| `slack_bot_token` | **Yes** | - | Slack bot token |

### Outputs

| Output | Description |
|--------|-------------|
| `ts` | Timestamp of the sent message |
| `thread_ts` | Thread timestamp, for chaining replies |

### Usage

```yaml
- name: Slack CI Notification
  uses: duynhlab/gha-workflows/.github/actions/slack-notification@main
  with:
    channel_id: "C0AD82A9A74"
    status: ${{ job.status }}
    slack_bot_token: ${{ secrets.SLACK_BOT_TOKEN }}
```

> Used internally by `status.yml`. You do not need to call it directly unless you are building a
> custom notification workflow.
