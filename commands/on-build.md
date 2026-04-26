---
name: on-build
description: Set up build tooling and CI/CD using the build agent
user_invocable: true
argument: target description or scope
---

# /on-build

Standalone build/CI setup using the build agent.

## Usage

```
/on-build <description of what to set up>
/on-build                                # set up build for entire project
```

## Instructions

1. Read the user's target argument. If no argument, set up build infrastructure for the entire project.

2. Generate a session name: `YYYYMMDD_HHMMSS_build-<slugified-target>` (e.g., `20260426_143052_build-entire-project`)

3. Create a session directory `.on-loop/sessions/<session-name>/`:
   - `state.json` with phase `"BUILD"` and the target as prompt
   - Empty `agent-notes/` directory
   - Update `.on-loop/index.json` (create if missing)

4. Survey the project:
   - Identify language(s) and framework(s)
   - Check for existing build configuration
   - Identify testing framework
   - Check for existing CI/CD configuration

5. Dispatch the **build agent** (`agents/build.md`):
   - Provide the target scope and project context

6. When complete, update `.on-loop/index.json` session status to `"complete"` and display:
   - Summary of build infrastructure created
   - Makefile targets available
   - CI pipeline configuration
   - Security scanning setup

## Notes

- This command runs only the build agent
- It creates Makefile, GitHub Actions CI, linter configs, and security scanning
- Existing configuration is updated, not overwritten
- The build agent follows security-first CI practices
