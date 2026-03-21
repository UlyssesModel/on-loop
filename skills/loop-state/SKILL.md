---
name: loop-state
description: Manages on-loop phase state transitions, validation, and persistence
---

# Loop State Management

This skill manages the `.on-loop/state.json` lifecycle — creation, valid transitions, and persistence.

## Valid Phase Transitions

```
INIT → SPEC
SPEC → PLAN
PLAN → CODE
CODE → TEST
TEST → CODE          (retry: test failures)
TEST → SECURITY      (pass)
SECURITY → CODE      (retry: security blockers)
SECURITY → DOC       (pass, parallel with BUILD)
SECURITY → BUILD     (pass, parallel with DOC)
DOC → REVIEW         (when BUILD also complete)
BUILD → REVIEW       (when DOC also complete)
REVIEW → CODE        (retry: changes requested)
REVIEW → GIT         (approved)
GIT → COMPLETE       (commit, push, PR done)
ANY → FAILED         (unrecoverable error)
FAILED → ANY         (resume)
```

## State Operations

### Initialize State

```json
{
  "version": "1.0",
  "loop_id": "<uuid>",
  "prompt": "<user prompt>",
  "phase": "INIT",
  "started_at": "<ISO 8601>",
  "updated_at": "<ISO 8601>",
  "retries": {
    "test_to_code": 0,
    "security_to_code": 0,
    "review_to_code": 0
  },
  "max_retries": {
    "test_to_code": 3,
    "security_to_code": 2,
    "review_to_code": 2
  },
  "branch": null,
  "pr_url": null,
  "phases_completed": [],
  "current_agent": "orchestrator",
  "error": null,
  "todos": []
}
```

### Transition Phase

When transitioning:
1. Validate the transition is allowed (see valid transitions above)
2. Add the current phase to `phases_completed` (if not already there)
3. Update `phase` to the new phase
4. Update `updated_at` to current timestamp
5. Update `current_agent` to the agent for the new phase
6. Write `state.json`

### Record Retry

When a retry is triggered:
1. Increment the appropriate retry counter
2. Check if the counter exceeds the maximum
3. If within limit: transition back to CODE
4. If exhausted: record TODO and advance to next phase

### Record TODO

When retry limit is exhausted:
```json
{
  "phase": "<phase that failed>",
  "description": "<what wasn't resolved>",
  "severity": "<CRITICAL|HIGH|MEDIUM|LOW>",
  "details": "<specific issues>"
}
```

### Record Error

When an unrecoverable error occurs:
1. Set `phase` to `"FAILED"`
2. Set `error` to a description of what went wrong
3. Set `updated_at`
4. The loop can be resumed with `/on-loop-resume`

## Phase to Agent Mapping

| Phase | Agent | Notes |
|-------|-------|-------|
| INIT | orchestrator | Workspace setup |
| SPEC | architect | Spec generation |
| PLAN | orchestrator | Plan from spec |
| CODE | coding | Implementation or remediation |
| TEST | testing | Test generation and execution |
| SECURITY | security | Security audit (read-only) |
| DOC | documentation | Documentation generation |
| BUILD | build | Build infrastructure |
| REVIEW | reviewer | Final review gate (read-only) |
| GIT | orchestrator | Commit, push, create PR |
| COMPLETE | orchestrator | Summary and cleanup |

## Rules

1. Only the orchestrator agent writes `state.json`
2. All other agents read `state.json` to understand context
3. Phase transitions must follow the valid transition graph
4. Retry counters persist across transitions (never reset mid-loop)
5. TODOs are append-only during a loop run
6. The `version` field enables future schema migrations
