# NeonDiff Review Action

NeonDiff reviews your pull requests with your own OpenAI key and posts the review on the PR. Reviews of public repositories are free and need no license key. Private repositories need a flat [Solo or Team license](https://neondiff.com/#pricing).

## Quickstart

1. Add your OpenAI API key as a repository or organization secret named `OPENAI_API_KEY`.
2. Save this as `.github/workflows/neondiff.yml`:

```yaml
name: NeonDiff
on:
  pull_request_target:
    types: [opened, ready_for_review]
  issue_comment:
    types: [created]

permissions:
  contents: read
  pull-requests: write
  issues: write

jobs:
  review:
    if: >-
      (github.event_name == 'pull_request_target' && !github.event.pull_request.draft) ||
      (github.event_name == 'issue_comment' &&
       github.event.issue.pull_request &&
       startsWith(github.event.comment.body, '/neondiff review') &&
       contains(fromJSON('["OWNER","MEMBER","COLLABORATOR"]'), github.event.comment.author_association))
    # One review per pull request at a time: a second trigger waits, then skips a head already reviewed.
    # Job-level, so ordinary PR comments never join the group or cancel a running review.
    concurrency:
      group: neondiff-review-${{ github.event.pull_request.number || github.event.issue.number }}
      cancel-in-progress: false
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - uses: electricsheephq/neondiff-action@v1
        with:
          openai-api-key: ${{ secrets.OPENAI_API_KEY }}
```

NeonDiff reviews each pull request when it opens (drafts are skipped) or is marked ready for review. Later pushes are not reviewed automatically: a maintainer comments `/neondiff review` to review the current head. On a public repository, anyone who opens a pull request triggers one review on your key.

## What it does and does not do

- Your workflow needs no checkout step and runs no code from the pull request: NeonDiff fetches the PR head into its own checkout. Codex may read it with read-only commands and is told never to build or run it; its sandbox blocks writes, and reviews of outside contributors get no shell at all.
- Codex runs read-only. With Codex 0.163.0 or later, its file reads are limited to the PR checkout, its git data, the Codex install and system directories. On Linux runners the action allows the user namespaces the Codex sandbox needs (`kernel.apparmor_restrict_unprivileged_userns=0`). GitHub-hosted runners are discarded after the job; on a self-hosted runner the setting stays until reboot. Where it can't be set, the review works from the diff alone.
- On public repositories, pull requests from outside contributors (anyone who is not an owner, member or collaborator) get a restricted, diff-only review: no shell, skills, hooks, web search or image tools, and no project docs. If that cannot be guaranteed, the review is skipped instead of running unrestricted, and the job log says why. In NeonDiff 1.1.0 a run that posts no review fails the job, for example such a skip or a head that moved before the job started; [#1218](https://github.com/electricsheephq/evaos-code-review-bot-neondiff/issues/1218) makes those skips pass.
- It posts one review per head with the built-in `GITHUB_TOKEN`. No GitHub App or config file is needed.
- Your code goes to OpenAI under your API key and your OpenAI data terms. Nothing goes to Electric Sheep, except the license check that private repositories need.
- Model cost grows with the size of the change and with how much surrounding code Codex reads. In an internal run on 25 ReviewBench pull requests, a completed review used about 90k tokens on average.

## Inputs

| Input | Required | Default | Notes |
|---|---|---|---|
| `openai-api-key` | yes | | Use a secret. |
| `github-token` | no | `github.token` | Needs `pull-requests: write` and `issues: write`. |
| `license-key` | no | | Needed only for private and internal repositories; a repository whose visibility GitHub doesn't report is not reviewed. Use a secret. In CI a license activates once for the repository owner: Team for an organization, Solo for your own account. |
| `model` | no | `gpt-6.1-sol` | |
| `reasoning-effort` | no | `high` | `low`, `medium`, `high`, `xhigh` or `max` |
| `dry-run` | no | `false` | `true` runs the review without posting it. |
| `neondiff-version` | no | `1.1.0` | npm version of `neondiff` |
| `codex-version` | no | `0.163.0` | npm version of `@openai/codex` |

## Private repositories

Add your license key as a secret and pass it to the action:

```yaml
      - uses: electricsheephq/neondiff-action@v1
        with:
          openai-api-key: ${{ secrets.OPENAI_API_KEY }}
          license-key: ${{ secrets.NEONDIFF_LICENSE_KEY }}
```

The action writes the key to a file only the runner can read and never passes it on the command line.

## License

This action's wrapper code is MIT licensed. It installs [NeonDiff](https://github.com/electricsheephq/evaos-code-review-bot-neondiff), which is source-available under its own license.
