# On-Loop Plugin

Spec-driven SDLC plugin that orchestrates specialist agents through a full development lifecycle.

## Commands

- `/on-loop <prompt>` — Run full SDLC loop (spec → code → test → security → docs → build → review → git)
- `/on-loop-status` — Check current loop progress
- `/on-loop-resume [--from=phase]` — Resume an interrupted loop
- `/on-loop:clear` — Clean up workspace, switch to main, pull latest
- `/on-loop:main-resolve` — Pull main, merge into current branch, resolve conflicts
- `/on-spec <description>` — Standalone spec generation
- `/on-test <target>` — Standalone test generation
- `/on-security <target>` — Standalone security audit
- `/on-doc <target>` — Standalone documentation generation
- `/on-build <target>` — Standalone build/CI setup
- `/on-review <target>` — Standalone code review

## Architecture

All agents communicate through the `.on-loop/` workspace directory (gitignored):
- `state.json` — Phase tracking (only orchestrator writes)
- `plan.md` — Implementation plan (all agents read)
- `changes.log` — Append-only file modification log
- `agent-notes/<agent>.md` — Structured output per agent

## Agent Roster

| Agent | Role |
|-------|------|
| orchestrator | Pipeline control, quality gates, retry logic |
| architect | Spec generation, ADRs, system design |
| coding | Implementation with security-first practices |
| testing | Unit, integration, and E2E tests |
| security | OWASP/STRIDE audit, compliance checks |
| documentation | READMEs, guides, CLAUDE.md files |
| build | Makefile, CI/CD, lint/security configs |
| reviewer | Final code review gate |

## Quality Standards

All agents operate as Staff Engineers with ISC2 certifications targeting regulated financial environments. See `shared/AGENT_PERSONA.md` for the full persona and `shared/QUALITY_STANDARDS.md` for the quality bar.

## Workspace Convention

The `.on-loop/` directory is ephemeral and gitignored. Never commit its contents. Each `/on-loop` invocation initializes a fresh workspace.
