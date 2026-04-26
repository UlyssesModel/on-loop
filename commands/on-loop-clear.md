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

1. Merges associated PRs (blocks if any PR is not mergeable)
2. Removes all on-loop git worktrees
3. Optionally deletes `.on-loop/` session logs (with `--include-logs`)
4. Switches to `main` (or `master` if `main` doesn't exist)
5. Pulls the latest changes from origin

## Instructions

When this command is invoked:

1. **Merge associated PRs before cleanup**:
   - Read `.on-loop/index.json` (if it exists)
   - For each session with `status: "active"` or `status: "complete"` that has a `pr_url`:
     - Extract the PR number from the URL
     - Check if the PR is mergeable:
       ```bash
       gh pr view <number> --json mergeable,mergeStateStatus,state,title,statusCheckRollup
       ```
     - If the PR state is `"MERGED"` — already merged, continue with cleanup
     - If the PR state is `"CLOSED"` — skip (already closed), continue with cleanup
     - If `mergeable` is `"MERGEABLE"` and all status checks have passed — merge:
       ```bash
       gh pr merge <number> --merge --delete-branch
       ```
     - If the PR is **not mergeable** (failing checks, merge conflicts, pending reviews, or `mergeable` is `"CONFLICTING"` / `"UNKNOWN"`):
       - **STOP** and report:
         ```
         Cannot clear: PR #<number> (<title>) is not mergeable.
         Reason: <merge state status / failing checks>
         
         Resolve the PR before running /on-loop:clear, or close it manually with:
           gh pr close <number>
         ```
       - Do NOT remove any worktrees or clean up. Exit immediately.
   - For sessions without a `pr_url` that have `status: "active"`, warn the user:
     ```
     Session <session-name> has no PR. It will be marked as failed during cleanup.
     ```

2. **List and remove on-loop worktrees**:
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

3. **Update session index**: If `.on-loop/index.json` exists, update any `"active"` sessions to `"failed"` (they were abandoned by the clear)

4. **Delete session logs** (only if `--include-logs` is passed):
   ```bash
   rm -rf .on-loop/
   ```
   - Warn the user that this removes all session history

5. **Determine default branch**: Check if `main` or `master` is the default branch
   ```bash
   git rev-parse --verify main 2>/dev/null && echo main || echo master
   ```

6. **Switch to default branch**:
   ```bash
   git checkout <default-branch>
   ```

7. **Pull latest**:
   ```bash
   git pull origin <default-branch>
   ```

8. **Confirm to user**: Report:
   - Number of worktrees removed
   - Whether session logs were deleted
   - Current branch and that it's up to date with origin

## Error Handling

- If there are uncommitted changes on the current branch, warn the user before switching branches. Ask if they want to stash or discard changes.
- If `git pull` fails (e.g., no remote), warn but don't fail — the cleanup is still successful.
- If worktree removal fails, try force removal. If that also fails, report which worktrees couldn't be removed.
