# Composite actions

Most callers use the public reusable workflows. Two composite actions remain
for behavior that runs inside an existing job.

| Action | Purpose |
|--------|---------|
| `sync-branches` | Open or update a branch synchronization PR |
| `slack-notification` | Send the optional status notification used by `status.yml` |

## `sync-branches`

The action compares a source branch with a target branch, updates a bot-owned
`sync/<source>-to-<target>` branch with `--force-with-lease`, and creates or
reuses one pull request.

The standard use is a `main → dev` PR after promotion or a production hotfix.
See the [action README](../.github/actions/sync-branches/README.md) and the
[complete caller](../examples/sync-main-to-dev.yml).

Use a GitHub App installation token or fine-grained PAT when the generated PR
must start required checks automatically. A PR created with `GITHUB_TOKEN` may
require a maintainer to approve its workflow runs.

Inputs are passed through environment variables and validated before use in
shell commands. Requested auto-merge is fail-closed: inability to enable it
fails the action.

## `slack-notification`

This action remains an implementation detail of `status.yml`.

| Input | Required | Description |
|-------|----------|-------------|
| `channel_id` | Yes | Slack channel ID |
| `status` | Yes | `success`, `failed`, or `cancelled` |
| `slack_bot_token` | Yes | Slack bot token |
| `thread_ts` | No | Existing thread timestamp |
| `update_ts` | No | Message timestamp to update |
| `reply_broadcast` | No | Broadcast a threaded reply |

When referenced directly, pin the action to an immutable commit SHA rather
than `main` or a floating major tag.
