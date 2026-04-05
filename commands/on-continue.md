---
name: on-continue
description: Pick up the next available step from the roadmap and execute it through the full agent pipeline
user_invocable: true
argument: feature slug (optional -- auto-detects if only one roadmap exists)
---

# /on-continue

The workhorse command. Reads the roadmap state, finds the next unclaimed and non-conflicting step, acquires a lock on it, and executes it through the full on-loop agent pipeline (spec->code->test->security->docs->review).

## Usage

```
/on-continue                  # auto-detect feature, pick next step
/on-continue <feature-slug>   # specify which feature
```

Multiple sessions can run `/on-continue` concurrently on the same repo. Each session picks up the next available step that does not conflict with any in-progress work.

## Instructions

When this command is invoked:

### 1. Initialize Session

1. Generate a session ID: `python3 -c "import uuid; print(str(uuid.uuid4()))"`
2. Store the session ID for the duration of this execution

### 2. Identify the Roadmap

Same logic as `/on-plan`:
- If slug provided, use it
- If not, auto-detect from `roadmap/.state/`
- If none found, suggest `/on-prepare`

### 3. Read State and Find Next Step

Following the `skills/roadmap-state/SKILL.md` "Identify Next Step" algorithm:

1. Read `roadmap/.state/<feature-slug>.json`
2. Read `roadmap/.state/_global.json`
3. Identify the current phase
4. If the current phase is complete, advance to the next non-complete phase
5. Within the phase, find the first eligible step:
   - Status is `not-started` (or `failed` -- can be retried)
   - Not `exclusive` while other steps are in-progress
   - No `conflicts_with` steps are in-progress
   - No file-level overlap with locked files
6. If no step is available:
   - If steps are `in-progress` (other sessions working), report "All available steps are currently in progress. Try again later or run `/on-pause` in other sessions."
   - If all steps are `complete` in all phases, report "Feature complete! All steps in all phases are done."
   - If steps are `blocked`, report which steps are blocked and why

### 4. Acquire Lock

Following `skills/roadmap-lock/SKILL.md`:

1. Check for stale locks on the target step (heartbeat > 10 min old)
2. If stale, reclaim it (log warning)
3. Write lock entry in the feature state file
4. Register session in `_global.json`
5. Verify lock with read-after-write
6. If lock acquisition fails (race condition), back off and try the next eligible step

### 5. Set Up Ephemeral Workspace

Create or clean the `.on-loop/` workspace for this step's execution:

1. Create `.on-loop/` directory with `agent-notes/` subdirectory
2. Write `.on-loop/state.json` with:
   - Phase: `"CODE"` (the step starts at code since the roadmap IS the spec/plan)
   - Prompt: The step's description from the roadmap
   - Context: phase title, step number, feature name
3. Write `.on-loop/plan.md` with:
   - The step's detailed description from the roadmap
   - Files to create/modify
   - Acceptance criteria from the phase
   - Any relevant context from other completed steps
4. Create empty `.on-loop/changes.log`

### 6. Execute Agent Pipeline

Run the on-loop agent pipeline for this step. The pipeline is adapted for step-level execution:

#### 6a. Coding Agent

Dispatch the **coding agent** (`agents/coding.md`):
- Provide the step plan from `.on-loop/plan.md`
- Provide context about the broader feature from the roadmap
- The coding agent implements the step

**Update heartbeat** after coding completes.

#### 6b. Testing Agent

Dispatch the **testing agent** (`agents/testing.md`):
- Provide coding agent's notes
- Focus tests on the specific files created/modified in this step
- If tests fail and retry budget allows (max 2 retries for step-level), loop back to coding

**Update heartbeat** after testing completes.

#### 6c. Security Agent

Dispatch the **security agent** (`agents/security.md`):
- Provide all prior agent notes
- Focus security review on this step's files
- If CRITICAL findings and retry budget allows (max 1 retry for step-level), loop back to coding

**Update heartbeat** after security completes.

#### 6d. Review Agent

Dispatch the **reviewer agent** (`agents/reviewer.md`):
- Provide all agent notes
- If REQUEST_CHANGES and retry budget allows (max 1 retry for step-level), loop back to coding

**Update heartbeat** after review completes.

Note: Documentation and build phases are skipped for individual steps. They are handled at the phase level when all steps complete, or via dedicated `/on-doc` and `/on-build` invocations.

### 7. Mark Step Complete

After successful pipeline execution:

1. Update the feature state file:
   - Set step status to `complete`
   - Set step `completed_at`
   - Increment phase `steps_completed`
2. Release the lock (per `skills/roadmap-lock/SKILL.md`)
3. Remove session from `_global.json`
4. Check if the phase is now complete (all steps done)
5. If phase complete, update phase status

### 8. Commit Step Work

After marking the step complete:

1. Read `.on-loop/changes.log` for files modified
2. Stage all modified/created files (explicit paths from changes.log)
3. Commit with message: `feat(<feature-slug>): phase <N> step <M> - <step title>`
   - End with `Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>`
4. Also stage and commit the updated state file `roadmap/.state/<feature-slug>.json`

### 9. Handle Failure

If the agent pipeline fails:

1. Mark the step as `failed` in the state file
2. Release the lock
3. Remove session from `_global.json`
4. Report the failure with details
5. The step will be available for retry on the next `/on-continue` invocation

### 10. Report

Display to the user:
- Which step was executed (phase N, step M, title)
- Summary of what was implemented
- Test results
- Security findings
- Files created/modified
- Whether the phase is now complete
- How many steps remain in the current phase
- Suggest running `/on-continue` again for the next step, or `/on-pause` to stop

## Concurrency Model

When 3 sessions run `/on-continue` simultaneously:

```
Session A: reads state -> picks step 1 -> locks step 1 -> executes -> completes
Session B: reads state -> picks step 2 -> locks step 2 -> executes -> completes
Session C: reads state -> step 1 locked, step 2 locked -> picks step 3 -> locks step 3 -> executes
```

The lock check happens at acquisition time. If two sessions race for the same step, the read-after-write verification ensures only one proceeds.

## Step-Level Retry Budget

Within a single step execution, the retry budget is tighter than the full `/on-loop`:

| Transition | Max Retries |
|-----------|-------------|
| TEST -> CODE | 2 |
| SECURITY -> CODE | 1 |
| REVIEW -> CODE | 1 |

This keeps individual step execution bounded in time.

## Important

- Each `/on-continue` invocation handles exactly ONE step
- The `.on-loop/` workspace is ephemeral per step (cleaned at start)
- State files in `roadmap/.state/` are the persistent record
- Always commit after a successful step
- Always release locks, even on failure
- Heartbeat is updated between each agent phase
