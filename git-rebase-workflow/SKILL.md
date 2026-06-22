---
name: git-rebase-workflow
description: >
  Patterns for interactive rebasing, fixup commits, and incremental
  verification of multi-commit branches. For complex, long-running tasks
  where testing is expensive and changes have side-effects across many
  modules.
---

# Git Rebase Workflow

This skill applies to complex, long-running tasks where:

1. Testing is expensive (e.g. large projects with long build times)
2. The feature is complex and changes may have side-effects in many places

This leads to back-and-forth to fix older commits that turn out to
contain bugs — hence the structured workflow below.

For simpler tasks, follow these patterns loosely. The human may also ask
you to create regular (non-fixup) commits, or take the lead on squashing
and rewriting only for the most complex cases.

## Interactive Rebase

Use `GIT_SEQUENCE_EDITOR` to script rebase operations — never use
`git rebase -i` interactively (it requires a terminal editor).

```bash
# Mark all commits as 'edit' to stop at each one:
GIT_SEQUENCE_EDITOR="sed -i 's/^pick /edit /'" git rebase -i <base>

# Autosquash fixup commits:
GIT_SEQUENCE_EDITOR="cat" git rebase -i --autosquash <base>  # preview
git rebase -i --autosquash <base>                              # execute

# Stop at a specific commit during rebase:
GIT_SEQUENCE_EDITOR="sed -i 's/pick <hash>/edit <hash>/'" git rebase -i <base>
```

At each stop: build, test, fix, then `git rebase --continue`.

## Fixup Commits

Fixup commits let the human inspect exactly what you changed. Each fixup
must target the right parent commit.

```bash
git commit -m "$(cat <<'EOF'
fixup! <exact subject line of the target commit>

<description of what this fixup does>
EOF
)"
```

### Targeting Rules

Match the fixup to the commit that introduced the code being fixed.

For example, in a branch with separate commits for datasource
infrastructure, Hibernate refactoring, and injection scanning:

- A fix to datasource code → fixup for the datasource commit
- A fix to Hibernate code → fixup for the Hibernate commit
- A fix that touches both → split into two fixup commits

A single logical fix that touches files from different commits must be
split into multiple fixup commits, one per target.

### Assertions Must Match the Commit Point

When a test is introduced at commit N but its assertions depend on error
messages from commit N+2, the test must use old-style assertions at
commit N and a separate fixup for commit N+2 updates the assertions.

To verify: check out the squashed commit, build, and run the test.

## Build and Test at Each Commit

During an interactive rebase, at each `edit` stop:

1. Quick build to verify compilation
2. Amend the commit message if needed
3. Run relevant tests — not just the module you changed, but all modules
   affected by the change
4. If tests fail, fix and create a fixup commit (not amend — amending
   makes it hard for the human to review what you changed compared to
   the original commit)
5. `git rebase --continue`

## When the Human Rewrites History

After the human rebases or rewrites:

```bash
git fetch origin
git log --oneline origin/main..HEAD   # see the new commit structure
```

Check what changed vs your previous work. If you notice something that
looks wrong (a fix that disappeared, a commit that seems incomplete),
double-check your fixes survived and ask the human what happened.

## Recovery

Use `git reflog` to find lost commits:

```bash
git reflog | grep "fixup\|amend"
git show <hash> --stat   # inspect
git cherry-pick <hash>   # recover
```

Tag important states so the human can fetch them from your fork:

```bash
git tag fixup-commit3-v1 <hash>
```

## Temporary Branches for Testing

To test at a specific commit without disrupting the branch:

```bash
git checkout <hash>          # detached HEAD
# build and test
git checkout main            # return
```

Or create a temporary branch:

```bash
git checkout -b temp-test <hash>
# build and test
git checkout main && git branch -D temp-test
```

## Handling Rebase Conflicts

When `git rebase --continue` hits a conflict:

1. Check if the conflict is from a fixup that was superseded by a newer
   fixup — if so, `git rebase --skip`
2. For real conflicts, resolve manually, understanding which version
   (ours vs theirs) has the right code at this point in the commit series
3. If a commit becomes empty after conflict resolution, either skip it
   or investigate why

## Conventions

- **Never use `--no-verify`** or skip hooks
- Force-pushing to your own fork is fine; never force-push to the
  human's branch unless explicitly asked
- **Prefer new commits over amending** — amending makes it hard for the
  human to review what you changed compared to the original commit
- **Use `git add <specific files>`** not `git add -A` — leftover
  untracked files from earlier operations can sneak in
- **Always check `git status`** before committing to catch unintended files
