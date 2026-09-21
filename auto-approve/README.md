# Auto Approve GitHub Action

This directory contains the **Auto Approve** GitHub Action. It reviews a pull request with a
[pi](https://www.npmjs.com/package/@earendil-works/pi-coding-agent) agent and approves it when the
agent records a low-risk verdict and the guards hold. If the agent does not record a verdict, or any
guard fails, the run reports why and does not approve.

## Usage Example

See the [`example-workflow.yaml`](./example-workflow.yaml) file in this directory for a complete
usage example.

The agent never holds the bot token: the review step gets the OpenRouter key and a read-only
GitHub token, and a later step, which has no model access, submits the approval. The workflow uses
`pull_request_target`, so the approving job always comes from the base branch and cannot be rewritten
by the pull request it judges.

## Inputs

- `openrouter-api-key` (**required**): OpenRouter API key for the review agent.
- `bot-token` (**required**): token of the bot account that comments and approves. Must not be
  `GITHUB_TOKEN`, which cannot approve pull requests in its own repository.
- `bot-login` (**required**): login of the bot account, used by the self-approval and author guards.
- `github-token` (optional, default `${{ github.token }}`): read token exposed to the review agent.
- `base-branch` (optional, default `main`): only pull requests targeting this branch are approved.
- `pi-version` (optional, default `0.85.1`): version of pi to install.
- `provider` (optional, default `openrouter`): model provider passed to pi.
- `model` (optional, default `deepseek/deepseek-v4.1-flash`): model passed to pi.
- `low-risk-criteria` (optional): fills the "A low-risk change is:" slot in the review prompt. The
  default describes a change a reviewer can read in one pass.
- `high-risk-conditions` (optional): conditions that make a change high risk. The agent must not
  approve when any of them holds. The caller's standards for its own repository belong here; the
  default stops on large API surfaces, exported-API changes, oversized diffs, CI/dependency
  configuration and external contributions.

## Consumer setup

- The calling workflow must use `pull_request_target` and grant `contents: read` and
  `pull-requests: read`. The action reads the pull request from the event payload, so it only does
  anything useful for pull request events.
- The bot account needs write access to the repository for its approval to count. Add it to
  `CODEOWNERS` if you also want it requested as a reviewer.
- The guards only approve pull requests against `base-branch` whose head is a branch of the same
  repository, that are not drafts, whose author is not the bot and is a member/collaborator, whose
  head has not moved since the verdict, and that do not touch `.github/`.
- The action approves only. Branch protection still has to be the thing that gates merging.

## License

See the [LICENSE.md](../../LICENSE.md) file for license information.
