---
name: on-pause
description: Release locks, commit WIP, update state, and write a handoff summary
user_invocable: true
argument: feature slug (optional -- auto-detects if only one roadmap exists)
---

# /on-pause

Gracefully pauses work on a roadmap feature. Releases all locks held by the current session, commits any work-in-progress, updates the state file, and writes a handoff summary so another session (or the same session later) can pick up where this one left off.

## Usage

```
/on-pause                     # auto-detect feature
/on-pause <feature-slug>      # specify which feature
```

## Instructions

When this command is invoked:

### 1. Identify the Roadmap

Same auto-detection logic as `/on-continue`:
- If slug provided, use it
- If not, auto-detect from `roadmap/.state/`
- If none found, report no active roadmap

### 2. Identify Current Session

Determine the current session by checking:
1. Read `roadmap/.state/_global.json`
2. Look for active sessions that match this environment (check locks in the feature state file)
3. If no active session is found for this environment, report "No active session to pause" and exit

### 3. Assess Work in Progress

For each step locked by the current session:
1. Read the step's status
2. Check if `.on-loop/agent-notes/` has any notes from the current execution
3. Determine the state of the step:
   - **Completed but not yet marked**: Mark it complete
   - **In-progress with partial work**: Commit WIP
   - **Just started, no meaningful work**: Release lock, leave as `not-started`

### 4. Commit Work in Progress

If there are uncommitted changes:

1. Check `git status` for modified/new files
2. Read `.on-loop/changes.log` if it exists for context
3. Stage all modified files related to the current step (explicit paths, not `git add -A`)
4. Commit with message: `wip(<feature-slug>): phase <N> step <M> - pausing work`
   - End with `Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>`
5. Include the updated state file in the commit

### 5. Release Locks

Following `skills/roadmap-lock/SKILL.md` "Release All Locks":

1. Read the feature state file
2. Find all locks belonging to the current session
3. For each locked step:
   - If the step has meaningful WIP (files were modified), leave status as `in-progress` so the lock TTL mechanism handles eventual cleanup
   - If no meaningful work was done, reset status to `not-started`
4. Remove all lock entries for this session from the feature state file
5. Remove the session from `roadmap/.state/_global.json`
6. Write both files

### 6. Write Handoff Summary

Write a handoff summary to `roadmap/.state/<feature-slug>-handoff.md`:

```markdown
# Handoff Summary: <Feature Title>

**Paused at**: <ISO 8601 timestamp>
**Session**: <session-id>
**Branch**: <current branch>

## Status at Pause

### Phase <N>: <Title>
- Steps completed: X / Y
- Steps in-progress (WIP): Z
- Steps remaining: W

## Work in Progress

### Phase <N>, Step <M>: <Title>
- **Status**: <what was done>
- **Files modified**: <list>
- **Agent notes**: <summary from .on-loop/agent-notes/ if available>
- **What remains**: <what the next session should do>

## Recommendations for Next Session

- Run `/on-continue <feature-slug>` to pick up the next available step
- Step <M> has WIP committed -- the next `/on-continue` for that step should review existing work before continuing
- <Any other context-specific advice>
```

### 7. Update State

Update `roadmap/.state/<feature-slug>.json`:
- Clear all locks for this session
- Update `last_activity` timestamp
- Ensure phase/step statuses accurately reflect current state

### 8. Report

Display to the user:
- Steps that were paused and their status
- Whether WIP was committed (and the commit hash)
- Lock releases performed
- Handoff summary file path
- How to resume: `/on-continue <feature-slug>`
- How to check status: cat the state file or roadmap

## Edge Cases

### No Active Locks

If the session has no active locks:
- Still write a handoff summary if there is useful context
- Report "No active locks to release"

### Uncommitted Changes Unrelated to Roadmap

If `git status` shows changes not tracked in `.on-loop/changes.log`:
- Warn the user about untracked changes
- Do NOT commit them automatically
- Suggest the user handles them manually

### Multiple Features

If the session has locks across multiple features:
- If a specific slug is provided, only pause that feature
- If auto-detecting, pause all features with active locks for this session

## Important

- Always release locks, even if the commit fails
- Never leave locks orphaned -- the whole point of `/on-pause` is clean shutdown
- The handoff summary is committed to the repo so the next person can read it
- After `/on-pause`, the session ID is invalidated (removed from global state)
