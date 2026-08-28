# Documentation

Reference and guides for the reusable workflows in this repository.
Start at the [repository README](../README.md) for the workflow catalog and a quick start.

| Document | What's in it |
|----------|--------------|
| [workflows.md](workflows.md) | Public v2 contracts, permissions, inputs, and examples |
| [actions.md](actions.md) | Branch synchronization and notification composite actions |
| [architecture.md](architecture.md) | PR, dev artifact, release, and image-identity flows |
| [go-security.md](go-security.md) | Security contract: policy matrix, ruleset settings, rollout phases, exception process |
| [configuration.md](configuration.md) | Required secrets, security prerequisites, Slack channels, CODEOWNERS |
| [troubleshooting.md](troubleshooting.md) | Common failures and how to fix them |
| [../examples/](../examples/) | Copy-paste caller workflows for a Go service |

## Where things live

Each topic has exactly one home, so there is one place to update when something changes:

- **What a workflow accepts** → `workflows.md`
- **How workflows connect to each other** → `architecture.md`
- **What to configure in GitHub** → `configuration.md` (general) and `go-security.md` (rulesets)
- **What to copy into a service repo** → `../examples/`

## About this repository's own CI

`.github/workflows/actionlint.yml` is **not** a reusable workflow — it is this repository's
self-CI, linting every workflow file with actionlint (plus shellcheck on `run:` blocks) on each
pull request and push to `main`. It catches a broken reusable workflow before consumers pull
the default branch, so it is not documented as a reusable workflow.

Known gap: this repository does not yet scan its own workflows with CodeQL. Because there is no
`go.mod` here, it cannot call `go-security.yml` — a self-CI job would need to call `codeql.yml`
directly with `language: actions` and `build-mode: none`. See
[go-security.md — Known gaps](go-security.md#known-gaps).
