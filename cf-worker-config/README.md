# cf-worker-config

Composite action for reconciling Cloudflare Workers Builds configuration with
[`worldcoin/cf-worker-config`](https://github.com/worldcoin/cf-worker-config).

The caller repository only needs a `cf-worker-config.json` file and a workflow
that calls this action. The action checks out and builds the tool internally, so
application repositories do not need to run `setup-go` or install the CLI
themselves.

## Example usage

```yaml
name: cf-worker-config

on:
  pull_request:
  push:
    branches:
      - main
      - prod
  workflow_dispatch:
    inputs:
      app_ref:
        description: Optional application ref to reconcile
        required: false
        default: ""

jobs:
  tfh-website-dev:
    runs-on:
      group: arc-public-large-amd64-runner
    permissions:
      contents: read
      issues: write
      pull-requests: write
    steps:
      - uses: worldcoin/gh-actions-public/cf-worker-config@main
        with:
          worker: tfh-website-dev
          config: cf-worker-config.json
          mode: plan-and-apply
          app_ref: ${{ github.event.inputs.app_ref || '' }}
          cf_worker_config_ref: main
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
```

Replace `@main` with a pinned `worldcoin/gh-actions-public` commit SHA in repositories
that enforce immutable GitHub Actions references.

`mode: plan-and-apply` is the default. It runs `plan` first and then `apply`,
including on pull requests, so Workers Builds configuration can be updated before
Cloudflare builds previews. Use `mode: plan` only for dry-runs.

Runtime secrets are intentionally separate from the PR/push reconciliation flow.
Use `mode: sync-runtime-secrets` from a manual workflow after updating GitHub
environment or repository secrets:

```yaml
name: cf-worker-runtime-secrets

on:
  workflow_dispatch:

jobs:
  tfh-website-dev-runtime-secrets:
    runs-on:
      group: arc-public-large-amd64-runner
    environment: dev
    permissions:
      contents: read
    steps:
      - uses: worldcoin/gh-actions-public/cf-worker-config@main
        with:
          worker: tfh-website-dev
          config: cf-worker-config.json
          mode: sync-runtime-secrets
          cf_worker_config_ref: main
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          NEXT_SERVER_HUBSPOT_API_TOKEN: ${{ secrets.NEXT_SERVER_HUBSPOT_API_TOKEN }}
          NEXT_SERVER_ASHBY_API_KEY: ${{ secrets.NEXT_SERVER_ASHBY_API_KEY }}
          NEXT_SERVER_IRONCLAD_API_KEY: ${{ secrets.NEXT_SERVER_IRONCLAD_API_KEY }}
          NEXT_SERVER_IP_HUB_API_KEY: ${{ secrets.NEXT_SERVER_IP_HUB_API_KEY }}
          NEXT_PUBLIC_EMAIL_ENCRYPTION_KEY: ${{ secrets.NEXT_PUBLIC_EMAIL_ENCRYPTION_KEY }}
```

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `worker` | yes | | Worker key from `cf-worker-config.json`. |
| `config` | no | `cf-worker-config.json` | Relative path to the config file in the caller repository. |
| `mode` | no | `plan-and-apply` | One of `plan`, `plan-and-apply`, or `sync-runtime-secrets`. |
| `cf_worker_config_ref` | no | `main` | Branch, tag, or commit SHA from `worldcoin/cf-worker-config` to build. |
| `github_token` | no | | Optional token for GitHub repository metadata lookups. Defaults to `github.token`, which is sufficient for the public tool repository and the caller repository. |
| `app_ref` | no | | Optional branch, tag, or commit SHA from the caller repository to check out. |

If `config` is not present in the caller repository, the action exits
successfully as a no-op.

## Environment

| Environment variable | Required | Description |
| --- | --- | --- |
| `CLOUDFLARE_API_TOKEN` | yes, when config exists | User-scoped Cloudflare token with Workers Builds Configuration edit access and Workers Scripts read/edit access. |

`mode: plan-and-apply` reads build secret values from environment variables when
they are available. It does not create or refresh runtime secrets.

`mode: sync-runtime-secrets` reads runtime secret values from environment
variables. The caller workflow must map each required secret explicitly in
`env`; this action intentionally does not expose a generic `toJson(secrets)`
secret-passing layer.

## Pull request comments

On pull requests, the action creates or updates a sticky comment per worker.
The comment is created when reconciliation starts and is updated after `plan`
and `apply` with the command status and output. Failed `plan` or `apply`
commands are still added to the comment before the job fails.

PR comments are best-effort. The runner must have `gh` and `jq` available, and
the workflow `github.token` must be allowed to create issue comments, for
example with `issues: write` in the workflow permissions.
