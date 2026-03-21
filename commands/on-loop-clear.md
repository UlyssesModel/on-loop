---
name: on-loop-clear
description: Clean up on-loop workspace, switch to main, and pull latest
user_invocable: true
argument: none
---

# /on-loop:clear

Clean up the on-loop workspace and return to a clean main branch state.

## Usage

```
/on-loop:clear
```

## What This Does

1. Deletes the `.on-loop/` workspace directory (if it exists)
2. Switches to `main` (or `master` if `main` doesn't exist)
3. Pulls the latest changes from origin

## Instructions

When this command is invoked:

1. **Delete workspace**: Remove the `.on-loop/` directory if it exists
   ```bash
   rm -rf .on-loop/
   ```

2. **Determine default branch**: Check if `main` or `master` is the default branch
   ```bash
   git rev-parse --verify main 2>/dev/null && echo main || echo master
   ```

3. **Switch to default branch**:
   ```bash
   git checkout <default-branch>
   ```

4. **Pull latest**:
   ```bash
   git pull origin <default-branch>
   ```

5. **Confirm to user**: Report that the workspace has been cleaned up, you're on the default branch, and it's up to date with origin.

## Error Handling

- If there are uncommitted changes on the current branch, warn the user before switching branches. Ask if they want to stash or discard changes.
- If `git pull` fails (e.g., no remote), warn but don't fail — the cleanup is still successful.
