# Implementation Plan

## Objective

Add git worktree isolation, persistent session logs, and version bump to on-loop v0.3.0.

## Spec Reference

- Worktrees at `.claude/worktrees/<branch-slug>/` for parallel session isolation
- Session logs at `.on-loop/sessions/<session-id>/` committed to repo
- `index.json` manifest at `.on-loop/` root
- state.json schema v1.1 with `session_id`, `worktree_path`, `session_dir` fields
- Version 0.2.0 → 0.3.0

## Tasks

### Phase 1: Version Bump (trivial)
1. Bump `.claude-plugin/plugin.json` version to 0.3.0 — assigned to coding
2. Bump `.claude-plugin/marketplace.json` version to 0.3.0 — assigned to coding

### Phase 2: Gitignore & Schema Changes (foundation)
3. Remove `.on-loop/` line from `.gitignore` — assigned to coding
4. Update `skills/loop-state/SKILL.md` state schema to v1.1 with new fields — assigned to coding

### Phase 3: Core Command Updates
5. Update `commands/on-loop.md` — rewrite INIT (worktree + session dir), GIT (from worktree), COMPLETE (remove worktree), remove .gitignore addition — assigned to coding
6. Update `agents/orchestrator.md` — worktree lifecycle, session tracking, workspace init changes — assigned to coding
7. Update `shared/COMMUNICATION_PROTOCOL.md` — new workspace structure, session layout — assigned to coding

### Phase 4: Supporting Command Updates
8. Update `commands/on-loop-resume.md` — `--session=<id>` arg, session-scoped state — assigned to coding
9. Update `commands/on-loop-status.md` — multi-session listing from index.json — assigned to coding
10. Update `commands/on-loop-clear.md` — worktree cleanup, session log handling — assigned to coding
11. Update `commands/on-continue.md` — worktree + session integration for step execution — assigned to coding

### Phase 5: Hook & Documentation Updates
12. Update `hooks/hooks.json` — session-scoped paths for both hooks — assigned to coding
13. Update `CLAUDE.md` — workspace convention, session/worktree docs — assigned to coding
14. Update `README.md` — architecture docs, new structure — assigned to documentation

## Constraints

- git 2.20+ compatibility
- `.claude/worktrees/` must stay gitignored
- Orchestrator sole writer of state.json
- Session dirs < 100KB typical
- All path references must be updated consistently
