---
name: on-loop
description: Run a full spec-driven SDLC loop with orchestrated specialist agents
user_invocable: true
argument: prompt or path to spec file
---

# /on-loop

Run a complete spec-driven software development lifecycle for the given prompt.

## Usage

```
/on-loop <prompt describing what to build>
/on-loop <path-to-spec-file>
```

## What This Does

Launches the **orchestrator agent** which drives the following pipeline:

1. **INIT** — Creates `.on-loop/` workspace, creates feature branch if on main
2. **SPEC** — Architect agent generates a detailed specification
3. **PLAN** — Orchestrator writes an implementation plan
4. **CODE** — Coding agent implements the spec
5. **TEST** — Testing agent writes and runs tests (retries up to 3x on failure)
6. **SECURITY** — Security agent audits the code (retries up to 2x on blockers)
7. **DOC + BUILD** — Documentation and build agents run in parallel
8. **REVIEW** — Reviewer agent performs final code review (retries up to 2x)
9. **GIT** — Commit all changes, push branch, create PR
10. **COMPLETE** — Summary of everything built with PR link

## Instructions

When this command is invoked:

1. Read the user's argument. If it's a file path, read the file contents as the prompt.

2. **Branch creation** (before workspace init):
   - Check current branch with `git branch --show-current`
   - If on `main` or `master`: create and checkout `on-loop/<slugified-prompt>` (lowercase, hyphens, max 50 chars)
   - If already on a feature branch: stay on it

3. Initialize the `.on-loop/` workspace:
   - Create `.on-loop/` directory with `agent-notes/` subdirectory
   - Create `state.json` with phase `"INIT"`, the user's prompt, and `"branch"` set to current branch name
   - Create empty `plan.md` and `changes.log`

4. Dispatch the **architect agent** (`agents/architect.md`):
   - Provide the user's prompt
   - The architect writes the spec to `.on-loop/agent-notes/architect.md`

5. Write `plan.md` based on the architect's spec output.

6. Update `state.json` to phase `"CODE"` and dispatch the **coding agent** (`agents/coding.md`):
   - Provide `plan.md` and architect's notes

7. Update to phase `"TEST"` and dispatch the **testing agent** (`agents/testing.md`):
   - Provide `plan.md`, architect's notes, and coding agent's notes
   - If tests fail and retries remain (max 3), go back to CODE with test feedback
   - If retries exhausted, record TODOs and continue

8. Update to phase `"SECURITY"` and dispatch the **security agent** (`agents/security.md`):
   - Provide all prior agent notes
   - If CRITICAL/HIGH findings and retries remain (max 2), go back to CODE with security feedback
   - If retries exhausted, record TODOs and continue

9. Update to phase `"DOC"` and `"BUILD"` — dispatch **documentation** (`agents/documentation.md`) and **build** (`agents/build.md`) agents in parallel.

10. Update to phase `"REVIEW"` and dispatch the **reviewer agent** (`agents/reviewer.md`):
    - Provide all agent notes
    - If REQUEST_CHANGES and retries remain (max 2), go back to CODE with review feedback
    - If retries exhausted, record TODOs and continue

11. Update to phase `"GIT"` (orchestrator handles directly):
    - Stage all modified/created files from `changes.log` (explicit paths, not `git add -A`)
    - Commit with a descriptive message summarizing the work, ending with `Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>`
    - Push branch to origin with `-u` flag
    - Create PR via `gh pr create` with title from prompt and body with summary, files changed, test results, security findings, TODOs
    - Store PR URL in `state.json` as `"pr_url"`

12. Update to phase `"COMPLETE"`:
    - Print a summary of what was built
    - List files created/modified
    - Report test results
    - Report security findings
    - List any outstanding TODOs
    - Display the PR URL

## Error Handling

If any phase fails unexpectedly:
- Set `state.json` phase to `"FAILED"` with error details
- Report the failure to the user
- Suggest using `/on-loop-resume` to continue

## Important

- All inter-agent communication goes through `.on-loop/` files
- Only the orchestrator writes `state.json`
- The `.on-loop/` directory is gitignored and ephemeral
- Each agent reads `shared/AGENT_PERSONA.md` for the Staff Engineer + ISC2 persona
