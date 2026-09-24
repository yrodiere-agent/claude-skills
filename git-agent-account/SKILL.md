---
name: git-agent-account
description: >
  Working with git and GitHub as an agent using a dedicated account
  (e.g. "yrodiere-agent"). Covers forking, pushing, pull requests, and
  the constraints of not having push access to upstream repositories.
  Load before any `git push` or `gh pr create` when the OS user or
  GitHub account contains "agent".
---

# Git Agent Account

This skill is only relevant if you operate under a dedicated GitHub
account (e.g. `yrodiere-agent`) that is separate from the human's
account. If you share the human's GitHub credentials, this skill does
not apply.

Operating under a separate account has practical consequences for how
you interact with git and GitHub.

**Key rule:** never push to a remote or open a pull request unless
the human explicitly asks you to. Committing locally is fine — it is
the external-facing actions (push, PR creation, commenting on issues)
that require explicit instructions.

## Push Access

You can only push to repositories owned by your agent account. You
cannot push to the human's repositories or to upstream organizations.

This means:

- You work on a local clone, committing locally
- To push, you push to **your own fork**
- Pull requests go from your fork to the human's repo (or upstream)

## Forking

Before you can push, you need a fork under your agent account.

```bash
gh repo fork <owner>/<repo> --clone=false
```

- Your fork may already exist from a previous session. `gh repo fork`
  handles this gracefully — it will reuse the existing fork.
- The local clone may already have a remote pointing to your fork
  (typically named `origin` if you cloned your fork, or added manually).
  Check with `git remote -v` before adding remotes.

## Pushing

Push to your fork, not to the human's remote:

```bash
git push origin <branch>          # if origin points to your fork
git push <your-fork-remote> <branch>  # if you named it differently
```

Force-pushing to your own fork is fine when needed (e.g. after rebase).

## Pull Requests

**Never create a PR unless the human explicitly asks you to.** Pushing
a branch is not the same as opening a PR — wait for explicit
instructions before running `gh pr create`. When in doubt, push the
branch and tell the human it is ready; let them decide when to open
the PR.

When asked, use `gh pr create` to open PRs from your fork to the
target repo:

```bash
gh pr create \
  --repo <target-owner>/<repo> \
  --head <your-account>:<branch> \
  --title "..." \
  --body "..."
```

- `--repo` specifies where the PR is opened (the human's repo or upstream)
- `--head` must include your account prefix (e.g. `yrodiere-agent:my-branch`)
- Only target repos the human told you to target — never spontaneously
  open PRs against repos you were not asked to contribute to

### Keeping PRs in Sync

When you push new commits to a branch that backs an existing PR,
**always update the PR title and description to match the changes.**
A PR whose description doesn't reflect its commits is confusing for
reviewers and may lead to wrong merge decisions.

```bash
gh pr edit <number> --repo <target-owner>/<repo> \
  --title "New title" \
  --body "$(cat <<'BODY'
Updated description.
BODY
)"
```

This applies after any push — rebases, fixups, added commits, or
force-pushes that rewrite history. The PR description is the
reviewer's entry point; stale descriptions waste their time.

## GitHub API Rate Limits

The GitHub API token is shared with other processes on the same
machine. Be conservative with API calls to avoid exhausting the rate
limit for everyone.

- **`gh run watch` polls every 3 seconds by default** — that burns
  through hundreds of requests in minutes. Always pass a longer
  interval: `gh run watch <id> --interval 30` (or higher). Even
  better, run a single delayed check in the background:
  `sleep 300 && gh pr checks ...`.
- Avoid tight polling loops. One check after a reasonable delay
  (5–10 min for unit tests, longer for integration) is usually enough.
- Before retrying after a rate-limit error, check
  `gh api rate_limit --jq '.resources.core'` and wait until the reset
  time.

## Responding to PR Reviews

Check for reviews with:

```bash
gh api repos/<owner>/<repo>/pulls/<number>/comments \
  --jq '.[] | "LINE \(.line) PATH \(.path):\n\(.body)\n---"'
```

Then push fixes to the same branch on your fork — the PR updates
automatically.

## Working on the Human's Local Clone

When the local clone has `origin` pointing to the human's repo (not yours),
you cannot push to `origin`. Options:

- Add your fork as a remote and push there
- OR Commit locally and let the human push
- OR Create fixup commits that the human will squash and push
