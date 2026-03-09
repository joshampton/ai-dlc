---
description: Remove orphaned AI-DLC worktrees
disable-model-invocation: true
---

## Name

`ai-dlc:cleanup` - Remove orphaned AI-DLC worktrees.

## Synopsis

```
/cleanup
```

## Description

**User-facing command** - Run this to clean up stale worktrees left behind by interrupted sessions.

Scans `.ai-dlc/worktrees/` for worktree directories and removes any that are orphaned (the backing git worktree entry is stale or the directory is left over from a crashed session).

This does not:
- Clear AI-DLC state (use `/reset` for that)
- Delete branches or commits
- Affect active worktrees with running sessions

## Implementation

### Pre-check: Reject Cowork Mode

```bash
if [ "${CLAUDE_CODE_IS_COWORK:-}" = "1" ]; then
  echo "ERROR: /cleanup cannot run in cowork mode."
  echo "Run this in a full Claude Code CLI session."
  exit 1
fi
```

If `CLAUDE_CODE_IS_COWORK=1`, stop immediately with the message above. Do NOT proceed.

### Step 1: Discover Worktrees

```bash
REPO_ROOT=$(git worktree list --porcelain | head -1 | sed 's/^worktree //')
WORKTREES_DIR="${REPO_ROOT}/.ai-dlc/worktrees"

if [ ! -d "$WORKTREES_DIR" ]; then
  echo "No .ai-dlc/worktrees/ directory found. Nothing to clean up."
  exit 0
fi

# List all directories in .ai-dlc/worktrees/
DIRS=$(find "$WORKTREES_DIR" -mindepth 1 -maxdepth 1 -type d 2>/dev/null)
if [ -z "$DIRS" ]; then
  echo "No worktree directories found. Nothing to clean up."
  exit 0
fi
```

### Step 2: Identify Orphaned Worktrees

```bash
# Get list of valid worktree paths from git
VALID_WORKTREES=$(git worktree list --porcelain | grep '^worktree ' | sed 's/^worktree //')

ORPHANED=()
ACTIVE=()

for dir in $DIRS; do
  if echo "$VALID_WORKTREES" | grep -qF "$dir"; then
    ACTIVE+=("$(basename "$dir")")
  else
    ORPHANED+=("$(basename "$dir")")
  fi
done
```

### Step 3: Report and Confirm

Show the user what was found:

```
## AI-DLC Worktree Cleanup

**Active worktrees:** {count}
{list of active worktree names, if any}

**Orphaned worktrees:** {count}
{list of orphaned worktree names, if any}
```

If there are orphaned worktrees, ask the user to confirm removal using `AskUserQuestion`.

If there are no orphaned entries but there are active worktrees, ask whether to force-remove all worktrees (with a warning that this will interrupt any running sessions).

If there is nothing to clean up, output:

```
No orphaned worktrees found. Everything is clean.
```

### Step 4: Remove

```bash
REPO_ROOT=$(git worktree list --porcelain | head -1 | sed 's/^worktree //')

# Remove orphaned directories (no valid git worktree entry)
for name in "${ORPHANED[@]}"; do
  rm -rf "${REPO_ROOT}/.ai-dlc/worktrees/${name}"
done

# If user chose to force-remove active worktrees too:
for name in "${FORCE_REMOVE[@]}"; do
  git worktree remove --force "${REPO_ROOT}/.ai-dlc/worktrees/${name}" 2>/dev/null
done

# Prune stale git worktree metadata
git worktree prune
```

### Step 5: Confirm

Output:
```
Cleanup complete.

Removed {count} orphaned worktree(s).
```
