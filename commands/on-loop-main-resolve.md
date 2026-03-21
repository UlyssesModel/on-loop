---
name: on-loop-main-resolve
description: Pull main and merge into current feature branch, resolving conflicts
user_invocable: true
argument: none
---

# /on-loop:main-resolve

Pull the latest main branch and merge it into your current feature branch, resolving any conflicts.

## Usage

```
/on-loop:main-resolve
```

## What This Does

1. Verifies you're on a feature branch (not main)
2. Fetches and merges the latest main into your branch
3. Resolves any merge conflicts intelligently

## Instructions

When this command is invoked:

1. **Check current branch**:
   ```bash
   git branch --show-current
   ```
   - If on `main` or `master`: report an error — this command is for feature branches only. Suggest using `git pull` instead.

2. **Determine default branch**: Check if `main` or `master` is the default branch
   ```bash
   git rev-parse --verify main 2>/dev/null && echo main || echo master
   ```

3. **Fetch latest main**:
   ```bash
   git fetch origin <default-branch>
   ```

4. **Merge main into current branch**:
   ```bash
   git merge origin/<default-branch>
   ```

5. **If conflicts occur**:
   - Read each conflicted file
   - Resolve conflicts intelligently using project context, preferring to preserve both sides' intent
   - Stage resolved files: `git add <resolved-file>`
   - Complete the merge: `git commit` (accept the default merge commit message)

6. **Report result to user**:
   - If clean merge: report success, list any new commits pulled in
   - If conflicts resolved: report which files had conflicts and how they were resolved
   - If merge failed for other reasons: report the error

## Error Handling

- If there are uncommitted changes: warn the user and suggest committing or stashing first
- If fetch fails: report network/remote error
- If conflict resolution is ambiguous: show the conflict to the user and ask for guidance rather than guessing
