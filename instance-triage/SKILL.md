---
name: instance-triage
description: >
  Process an instance inventory report to determine which stopped isx
  instances are safe to delete, which have open work, and which need
  manual review.
---

# Instance Inventory Triage

Use when processing output from `instance-inventory.sh` (a host tool or
manual run). The goal is to classify each instance as safe to delete,
needing to be kept, or unclear.

## Triage procedure

For each deeply inspected instance:

1. **Check every PR reference** listed under "PR references in sessions".
   Use `gh pr view <url> --json state,title,mergedAt,url` on each.
   A merged PR whose title or branch matches the instance's work means
   the instance is safe to delete.

2. **Check the git repo state.** For each repo in the instance:
   - Branch on `main` with no changes: exploratory, safe to delete.
   - Branch not on any remote with no uncommitted changes: search for
     a matching merged or open PR by branch name and commit message.
   - Uncommitted changes or unpushed commits: keep.

3. **Search upstream repos** using the branch name, commit message
   keywords, and any fork remotes. Check all forks listed in remotes,
   not just the agent account:
   ```bash
   gh pr list --repo <upstream> --search "<branch-name>" --state all --json url,state,title
   gh pr list --repo <upstream> --head <fork-owner>:<branch> --state all --json url,state,title
   ```

4. **Account for rebased work.** The user may have taken the agent's
   work and rebased it into their own PR. Commit hashes will differ.
   Match on branch name keywords, commit message, or issue numbers
   in the instance name (e.g. instance `quarkus-56834-*` relates to
   issue #56834).

## Verdicts

- **SAFE TO DELETE**: all work merged upstream, no uncommitted changes.
- **KEEP**: open PRs or uncommitted/unpushed work.
- **UNCLEAR**: no matching PR found; may be unsubmitted or merged under
  a different name. Flag for manual review.
- **NOT INSPECTED**: template not in the allowed list; cannot determine.

## Output format

Produce a markdown table with columns: Instance, Verdict, PR (with full
URL), and a one-line Reason. Always include full PR URLs so the user can
verify each conclusion.

For non-inspected instances, list them at the bottom as NOT INSPECTED.
