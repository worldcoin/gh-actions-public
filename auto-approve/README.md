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
  default describes a change a reviewer can read in one pass, including a verified dependency bump.
- `high-risk-conditions` (optional): conditions that make a change high risk. The agent must not
  approve when any of them holds. The caller's standards for its own repository belong here; the
  default stops on large API surfaces, exported-API changes, oversized diffs, CI/workflow/release
  configuration, unverified dependency changes and external contributions.
- `prefilter` (optional, default `true`): screen the pull request with a cheap decision model before
  running the review. Set to `false` to always run the review.
- `prefilter-model` (optional, default `typesafe/jev-1.13`): model used to screen.
- `prefilter-threshold` (optional, default `0.5`): skip the review when the model scores the
  skip-review question at or above this value.

Dependency bumps are not blocked outright. The prompt tells the agent to check what actually changed
upstream between the old and new version — release notes, changelog and the diff where available —
and to refuse when the version does not exist upstream, does not match the pinned commit or integrity
hash, changes a source or registry URL, adds or changes an install/build/postinstall hook, pulls in
unexpected transitive dependencies, or changes maintainership.

Before the agent runs, a decision model scores the diff against a coarse "should this skip the
detailed review?" question and can skip it, which saves the install and the agent run on pull
requests that would not be approved anyway. It can only skip a review, never approve one: an error, a
missing answer, an empty diff, or an answer below the threshold all still go to the agent, and so do
dependency bumps, which it cannot check against upstream.

## Consumer setup

- The calling workflow must use `pull_request_target` and grant `contents: read` and
  `pull-requests: read`. The action reads the pull request from the event payload, so it only does
  anything useful for pull request events.
- Gate the job on the base branch you intend to approve (`if: github.event.pull_request.base.ref ==
  'main'`). `pull_request_target` loads the workflow file from the pull request's base branch, so any
  other base runs that branch's copy of it with your secrets in scope. The action's own `base-branch`
  guard runs far too late to prevent that.
- The bot account needs write access to the repository for its approval to count. Add it to
  `CODEOWNERS` if you also want it requested as a reviewer.
- **Required: the repository must dismiss stale approvals when new commits are pushed**
  (`dismiss_stale_reviews`, which is the default in `worldcoin/terraform-github-modules//
  app-repository`), and it should also require the approval to come from someone other than the last
  pusher (`require_last_push_approval`). The action binds its approval to the head it reviewed, but
  GitHub carries an approval across commits when stale approvals are not dismissed: without this, a
  push landing after the review leaves the new head satisfying the approval requirement without ever
  having been reviewed. The action cannot check or enforce this, so verify it before adopting.
- The guards only approve pull requests against `base-branch` whose head is a branch of the same
  repository, that are not drafts, whose author is not the bot and is a member/collaborator, whose
  head has not moved since the verdict, and that do not touch `.github/`.
- The action approves only. Branch protection still has to be the thing that gates merging.

## License

See the [LICENSE.md](../../LICENSE.md) file for license information.
