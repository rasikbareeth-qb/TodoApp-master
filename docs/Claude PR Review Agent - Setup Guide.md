# Claude PR Review Remote Agent – Setup Guide

Nov 2, 2026 · @Rasik Bareeth

## Overview

Any GitHub repository can have Claude review every pull request automatically, using a review agent defined in the repo, and post the result as a PR comment.

1. A PR is opened, updated, reopened or marked ready for review. Drafts are skipped.
2. GitHub Actions starts the review workflow.
3. Claude reads your reviewer agent file and your review guidelines.
4. It reviews only the PR diff against the base branch.
5. It posts one comment with findings grouped by severity.

## Prerequisites

| Item | Why | Where |
| --- | --- | --- |
| GitHub repo with admin access | To add workflows and secrets | Repo **Settings** |
| Git push credentials with `workflow` scope | GitHub blocks pushing `.github/workflows/` files without it | github.com/settings/tokens |
| Claude Pro/Max subscription **or** Anthropic API key | Pays for the review runs | Claude Code CLI or console.anthropic.com |
| Claude Code CLI (subscription route only) | Generates the subscription token | `npm install -g @anthropic-ai/claude-code` |

## Setup steps

### 1. Add the reviewer agent and guidelines

Commit two files to the repo:

- `.claude/agents/<name>-code-reviewer.md` — the agent: read-only rules, scope, and output format. Add a **PR mode** section: review only the PR diff against its base branch, and post the whole review as a single comment via `gh pr comment`.
- `docs/<name>-code-review-guidelines.md` — the rulebook the agent reviews against (severity levels plus numbered rule sections).

### 2. Create a Claude credential

Pick one:

- **Subscription (no API billing):** run `claude setup-token`, sign in in the browser, and copy the printed token.
- **API key (pay per use):** console.anthropic.com → **Settings → API Keys → Create Key**. Make sure billing has credits.

### 3. Save it as a repository secret

Repo **Settings → Secrets and variables → Actions → New repository secret**:

| Credential | Secret name |
| --- | --- |
| Subscription token | `CLAUDE_CODE_OAUTH_TOKEN` |
| API key | `ANTHROPIC_API_KEY` |

Never paste the value into chat, code, or commits.

### 4. Add the workflow

Create `.github/workflows/claude-pr-review.yml`:

```yaml
name: Claude PR Review

on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]

concurrency:
  group: claude-pr-review-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  review:
    if: github.event.pull_request.draft == false
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
      id-token: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: anthropics/claude-code-action@v1
        with:
          # Use ONE of these two lines:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          # anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt: |
            Act as the reviewer in .claude/agents/<name>-code-reviewer.md, in PR mode.
            Read it and docs/<name>-code-review-guidelines.md first.
            Scope: `git diff origin/${{ github.event.pull_request.base.ref }}...HEAD`.
            Post the review with `gh pr comment ${{ github.event.pull_request.number }}`.
          claude_args: |
            --model claude-sonnet-5-5
            --allowedTools "Read,Grep,Glob,Bash(git diff:*),Bash(git log:*),Bash(git show:*),Bash(gh pr comment:*),Bash(gh pr view:*),Bash(gh pr diff:*)"
```

Replace `<name>` with your agent's file names.

### 5. Push with workflow scope

If the push is rejected with *refusing to allow a Personal Access Token to create or update workflow*:

- **Classic token:** tick **`workflow`** and click **Update token**.
- **Fine-grained token:** set **Contents** and **Workflows** to **Read and write**.
- Clear the cached login, then push again:

```bash
printf "host=github.com\nprotocol=https\n\n" | git credential-osxkeychain erase
```

### 6. Open a pull request

Open any non-draft PR. The **Actions** tab shows the run; the review appears as a PR comment within a few minutes.

## How it works

The same agent file serves both local and PR reviews; only the scope and output change.

| File | Role |
| --- | --- |
| `.claude/agents/<name>-code-reviewer.md` | Rules, scope, output format, PR mode |
| `docs/<name>-code-review-guidelines.md` | Rulebook: severity levels and numbered rule sections |
| `.github/workflows/claude-pr-review.yml` | Trigger, permissions, credential, prompt, allowed tools |

| Mode | Triggered by | Scope | Output |
| --- | --- | --- | --- |
| Local | Asking Claude Code to run the reviewer | 1. Uncommitted diff; 2. existing code issues | Two sections in the terminal |
| PR | Opening or updating a PR | PR diff vs base branch only | One PR comment |

The workflow can only read code and comment; its tool allow-list blocks edits, commits and pushes.

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| Push rejected: *without `workflow` scope* | Token lacks workflow permission | Add `workflow` scope (classic) or Workflows: Read and write (fine-grained); clear cached login |
| `Password authentication is not supported` | Account password used for git | Use a personal access token as the password |
| Workflow never runs | PR is a draft, or workflow not on the PR branch | Mark ready for review; confirm the file is pushed |
| Run fails at the Claude step | Secret missing or misnamed | Check the secret name matches the workflow input exactly |
| Run succeeds but no comment | `pull-requests: write` missing or `gh pr comment` not allowed | Check `permissions` and `--allowedTools` |
| API key run fails with billing error | No credits on the Console account | Add credits, or switch to the subscription token |
