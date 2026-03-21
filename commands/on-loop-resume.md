---
name: on-loop-resume
description: Resume an interrupted SDLC loop from the last phase or a specified phase
user_invocable: true
argument: "[--from=phase]"
---

# /on-loop-resume

Resume an interrupted on-loop run from where it left off, or from a specified phase.

## Usage

```
/on-loop-resume              # Resume from last phase
/on-loop-resume --from=CODE  # Resume from a specific phase
```

## Instructions

1. Check if `.on-loop/state.json` exists. If not, report that no loop exists to resume.

2. Read `.on-loop/state.json` to understand the current state.

3. If `--from=<phase>` is specified:
   - Validate the phase name is valid (INIT, SPEC, PLAN, CODE, TEST, SECURITY, DOC, BUILD, REVIEW)
   - Reset the state to that phase
   - Clear any error state
   - Note: Retry counts are NOT reset (to prevent infinite loops)

4. If no `--from` argument:
   - If phase is `"FAILED"`, resume from the phase that failed
   - If phase is any active phase, resume from that phase
   - If phase is `"COMPLETE"`, report the loop is already complete

5. Resume the orchestrator pipeline from the determined phase:
   - Re-read all existing agent notes for context
   - Continue the normal phase sequence from the resume point
   - The orchestrator handles everything from here (same as `/on-loop`)

6. Report to the user:
   - Which phase is being resumed from
   - What context is available from previous phases
   - Any warnings about state that may be stale

## Validation

- The `.on-loop/` workspace must exist
- `state.json` must be valid and readable
- The resume phase must be a valid phase in the pipeline
- Warn if resuming from a phase earlier than the last completed phase (re-running work)
