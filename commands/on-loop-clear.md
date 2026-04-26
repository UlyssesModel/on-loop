---
name: on-loop-clear
description: Clean up on-loop workspace, switch to main, and pull latest
user_invocable: true
argument: "[--include-logs]"
---

# /on-loop:clear

Clean up on-loop worktrees and optionally session logs, then return to a clean main branch state.

## Usage

```
/on-loop:clear                # Clean worktrees only, keep session logs
/on-loop:clear --include-logs # Also delete session logs
```

## What This Does

1. Removes all on-loop git worktrees
2. Optionally deletes `.on-loop/` session logs (with `--include-logs`)
3. Switches to `main` (or `master` if `main` doesn't exist)
4. Pulls the latest changes from origin

## Instructions

When this command is invoked:

1. **List and remove on-loop worktrees**:
   ```bash
   git worktree list
   ```
   - Identify any worktrees under `.claude/worktrees/`
   - For each on-loop worktree:
     ```bash
     git worktree remove .claude/worktrees/<slug> --force
     ```
   - Clean up any remnant directories:
     ```bash
     rm -rf .claude/worktrees/
     ```

2. **Update session index**: If `.on-loop/index.json` exists, update any `"active"` sessions to `"failed"` (they were abandoned by the clear)

3. **Delete session logs** (only if `--include-logs` is passed):
   ```bash
   rm -rf .on-loop/
   ```
   - Warn the user that this removes all session history

4. **Determine default branch**: Check if `main` or `master` is the default branch
   ```bash
   git rev-parse --verify main 2>/dev/null && echo main || echo master
   ```

5. **Switch to default branch**:
   ```bash
   git checkout <default-branch>
   ```

6. **Pull latest**:
   ```bash
   git pull origin <default-branch>
   ```

7. **Confirm to user**: Report:
   - Number of worktrees removed
   - Whether session logs were deleted
   - Current branch and that it's up to date with origin

## Error Handling

- If there are uncommitted changes on the current branch, warn the user before switching branches. Ask if they want to stash or discard changes.
- If `git pull` fails (e.g., no remote), warn but don't fail — the cleanup is still successful.
- If worktree removal fails, try force removal. If that also fails, report which worktrees couldn't be removed.
