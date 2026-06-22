---
name: git-rebase-workflow
description: >
  Patterns for interactive rebasing, fixup commits, and incremental
  verification of multi-commit branches. Covers commit organization,
  testing at each commit point, and coordinating with a human who
  rewrites history.
---

# Git Rebase Workflow

## Core Principle

The human owns the git history. You produce fixup commits; the human
squashes, reorders, and rewrites. Never force-push, never amend commits
the human wrote, and never squash fixups yourself unless explicitly told to.

## Interactive Rebase

Use `GIT_SEQUENCE_EDITOR` to script rebase operations — never use
`git rebase -i` interactively (it requires a terminal editor).

```bash
# Mark all commits as 'edit' to stop at each one:
GIT_SEQUENCE_EDITOR="sed -i 's/^pick /edit /'" git rebase -i <base>

# Autosquash fixup commits:
GIT_SEQUENCE_EDITOR="cat" git rebase -i --autosquash <base>  # preview
git rebase -i --autosquash <base>                              # execute (non-interactive)

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

Match the fixup to the commit that introduced the code being fixed:

- Agroal/datasource changes → fixup for the datasource commit
- Hibernate ORM/Reactive changes → fixup for the Hibernate commit
- Injection scanning changes → fixup for the injection commit
- Test assertion updates → fixup for whichever commit changed the error messages

A single logical fix that touches files from different commits must be
split into multiple fixup commits, one per target.

### Assertions Must Match the Commit Point

When a test is introduced at commit N but its assertions depend on error
messages from commit N+2, the test must use old-style assertions at
commit N and a separate fixup for commit N+2 updates the assertions.

To verify: check out the squashed commit, build, and run the test.

## Build and Test at Each Commit

During an interactive rebase, at each `edit` stop:

1. `./mvnw -Dquickly` — verify compilation (ignore pre-existing failures
   like Gradle plugin issues)
2. Amend the commit message if needed
3. Run relevant extension tests:
   ```bash
   ./mvnw verify -f extensions/<name>/ -Dtest-containers -Dstart-containers
   ```
4. If tests fail, fix and create a fixup commit (not amend — the human
   wants to see what you changed)
5. `git rebase --continue`

Run ALL relevant tests, not just the extension you changed. A datasource
change affects Agroal, Flyway, Liquibase, Hibernate ORM, Hibernate
Reactive, Hibernate Envers, and Hibernate Search.

## When the Human Rewrites History

After the human rebases or rewrites:

```bash
git fetch origin
git log --oneline origin/main..HEAD   # see the new commit structure
```

Check what changed vs your previous work:
- Are your fixups squashed?
- Were commits reordered or split?
- Did the human add new commits?

Do not assume your fixes survived — verify with `grep` for key changes.

## Recovery

Use `git reflog` to find lost commits:

```bash
git reflog | grep "fixup\|amend"
git show <hash> --stat   # inspect
git cherry-pick <hash>   # recover
```

Tag important states so the human can fetch them:

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

## Build Step Cycles

When introducing build steps that consume `BeanDiscoveryFinishedBuildItem`
(or similar late-phase items), anything that transitively produces
`AdditionalBeanBuildItem`, `AnnotationsTransformerBuildItem`, or other
items feeding into Arc's early phases creates a cycle. The fix is always
to move the Arc-feeding production to a step that does not depend on
the late-phase item:

- Move `AdditionalBeanBuildItem` to an early step without the dependency
- Convert beans from `AdditionalBeanBuildItem` to `SyntheticBeanBuildItem`
  (which feeds into a later Arc phase)
- Extract specific build item production into a separate step

The cycle detector is static — it checks the step signature, not runtime
behavior. Even if a step only conditionally produces the problematic item,
the cycle is detected.

## Conventions

- **Never use `--no-verify`** or skip hooks
- **Never force-push** unless the human explicitly asks
- **Prefer new commits over amending** — amending can lose the human's
  work if a pre-commit hook fails
- **Use `git add <specific files>`** not `git add -A` — leftover
  untracked files from earlier operations can sneak in
- **Always check `git status`** before committing to catch unintended files
