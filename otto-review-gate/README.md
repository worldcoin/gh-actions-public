# Otto Review Gate GitHub Action

This directory contains the **Otto Review Gate** composite action, used to enforce a security control on pull requests authored by Otto (`agentotto`).

## What it does

When a pull request is created by `agentotto`, this action:

1. Reads the `<!-- otto-requester: @username -->` tag that Otto embeds in every PR body to identify who originally requested the PR.
2. Checks the list of PR approvers.
3. Fails the `otto/review-gate` commit status if the requester is the **only** approver — i.e., if no independent reviewer has approved.
4. Passes automatically for any PR not authored by `agentotto`.

This prevents the bypass where a developer asks Otto to create a PR containing their own code, then approves it themselves to satisfy the repository's approval requirement.

## Inputs

- `pr_number` (**required**): The pull request number to check.
- `head_sha` (**required**): The head SHA of the pull request.

## Permissions required

The calling workflow job must grant:

```yaml
permissions:
  statuses: write      # to post the otto/review-gate commit status
  pull-requests: read  # to read PR reviews and body
```

## Usage Example

See the [`example-workflow.yaml`](./example-workflow.yaml) file in this directory for a complete usage example.

## License

See the [LICENSE.md](../../LICENSE.md) file for license information.