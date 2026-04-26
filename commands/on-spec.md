---
name: on-spec
description: Generate a detailed specification from a feature description using the architect agent
user_invocable: true
argument: description of what to specify
---

# /on-spec

Standalone spec generation using the architect agent.

## Usage

```
/on-spec <description of the feature or system to specify>
```

## Instructions

1. Read the user's description.

2. Generate a session name: `YYYYMMDD_HHMMSS_spec-<slugified-description>` (e.g., `20260426_143052_spec-user-management`)

3. Create a session directory `.on-loop/sessions/<session-name>/`:
   - `state.json` with phase `"SPEC"` and the user's description as prompt
   - Empty `agent-notes/` directory
   - Update `.on-loop/index.json` (create if missing)

4. Dispatch the **architect agent** (`agents/architect.md`):
   - Provide the user's description
   - Provide the session directory path
   - Provide any existing project context (README, CLAUDE.md, code structure)

5. When complete, display the specification from the session's `agent-notes/architect.md` to the user.

6. Update `.on-loop/index.json` session status to `"complete"`.

7. The spec is left in the session directory for potential use by `/on-loop-resume` or other commands.

## Notes

- This command runs only the architect agent — it does NOT trigger the full SDLC loop
- Use `/on-loop` with the resulting spec to run the full pipeline
- The spec includes requirements, architecture, security considerations, and ADRs
