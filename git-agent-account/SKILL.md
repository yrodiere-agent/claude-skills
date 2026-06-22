---
name: git-agent-account
description: >
  Working with git and GitHub as an agent using a dedicated account
  (e.g. "yrodiere-agent"). Covers forking, pushing, pull requests, and
  the constraints of not having push access to upstream repositories.
---

# Git Agent Account

You operate under a dedicated GitHub account (e.g. `yrodiere-agent`)
that is separate from the human's account. This has practical
consequences for how you interact with git and GitHub.

## Push Access

You can only push to repositories owned by your agent account. You
cannot push to the human's repositories or to upstream organizations.

This means:

- You work on the human's local clone, committing locally
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

Use `gh pr create` to open PRs from your fork to the target repo:

```bash
gh pr create \
  --repo <target-owner>/<repo> \
  --head <your-account>:<branch> \
  --title "..." \
  --body "..."
```

- `--repo` specifies where the PR is opened (the human's repo or upstream)
- `--head` must include your account prefix (e.g. `yrodiere-agent:my-branch`)

## Responding to PR Reviews

Check for reviews with:

```bash
gh api repos/<owner>/<repo>/pulls/<number>/comments \
  --jq '.[] | "LINE \(.line) PATH \(.path):\n\(.body)\n---"'
```

Then push fixes to the same branch on your fork — the PR updates
automatically.

## Common Patterns

### Contributing to a repo you don't own

```bash
# Fork (idempotent — reuses existing fork)
gh repo fork <owner>/<repo> --clone=true
cd <repo>

# Create branch, make changes, push
git checkout -b my-feature
# ... make changes ...
git push -u origin my-feature

# Open PR
gh pr create --repo <owner>/<repo> \
  --head <your-account>:my-feature \
  --title "My feature" --body "..."
```

### Working on the human's local clone

When the human's clone has `origin` pointing to their repo (not yours),
you cannot push to `origin`. Options:

1. Add your fork as a remote and push there
2. Commit locally and let the human push
3. Create fixup commits that the human will squash and push
