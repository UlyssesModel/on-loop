# Implementation Plan

## Objective

Create the `/on-loop-check` slash command that checks GitHub CI status, classifies failures as regressions vs pre-existing, auto-fixes regressions, alerts on pre-existing failures, and bumps plugin version to 0.4.0.

## Spec Reference

- Standalone utility command (no dedicated on-loop session) — like `/on-loop-status`
- Regression detection via workflow conclusion comparison between PR branch and main
- Version bump 0.3.0 → 0.4.0 only on full success
- Mixed failures: fix regressions, alert on pre-existing, no version bump
- Requires `gh` CLI authenticated

## Tasks

1. **Create `commands/on-loop-check.md`** — assigned to coding agent
   - Frontmatter: name, description, user_invocable: true, argument
   - Instructions covering: PR resolution, CI status check, failure classification, regression fix dispatch, pre-existing alert, version bump, mixed failure handling
   - Follow the pattern from `commands/on-loop-status.md` and `commands/on-loop-clear.md`
   
2. **Bump `.claude-plugin/plugin.json` version** — assigned to coding agent
   - Change `"version": "0.3.0"` to `"version": "0.4.0"`

3. **Bump `.claude-plugin/marketplace.json` version** — assigned to coding agent
   - Change `"version": "0.3.0"` to `"version": "0.4.0"`

4. **Update `CLAUDE.md`** — assigned to coding agent
   - Add `/on-loop-check` to the Commands section in alphabetical order

## Constraints

- Command file must follow existing frontmatter + markdown pattern exactly
- `gh` CLI prerequisite check must be the first step in the command
- Branch name validation: `^[a-zA-Z0-9._/-]+$`
- Version bump must be idempotent (check before writing)
- Failed test logs must not be persisted to files (security: may contain leaked secrets)
- Regression auto-fix only dispatches once (no retry loops)

## Open Questions

- None — all resolved in spec (push automatically, single fix attempt, no dry-run in v1)
