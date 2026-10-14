---
name: git-rebase-workflow
description: >
  Fixup commit discipline and incremental verification for complex,
  long-running tasks where testing is expensive and changes span many
  modules. Builds on git-rebase primitives.
---

# Git Rebase Workflow

Use for complex commit series with expensive tests. For mechanics, read
[git-rebase](../git-rebase/SKILL.md). Respect the user's chosen commit workflow.

## Fixups

- Target the commit that introduced the code being fixed. Split fixes across
  target commits so each remains independently correct.
- Keep test assertions valid at their commit point; update later expectations
  in a fixup targeting the later behavior change.
- Prefer new fixup commits over amending during review, so corrections remain
  inspectable before squashing.

## Verify the Series

At each rebase `edit` stop, compile and run tests for affected modules,
fix failures, then `git rebase --continue`. Verify the squashed commits too.
After the human rewrites history, fetch and compare the new series with the
previous version to confirm fixes survived; investigate missing changes.

Do not bypass hooks with `--no-verify`. Check `git status` and stage specific
files. Force-push only to your fork unless explicitly authorized otherwise;
use `--force-with-lease`.
