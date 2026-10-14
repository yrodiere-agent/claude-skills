---
name: git-rebase
description: >
  Non-interactive git rebase primitives: rewording commit messages,
  squashing/reordering commits, resolving conflicts, and recovery.
  Toolbox for any rebase task, not just complex workflows.
---

# Git Rebase Primitives

## GIT_SEQUENCE_EDITOR Basics

Use `GIT_SEQUENCE_EDITOR` to script the rebase todo list — never use
`git rebase -i` interactively (it requires a terminal editor).

```bash
# Mark all commits as 'edit' to stop at each one:
GIT_SEQUENCE_EDITOR="sed -i 's/^pick /edit /'" git rebase -i <base>

# Autosquash fixup commits:
GIT_SEQUENCE_EDITOR=true git rebase -i --autosquash <base>      # execute

# Stop at a specific commit during rebase:
GIT_SEQUENCE_EDITOR="sed -i 's/pick <hash>/edit <hash>/'" git rebase -i <base>
```

At each stop: build, test, fix, then `git rebase --continue`.

## Rewording a Commit Message

`GIT_SEQUENCE_EDITOR` controls the rebase todo list; `GIT_EDITOR`
controls the commit message editor invoked for `reword` stops. Both
must be set for a non-interactive reword to work.

Write the new message to a file, then use a shell script as editor:

```bash
cat > /tmp/new-msg.txt <<'EOF'
New commit subject

New commit body.
EOF

cat > /tmp/replace-msg.sh <<'SCRIPT'
#!/bin/bash
cp /tmp/new-msg.txt "$1"
SCRIPT
chmod +x /tmp/replace-msg.sh

GIT_SEQUENCE_EDITOR="sed -i '1s/^pick/reword/'" \
  GIT_EDITOR="/tmp/replace-msg.sh" \
  git rebase -i <commit>^
```

**Pitfall:** `EDITOR` does not work here — git rebase uses
`GIT_EDITOR` (or `core.editor`), not `EDITOR`. Using the wrong
variable silently keeps the old message.

## Squashing a Specific Commit

To squash commit B into an earlier commit A (when B is already
adjacent to A, i.e. directly after it):

```bash
GIT_SEQUENCE_EDITOR="sed -i 's/^pick <hash-of-B>/fixup <hash-of-B>/'" \
  git rebase -i <parent-of-A>
```

When B is *not* adjacent to A, reorder first:

```bash
# Move B right after A and fixup in one operation:
GIT_SEQUENCE_EDITOR='
  /^pick <hash-of-B>/{H;d}
  /^pick <hash-of-A>/{ p; g; s/^pick/fixup/; }
' git rebase -i <parent-of-A>
```

Or use two steps: first reorder with `GIT_SEQUENCE_EDITOR`, then
squash with `--autosquash` on a second pass.

## Fixup Commits

Create a fixup, then autosquash it:

```bash
git commit --fixup=<target-commit>
```

Then squash with autosquash:

```bash
GIT_SEQUENCE_EDITOR=true git rebase -i --autosquash <base>
```

## Handling Rebase Conflicts

When `git rebase --continue` hits a conflict:

1. Check if the conflict is from a fixup that was superseded by a newer
   fixup — if so, `git rebase --skip`
2. For real conflicts, resolve manually, understanding which version
   (ours vs theirs) has the right code at this point in the commit series
3. If a commit becomes empty after conflict resolution, either skip it
   or investigate why

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

## Test Earlier Commits

Use a separate worktree to avoid disturbing the active rebase or local changes:

```bash
git worktree add --detach /tmp/rebase-check <commit>
# Build and test in /tmp/rebase-check
git worktree remove /tmp/rebase-check
```
