---
name: on-loop-status
description: Check the progress of the current SDLC loop
user_invocable: true
---

# /on-loop-status

Display the current status of an active or completed on-loop run.

## Instructions

1. Check if `.on-loop/state.json` exists. If not, report that no loop is active.

2. Read `.on-loop/state.json` and display:
   - **Loop ID**: The unique identifier
   - **Phase**: Current phase (with human-readable label)
   - **Started**: When the loop began
   - **Last Updated**: When state was last modified
   - **Current Agent**: Which agent is active (if any)
   - **Phases Completed**: List of completed phases with checkmarks
   - **Retries Used**: Show retry counts vs maximums
   - **TODOs**: Any recorded TODOs from exhausted retries
   - **Error**: Any error state

3. If agent notes exist, show a brief summary from each completed agent:
   - Read each file in `.on-loop/agent-notes/`
   - Extract and display the `## Summary` section

4. Format the output as a clear status dashboard:

```
On-Loop Status
══════════════
Loop ID:  <uuid>
Phase:    <phase> (<description>)
Started:  <timestamp>
Updated:  <timestamp>
Agent:    <current agent>

Progress:
  ✅ INIT
  ✅ SPEC — <brief summary>
  ✅ PLAN
  ✅ CODE — <brief summary>
  🔄 TEST — In progress
  ⬜ SECURITY
  ⬜ DOC
  ⬜ BUILD
  ⬜ REVIEW

Retries:
  test→code:     1/3
  security→code: 0/2
  review→code:   0/2

TODOs: None
```
