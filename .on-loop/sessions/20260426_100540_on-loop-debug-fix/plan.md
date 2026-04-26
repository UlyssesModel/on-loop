# Implementation Plan

## Objective

Create `/on-loop-debug-fix` slash command with dual-mode debugging (log discovery + user prompt), complexity-gated orchestration, worktree-aware .env handling, and docker optimization. Also update `/on-loop-check` to poll and wait for CI completion instead of telling users to re-run. Bump plugin version to 0.5.0.

## Spec Reference

- Complexity-gated orchestration: TRIVIAL (coding only) / MODERATE (coding+test) / COMPLEX (full pipeline) / USER_ACTION (instructions)
- No new worktree, reuse active session if available
- .env via `--env-file <repo-root>/.env`, never copy
- Log redaction before processing
- Auto-detect log sources (docker, kubectl, MCP, Thanos, Loki)
- Docker optimization as advisory output
- On-loop-check: add polling loop for pending CI checks

## Tasks

1. **Create `commands/on-loop-debug-fix.md`** — assigned to coding agent
   - Frontmatter with name, description, user_invocable, argument
   - Full instructions covering both modes, complexity assessment, agent dispatch tiers, .env resolution, log ingestion, user-action detection, docker recommendations
   - Follow pattern from on-loop-check.md

2. **Update `commands/on-loop-check.md`** — assigned to coding agent
   - Replace the "STOP and tell user to wait" behavior for pending checks
   - Add a polling loop: check every 30 seconds, report progress, timeout after 10 minutes
   - When checks complete, proceed with the normal pass/fail flow

3. **Bump `.claude-plugin/plugin.json`** — 0.4.0 → 0.5.0

4. **Bump `.claude-plugin/marketplace.json`** — 0.4.0 → 0.5.0

5. **Update `CLAUDE.md`** — add `/on-loop-debug-fix` to Commands section

6. **Update `README.md`** — add `/on-loop-debug-fix` to command table, update version

## Constraints

- Command file follows existing frontmatter + markdown pattern
- .env must never be copied/symlinked into worktree
- Raw logs never persisted to files
- Agent dispatches use existing agent definitions (no new agents)
- on-loop-check polling must have a timeout to prevent infinite loops
- Version bump must be idempotent
