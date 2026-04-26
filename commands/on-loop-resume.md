---
name: on-loop-resume
description: Resume an interrupted SDLC loop from the last phase or a specified phase
user_invocable: true
argument: "[--from=phase] [--session=<name>]"
---

# /on-loop-resume

Resume an interrupted on-loop run from where it left off, or from a specified phase.

## Usage

```
/on-loop-resume                                              # Resume most recent non-complete session
/on-loop-resume --from=CODE                                  # Resume from a specific phase
/on-loop-resume --session=20260426_143052_user-management-api  # Resume a specific session
/on-loop-resume --session=<name> --from=CODE                  # Both
```

## Instructions

1. **Find the session to resume**:
   - If `--session=<name>` is provided, look for `.on-loop/sessions/<name>/state.json`
   - Otherwise, read `.on-loop/index.json` and find the most recent session with `status: "active"` or `status: "failed"`
   - If no session found, report that no loop exists to resume

2. Read the session's `state.json` to understand the current state.

3. **Verify worktree exists**:
   - Read `worktree_path` from `state.json`
   - Check if the worktree directory exists at that path
   - If not, recreate it: `git worktree add <worktree_path> <branch>`
   - If the branch no longer exists, report error and suggest starting fresh with `/on-loop`

4. If `--from=<phase>` is specified:
   - Validate the phase name is valid (INIT, SPEC, PLAN, CODE, TEST, SECURITY, DOC, BUILD, REVIEW)
   - Reset the state to that phase
   - Clear any error state
   - Note: Retry counts are NOT reset (to prevent infinite loops)

5. If no `--from` argument:
   - If phase is `"FAILED"`, resume from the phase that failed
   - If phase is any active phase, resume from that phase
   - If phase is `"COMPLETE"`, report the loop is already complete

6. Resume the orchestrator pipeline from the determined phase:
   - Re-read all existing agent notes from the session directory for context
   - Continue the normal phase sequence from the resume point
   - All agent work happens in the worktree directory
   - The orchestrator handles everything from here (same as `/on-loop`)

7. Update `.on-loop/index.json` session status back to `"active"` if it was `"failed"`

8. Report to the user:
   - Which session is being resumed (session name)
   - Which phase is being resumed from
   - The worktree path being used
   - What context is available from previous phases
   - Any warnings about state that may be stale

## Validation

- The session directory must exist with a valid `state.json`
- The resume phase must be a valid phase in the pipeline
- Warn if resuming from a phase earlier than the last completed phase (re-running work)
