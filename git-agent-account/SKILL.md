---
name: git-agent-account
description: >
  Working with git and GitHub as an agent using a dedicated account
  (e.g. "yrodiere-agent"). Covers forking, pushing, pull requests, and
  the constraints of not having push access to upstream repositories,
  and checking that work is based on the latest target branch.
  Load before starting repository work, and before any `git push` or
  `gh pr create`, when the OS user or GitHub account contains "agent".
---

# Git Agent Account

Use with a dedicated agent account, not the human's credentials.
**Push, PR creation, and issue comments require explicit user instructions.**
Push only to the agent's fork; target only the requested repository.

## Check the Target Branch

Before starting work and before opening or updating a PR:

```bash
git fetch <target-remote> <target-branch>
git merge-base --is-ancestor FETCH_HEAD HEAD
```

Fetch from the target repository, not the fork. If behind, rebase and rerun
relevant checks. Respect user constraints; report failed fetches.

## Publish

Check remote URLs before choosing where to push. Create or reuse the fork:

```bash
gh repo fork <target-owner>/<repo> --clone=false
git push <fork-remote> <branch>
gh pr create --repo <target-owner>/<repo> --head <agent-account>:<branch> \
  --title "..." --body-file /tmp/pr-body.md
```

Use `--force-with-lease` for rebased branches. After every PR update,
keep its title and description aligned with the final changes:

```bash
gh pr edit <number> --repo <target-owner>/<repo> \
  --title "..." --body-file /tmp/pr-body.md
```

## CI and Reviews

The GitHub token is shared. Avoid tight polling; use `gh run watch --interval 30`
or check once after a suitable delay. On rate-limit errors, check
`gh api rate_limit --jq '.resources.core'` and wait for reset.

```bash
gh api repos/<owner>/<repo>/pulls/<number>/comments \
  --jq '.[] | {path, line, body}'
```

Push review fixes to the existing PR branch.
