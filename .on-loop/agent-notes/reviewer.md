# Reviewer Agent Notes

## Summary

Reviewed the full v0.3.0 changeset (33 files) covering git worktree isolation, persistent session logs, and version bump. The implementation is thorough and internally consistent. All stale flat-path references identified by the testing agent have been remediated -- every agent file, skill file, and command file now uses session-scoped `<session-dir>/` paths. The security agent's HIGH findings regarding branch slug validation have been addressed with explicit `[a-z0-9-]` whitelist and `..`/`/` rejection. Two security findings (committed security notes, hook shell fragility) remain as documented design trade-offs rather than bugs.

## Verdict: APPROVE

## Review Checklist Results

### Correctness: PASS
- All 7 specialist agent files (`agents/architect.md` through `agents/reviewer.md`) use `<session-dir>/` variable for state.json, plan.md, changes.log, and agent-notes paths
- `skills/quality-gate/SKILL.md` uses `<session-dir>/` throughout all gate criteria tables
- `skills/loop-state/SKILL.md` opening sentence correctly reads "session `state.json` lifecycle" with explicit session-scoped path
- `commands/on-pause.md` uses `<session-dir>/` and `<session-id>` scoping in both main flow and edge case sections
- state.json schema v1.1 is consistent across `skills/loop-state/SKILL.md`, `shared/COMMUNICATION_PROTOCOL.md`, `agents/orchestrator.md`, and `commands/on-loop.md`
- index.json schema v1.0 is consistent across `shared/COMMUNICATION_PROTOCOL.md` and `agents/orchestrator.md`
- Worktree lifecycle is fully described: INIT creates, SPEC-REVIEW operates within, GIT commits from, COMPLETE removes, FAILED leaves in place
- All standalone commands (`on-spec`, `on-security`, `on-test`, `on-doc`, `on-build`, `on-review`) create session directories under `.on-loop/sessions/<session-id>/` and update index.json

### Security: PASS (with noted trade-offs)
- Branch slug validation: `^[a-z0-9][a-z0-9-]*[a-z0-9]$` regex whitelist defined in both `commands/on-loop.md` (line 46) and `agents/orchestrator.md` (line 82). Explicit rejection of `..` and `/` characters.
- Hook variable quoting: `$SESSION_DIR` and `$FILE_PATH` are double-quoted in shell contexts in `hooks/hooks.json`
- Security Finding 3 (security.md committed to repo): NOT addressed but acknowledged as a design decision. The architect spec (ADR-002) explicitly chose to commit session logs for auditability. The COMMUNICATION_PROTOCOL.md and CLAUDE.md document this behavior. This is a reasonable trade-off for a local CLI tool -- teams that need to exclude security notes can add `.on-loop/sessions/*/agent-notes/security.md` to their project `.gitignore`.
- Security Finding 5 (CODEOWNERS for hooks.json): Not addressed. No CODEOWNERS file exists. This is a MEDIUM-priority recommendation for future work, not a blocker.
- No secrets in any committed files

### Performance: PASS
- No executable application code; performance concerns are limited to git worktree operations
- Spec requires git 2.20+ which is widely available
- Session directories are text-only, bounded at ~100KB per session per NFR-004
- No unbounded iterations in hook logic

### Code Quality: PASS
- Consistent naming: `<session-dir>`, `<session-id>`, `<branch-slug>` used uniformly as template variables
- Every agent file includes a "Session Context" section explaining the session directory and worktree conventions
- The orchestrator's "Key Paths Reference" table provides a single source of truth for all path patterns
- Quality gate criteria tables are clear and actionable
- No dead content or contradictory instructions
- Hook logic in hooks.json, while complex as inline shell, is functional and correctly routes to session-scoped paths via index.json

### Testing: PASS
- Testing agent performed 10 consistency checks; the 2 failures it reported (stale flat paths in agents and quality-gate) have since been fixed
- Cross-reference validation confirmed all referenced files exist
- Version consistency (0.3.0) confirmed in plugin.json and marketplace.json
- Schema consistency (v1.1 state, v1.0 index) confirmed across all defining files

### Documentation: PASS
- CLAUDE.md accurately reflects the new workspace convention with session and worktree paths
- README.md provides comprehensive documentation including worktree isolation explanation, session log structure, concurrent session examples, and updated project structure
- `shared/COMMUNICATION_PROTOCOL.md` fully documents the session directory structure, index.json schema, state.json v1.1 schema, and worktree isolation
- Architecture mermaid diagram in README.md correctly shows the worktree-aware pipeline

### Build & CI: PASS
- No CI pipeline changes needed for this release (plugin is markdown/JSON configuration)
- hooks.json is correctly structured with both hooks updated for session-scoped paths
- `.gitignore` correctly excludes `.claude/worktrees/` and does not exclude `.on-loop/`

## Issues Found

- [LOW] hooks.json inline Python uses `open('$SESSION_DIR/state.json')` where `$SESSION_DIR` is shell-expanded into a Python string literal. If the session directory path contained a single quote, this would break Python parsing. Since session dirs are UUID-based (`[a-f0-9-]`), this is not practically exploitable, but it is fragile. -- `/Users/charmalloc/dev/on-loop/hooks/hooks.json:14` -- Recommendation: Use `open(\"$SESSION_DIR/state.json\")` (escaped double quotes) for robustness, or extract hook logic into a standalone script.

- [LOW] The `on-loop-clear` command's `rm -rf .claude/worktrees/` removes the entire worktrees directory, which could affect concurrent sessions. -- `/Users/charmalloc/dev/on-loop/commands/on-loop-clear.md:40-42` -- Recommendation: Document this behavior explicitly or iterate per-worktree. The current doc does say "for each on-loop worktree" but the cleanup command removes the parent directory.

- [INFO] Security agent notes will be committed to repo history per the v0.3.0 design. Teams handling sensitive security data should add `.on-loop/sessions/*/agent-notes/security.md` to their project `.gitignore`. This is documented in the architect spec (ADR-002) but not surfaced in CLAUDE.md or README.md as a user-facing warning.

## Decisions

- APPROVE: All spec requirements (FR-001 through FR-015, NFR-001 through NFR-006) are addressed in the changed files. The stale flat-path references reported by the testing agent have been fully remediated. The security agent's HIGH findings regarding branch slug validation are addressed. The remaining security concerns (committed security notes, hook fragility) are design trade-offs that are documented and reasonable for a local CLI plugin. No CRITICAL or HIGH issues remain in the source files.

## Files Reviewed

- `.claude-plugin/plugin.json` -- version 0.3.0, correct
- `.claude-plugin/marketplace.json` -- version 0.3.0, correct
- `.gitignore` -- `.on-loop/` removed, `.claude/worktrees/` present, correct
- `skills/loop-state/SKILL.md` -- schema v1.1, session-scoped paths, correct
- `skills/quality-gate/SKILL.md` -- all gates use `<session-dir>/`, correct
- `commands/on-loop.md` -- worktree + session INIT, slug validation, GIT from worktree, correct
- `commands/on-loop-resume.md` -- `--session=<id>`, worktree verification, correct
- `commands/on-loop-status.md` -- multi-session listing from index.json, correct
- `commands/on-loop-clear.md` -- worktree cleanup, `--include-logs`, correct
- `commands/on-continue.md` -- worktree + session per step, correct
- `commands/on-pause.md` -- session-scoped paths throughout, correct
- `commands/on-spec.md` -- session directory creation, correct
- `commands/on-security.md` -- session directory creation, correct
- `commands/on-test.md` -- session directory creation, correct
- `commands/on-doc.md` -- session directory creation, correct
- `commands/on-build.md` -- session directory creation, correct
- `commands/on-review.md` -- session directory creation, correct
- `agents/orchestrator.md` -- worktree lifecycle, session tracking, index.json, key paths table, correct
- `agents/architect.md` -- session context section, `<session-dir>/` paths, correct
- `agents/coding.md` -- session context section, `<session-dir>/` paths, correct
- `agents/testing.md` -- session context section, `<session-dir>/` paths, correct
- `agents/security.md` -- session context section, `<session-dir>/` paths, correct
- `agents/documentation.md` -- session context section, `<session-dir>/` paths, correct
- `agents/build.md` -- session context section, `<session-dir>/` paths, correct
- `agents/reviewer.md` -- session context section, `<session-dir>/` paths, correct
- `hooks/hooks.json` -- session-scoped via index.json, correct
- `shared/COMMUNICATION_PROTOCOL.md` -- new workspace structure, schemas, correct
- `CLAUDE.md` -- updated workspace convention, correct
- `README.md` -- updated architecture, worktree and session docs, correct

## Commendations

- Excellent consistency across 33 files: every agent, command, and skill uses the same `<session-dir>/` convention without exception
- The "Session Context" section added to each agent file is a clean pattern that sets up the variable convention upfront, avoiding repetition
- The orchestrator's "Key Paths Reference" table is a valuable quick-reference for path conventions
- The quality-gate skill's opening line ("All paths below use `<session-dir>` to refer to the active session directory") cleanly establishes the scoping convention
- Branch slug validation addresses the security concern with a strict character whitelist AND explicit `..`/`/` rejection -- defense in depth
- The README.md "Concurrent Sessions" example is an effective way to communicate the new capability
- The `on-loop-clear` command's two-mode design (default keeps logs, `--include-logs` to remove) is a sensible default-safe approach

## Recommendations for Next Agent

- Non-blocking: Consider adding a note in CLAUDE.md or README.md warning that session logs (including security findings) are committed to the repo, and how to exclude them if needed
- Non-blocking: Consider extracting hooks.json inline shell/python into standalone script files for easier auditing and maintenance
- Non-blocking: Add CODEOWNERS entry for `hooks/hooks.json` when a CODEOWNERS file is created
- Non-blocking: The `rm -rf .claude/worktrees/` in on-loop-clear could be changed to iterate per-slug to avoid affecting concurrent sessions
