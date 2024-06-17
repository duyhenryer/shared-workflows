# Reusable Workflows

The repository contains all general workflows which can be used for every workflow in another repository.

If you want to use a template workflow you can copy the following template and adapt it to your specific use case.
You can find all possible template workflows in the directory `.github/workflows`.

```yml
name: <your name>

on:
  ...

jobs:
  <action name>:
    uses: duyhenryer/workflows/.github/workflows/[action]*.yml@main
    with:
      ...
```

## Example Workflows 🔧
In this section you can find examples of how to use template workflows.

### [ci-common.yml](.github/workflows/ci-common.yml)
- As of now, this workflow is calling `pr-checks.yml`.
- Any new reusable workflows that need to be called by all service repos in the future, add them in this workflow file.

### [pr-checks.yml](.github/workflows/pr-checks.yml)
- This workflow will notifies PR events like created, updated, review_requested, etc on the respective slack channel by tagging the respective `CODEOWNERS`.

### [status.yml](.github/workflows/status.yml)
<details>
<summary>This workflow when invoked, notifies the calling workflow status on specifid slack channel.</summary>

- Accepted Inputs :
    - `slack_channel_id` : Slack channel name to be notified on.
    - `slack_msg` : The message to be sent to slack channel.
</details>

### [go-check.yml](.github/workflows/go-check.yml)
<details>
<summary>This workflow provides a robust way to check the quality of your Go code by running tests and linting.</summary>

- `test`: This job checks out the code, sets up Go (if specified by the setup-go input), and runs the unit tests. It's only run on pull requests.
- `lint`: This job checks out the code, sets up Go, downloads dependencies, and runs the linter. It's only run on pull requests if the lint input is true.
- `status`: This job runs the status.yml workflow. It's always run, regardless of the outcome of the other jobs.

```yml
name: Check

on:
    pull_request:
        branches:
            - main
            - dev
            - test
jobs:
    check:
        name: Check
        uses: duyhenryer/workflows/.github/workflows/go-check.yml@main
        with:
            command-test: 'echo bypass-unittest'
        secrets: inherit
```
</details>

### Notification
