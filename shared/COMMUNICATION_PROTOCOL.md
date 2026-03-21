# Communication Protocol

## Workspace: `.on-loop/`

All inter-agent communication happens through the `.on-loop/` directory in the project root. This directory is gitignored and ephemeral — each `/on-loop` invocation starts fresh.

## Workspace Structure

```
.on-loop/
├── state.json              # Phase tracking (orchestrator-only writes)
├── plan.md                 # Implementation plan (all agents read)
├── changes.log             # Append-only file modification log
└── agent-notes/
    ├── architect.md        # Architect agent output
    ├── coding.md           # Coding agent output
    ├── testing.md          # Testing agent output
    ├── security.md         # Security agent output
    ├── documentation.md    # Documentation agent output
    ├── build.md            # Build agent output
    └── reviewer.md         # Reviewer agent output
```

## state.json Schema

```json
{
  "version": "1.0",
  "loop_id": "<uuid>",
  "prompt": "<original user prompt>",
  "phase": "INIT | SPEC | PLAN | CODE | TEST | SECURITY | DOC | BUILD | REVIEW | GIT | COMPLETE | FAILED",
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
  "current_agent": "<agent name>",
  "error": null,
  "todos": []
}
```

### Rules

- **Only the orchestrator writes `state.json`**. All other agents read it.
- Phase transitions must follow the valid sequence (see `skills/loop-state/SKILL.md`).
- When a retry limit is exhausted, the orchestrator records a TODO and advances to the next phase.

## plan.md Format

Written by the orchestrator after the SPEC phase. All agents reference this.

```markdown
# Implementation Plan

## Objective
<one-line summary>

## Spec Reference
<key decisions from architect's spec>

## Tasks
1. <task> — assigned to <agent>
2. <task> — assigned to <agent>
...

## Constraints
- <constraint from spec>

## Open Questions
- <question> — owner: <agent>
```

## changes.log Format

Append-only. Every agent appends when it creates, modifies, or deletes a file.

```
[<ISO 8601>] <agent> <action> <file_path> — <reason>
```

Example:
```
[2026-03-21T10:15:00Z] coding CREATE src/api/handler.ts — implement POST /users endpoint
[2026-03-21T10:16:30Z] coding MODIFY src/api/handler.ts — add input validation
[2026-03-21T10:20:00Z] testing CREATE tests/api/handler.test.ts — unit tests for POST /users
```

## Agent Notes Format

Each agent writes structured notes to `agent-notes/<agent>.md`:

```markdown
# <Agent Name> Notes

## Summary
<2-3 sentence summary of what was done>

## Decisions
- <decision made and rationale>

## Files Modified
- `<path>` — <what changed>

## Issues Found
- [SEVERITY] <description> — <recommendation>

## Recommendations for Next Agent
- <actionable recommendation>
```

### Severity Levels (for Issues)

| Level | Meaning |
|-------|---------|
| CRITICAL | Must fix before merge; security vulnerability or data loss risk |
| HIGH | Should fix before merge; significant correctness or reliability issue |
| MEDIUM | Fix soon; code quality, performance, or maintainability concern |
| LOW | Nice to have; style, naming, minor improvements |
| INFO | Observation; no action required |

## Rules for All Agents

1. **Read before write** — Always read `state.json` and `plan.md` before starting work
2. **Append to changes.log** — Log every file operation
3. **Write agent notes** — Always write structured notes when your phase completes
4. **Respect boundaries** — Only perform actions within your agent's responsibility
5. **Flag blockers** — If you cannot proceed, document the blocker in your agent notes and set an issue with CRITICAL severity
6. **No direct agent-to-agent communication** — All information flows through `.on-loop/` files
