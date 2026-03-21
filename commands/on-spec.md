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

2. Create a minimal `.on-loop/` workspace if one doesn't exist:
   - `state.json` with phase `"SPEC"` and the user's description as prompt
   - Empty `agent-notes/` directory

3. Dispatch the **architect agent** (`agents/architect.md`):
   - Provide the user's description
   - Provide any existing project context (README, CLAUDE.md, code structure)

4. When complete, display the specification from `.on-loop/agent-notes/architect.md` to the user.

5. The spec is left in `.on-loop/agent-notes/architect.md` for potential use by `/on-loop-resume` or other commands.

## Notes

- This command runs only the architect agent — it does NOT trigger the full SDLC loop
- Use `/on-loop` with the resulting spec to run the full pipeline
- The spec includes requirements, architecture, security considerations, and ADRs
