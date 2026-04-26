# Testing Agent Notes

## Summary

Validated consistency and correctness of the 14-file v0.3.0 release. All versioning, schema, and path changes are internally consistent with one exception: 8 agent files and 2 skill files still use the old flat `.on-loop/state.json`, `.on-loop/plan.md`, `.on-loop/agent-notes/`, and `.on-loop/changes.log` paths instead of the new session-scoped `.on-loop/sessions/<session-id>/` structure. These are stale references that were not updated in this release and represent a significant gap between documentation and implementation intent.

## Test Results

- Total checks: 10 (per validation checklist)
- Passed: 8
- Failed: 2 (stale flat-path references across agents and skills)

## Checks Performed

### 1. Version Consistency — PASS

Both `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` show `"version": "0.3.0"`. Consistent.

### 2. Schema Consistency (state.json v1.1) — PASS

The state.json v1.1 schema is consistent across all four files that define or show it:

- `skills/loop-state/SKILL.md` — defines schema with `version`, `session_id`, `worktree_path`, `session_dir`. Correct.
- `shared/COMMUNICATION_PROTOCOL.md` — defines same schema with identical fields. Correct.
- `agents/orchestrator.md` — shows the full JSON example with all v1.1 fields. Correct.
- `commands/on-loop.md` — lists required fields in prose (version 1.1, session_id, worktree_path, session_dir). Correct.

### 3. Path Consistency (session-scoped) — FAIL

**orchestrator.md**, **on-loop.md**, **on-loop-resume.md**, **on-loop-status.md**, **on-loop-clear.md**, **on-continue.md**, **COMMUNICATION_PROTOCOL.md**, and **CLAUDE.md** all correctly reference `.on-loop/sessions/<session-id>/` for state, plan, changes.log, and agent-notes.

However, **all 7 specialist agent files and 2 skill files were NOT updated** and still use the old flat paths:

Stale flat-path references found:

| File | Stale Reference |
|------|----------------|
| `agents/architect.md` | `.on-loop/state.json`, `.on-loop/plan.md`, `.on-loop/agent-notes/architect.md`, `.on-loop/changes.log` |
| `agents/coding.md` | `.on-loop/state.json`, `.on-loop/plan.md`, `.on-loop/agent-notes/coding.md`, `.on-loop/changes.log` |
| `agents/testing.md` | `.on-loop/state.json`, `.on-loop/plan.md`, `.on-loop/agent-notes/testing.md` |
| `agents/security.md` | `.on-loop/state.json`, `.on-loop/plan.md`, `.on-loop/agent-notes/security.md`, `.on-loop/changes.log` |
| `agents/documentation.md` | `.on-loop/state.json`, `.on-loop/plan.md`, `.on-loop/agent-notes/documentation.md`, `.on-loop/changes.log` |
| `agents/build.md` | `.on-loop/state.json`, `.on-loop/plan.md`, `.on-loop/agent-notes/build.md`, `.on-loop/changes.log` |
| `agents/reviewer.md` | `.on-loop/state.json`, `.on-loop/plan.md`, `.on-loop/agent-notes/reviewer.md`, `.on-loop/changes.log` |
| `skills/quality-gate/SKILL.md` | `.on-loop/agent-notes/<agent>.md`, `.on-loop/plan.md` (all gate criteria use flat paths) |
| `skills/loop-state/SKILL.md` | Opening line "manages the `.on-loop/state.json` lifecycle" (the body schema is correct, but the intro sentence is stale) |
| `commands/on-spec.md` | `.on-loop/agent-notes/architect.md` (standalone command, may be intentional — see below) |
| `commands/on-security.md` | `.on-loop/agent-notes/security.md` (standalone command) |
| `commands/on-pause.md` | `.on-loop/agent-notes/` (in handoff summary template), `.on-loop/changes.log` |

Note: `commands/on-spec.md` and `commands/on-security.md` are standalone commands (not part of the session pipeline), so their use of flat paths may be intentional if those commands don't participate in sessions. This should be clarified.

### 4. Worktree Path Consistency — PASS

All files that reference the worktree location use `.claude/worktrees/<branch-slug>` consistently:
- `skills/loop-state/SKILL.md` — correct
- `shared/COMMUNICATION_PROTOCOL.md` — correct
- `agents/orchestrator.md` — correct (both in schema and Key Paths table)
- `commands/on-loop.md` — correct
- `commands/on-continue.md` — correct
- `commands/on-loop-resume.md` — correct (reads from state.json worktree_path)
- `commands/on-loop-clear.md` — correct (removes `.claude/worktrees/<slug>`)

### 5. index.json Schema Consistency — PASS

The index.json schema is shown in two places and is identical:

- `shared/COMMUNICATION_PROTOCOL.md`: version, sessions array with session_id, loop_id, prompt, branch, status, started_at, completed_at, worktree_path, pr_url
- `agents/orchestrator.md`: same fields in the same structure

`commands/on-loop.md` prose description (step 5) refers to "append session entry with `status: active`" and lists the same fields. Consistent.

### 6. Stale References — FAIL

Confirmed stale references in agents and skills (detailed in check 3 above). Additionally:

- `skills/loop-state/SKILL.md` line 8: "This skill manages the `.on-loop/state.json` lifecycle" — the intro description is stale. The schema block is correct (it uses session-scoped paths implicitly), but the sentence implies state.json is at the flat path.
- `commands/on-pause.md` line 94 and 131: references `.on-loop/agent-notes/` and `.on-loop/changes.log` without session scoping.
- `.on-loop/` directory in the repo currently has the OLD flat structure (state.json, plan.md, changes.log, agent-notes/ at root of .on-loop/) — this is the workspace for the current on-loop session managing this PR. It is not a bug in the plugin files, but it means the migration is not yet reflected in the running workspace.

### 7. .gitignore Correctness — PASS

- `.on-loop/` is NOT in `.gitignore` (the old entry was removed). Correct.
- `.claude/worktrees/` IS in `.gitignore` (line 192: `# Claude Code worktrees` / `.claude/worktrees/`). Correct.

### 8. Hook Correctness — PASS

Both hooks in `hooks/hooks.json` correctly use session-scoped paths via index.json:

- `on-loop-active-reminder`: reads `.on-loop/index.json`, loops sessions. Correct.
- `on-loop-change-tracker`: reads `.on-loop/index.json`, extracts `session_dir` from the active session entry, writes to `$SESSION_DIR/changes.log`. Correct — this will resolve to `.on-loop/sessions/<id>/changes.log`.

### 9. Command Argument Consistency — PASS

- `commands/on-loop-resume.md` frontmatter argument: `"[--from=phase] [--session=<id>]"` — correct. Usage block shows `--session=<session-id>`. Correct.
- `commands/on-loop-clear.md` frontmatter argument: `"[--include-logs]"` — correct. Usage block shows `--include-logs`. Consistent.

### 10. Cross-Reference Validity — PASS

Verified files referenced from within the changed files:
- `skills/quality-gate/SKILL.md` — exists at `/Users/charmalloc/dev/on-loop/skills/quality-gate/SKILL.md`
- `skills/loop-state/SKILL.md` — exists
- `skills/roadmap-state/SKILL.md` — exists (referenced from on-continue.md)
- `skills/roadmap-lock/SKILL.md` — exists (referenced from on-continue.md)
- `shared/AGENT_PERSONA.md` — exists (referenced via @shared/AGENT_PERSONA.md in orchestrator)
- `shared/QUALITY_STANDARDS.md` — exists
- All 8 agent files (`agents/architect.md`, `agents/coding.md`, etc.) — all exist

No broken cross-references found.

## Issues Found

- [HIGH] Agents not updated to session-scoped paths — `agents/architect.md`, `agents/coding.md`, `agents/testing.md`, `agents/security.md`, `agents/documentation.md`, `agents/build.md`, `agents/reviewer.md` all still reference `.on-loop/state.json`, `.on-loop/plan.md`, `.on-loop/agent-notes/<agent>.md`, and `.on-loop/changes.log`. When these agents run under v0.3.0 orchestration, they will be directed to the session-scoped directories via the orchestrator dispatch, but their own internal process instructions will send them to the wrong paths if followed literally. This creates a contradiction between the orchestrator's instructions and the agents' own self-descriptions.

- [HIGH] `skills/quality-gate/SKILL.md` not updated — all gate criteria reference `.on-loop/agent-notes/<agent>.md` and `.on-loop/plan.md` (flat paths). The orchestrator reads this skill to determine quality gate pass/fail. If the orchestrator follows the skill literally it will look for files in the wrong location.

- [MEDIUM] `commands/on-pause.md` not fully updated — the handoff summary template (step 6) refers to `.on-loop/agent-notes/` and step "Edge Cases" section refers to `.on-loop/changes.log` without session scoping. The main flow (steps 2 and 4) correctly uses `<session-id>` scoping, making the file partially updated.

- [MEDIUM] `skills/loop-state/SKILL.md` opening sentence stale — "This skill manages the `.on-loop/state.json` lifecycle" should read "This skill manages the `.on-loop/sessions/<session-id>/state.json` lifecycle". Minor but misleading.

- [LOW] `commands/on-spec.md` and `commands/on-security.md` use flat `.on-loop/agent-notes/` paths. If these standalone commands are meant to be session-aware in v0.3.0, they also need updating. If they intentionally operate outside the session model, this should be documented.

## Decisions

- The validation is conducted as document consistency analysis since there is no executable code to run tests against. Pass/fail is determined by whether the specification documents are internally consistent.
- Standalone commands (`/on-spec`, `/on-security`, `/on-test`, etc.) were not part of the stated change set. Their stale paths are noted as LOW severity pending clarification of their intended session behavior.

## Files Modified

- `.on-loop/agent-notes/testing.md` — this file (created)
- `.on-loop/changes.log` — appended

## Failures Detail

### Agent files still reference old flat .on-loop paths
- **Expected**: All agent files read from `.on-loop/sessions/<session-id>/state.json`, `.on-loop/sessions/<session-id>/plan.md`, write to `.on-loop/sessions/<session-id>/agent-notes/<agent>.md`, and append to `.on-loop/sessions/<session-id>/changes.log`
- **Actual**: All 7 specialist agent files instruct agents to use `.on-loop/state.json`, `.on-loop/plan.md`, `.on-loop/agent-notes/<agent>.md`, `.on-loop/changes.log` (flat paths from v0.2.0)
- **Root Cause**: The coding pass updated the orchestrator, command files, and protocol docs but did not update the individual agent definitions
- **Fix Recommendation**: Update each agent file's "Context" / "Process" / "Read before write" section to use `<session-dir>/state.json`, `<session-dir>/plan.md`, `<session-dir>/agent-notes/<agent>.md`, and `<session-dir>/changes.log`. The session dir is provided to each agent at dispatch time by the orchestrator.

### quality-gate/SKILL.md gate criteria use flat paths
- **Expected**: Gate criteria reference `.on-loop/sessions/<session-id>/agent-notes/<agent>.md`
- **Actual**: Gate criteria reference `.on-loop/agent-notes/<agent>.md`
- **Root Cause**: Same as agent files — this skill was not updated in this coding pass
- **Fix Recommendation**: Update all path literals in gate criteria tables to use the session-scoped path pattern, or use a variable like `<session-dir>/agent-notes/<agent>.md`

## Recommendations for Next Agent

- The coding agent must update all 7 specialist agent files (`agents/architect.md`, `agents/coding.md`, `agents/testing.md`, `agents/security.md`, `agents/documentation.md`, `agents/build.md`, `agents/reviewer.md`) to replace flat `.on-loop/` paths with session-scoped `<session-dir>/` paths throughout their "Context Gathering" and output sections
- The coding agent must update `skills/quality-gate/SKILL.md` to use session-scoped paths in all gate criteria
- The coding agent should update the opening sentence of `skills/loop-state/SKILL.md` to reflect the session-scoped location
- The coding agent should review `commands/on-pause.md` lines 94 and 131 for the remaining flat-path references
- Consider explicitly documenting whether standalone commands (`/on-spec`, `/on-security`, `/on-test`) are session-aware in v0.3.0 or intentionally operate outside sessions
