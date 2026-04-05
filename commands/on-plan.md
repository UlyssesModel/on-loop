---
name: on-plan
description: Read a roadmap and produce a detailed implementation plan with parallelism annotations and branch setup
user_invocable: true
argument: feature slug (optional -- auto-detects if only one roadmap exists)
---

# /on-plan

Reads a roadmap document and state file, identifies the current/next phase, analyzes the codebase, and produces a detailed implementation plan with parallelism annotations and file-level assignments.

## Usage

```
/on-plan                     # auto-detect if only one roadmap exists
/on-plan <feature-slug>      # specify which roadmap to plan
```

## Instructions

When this command is invoked:

### 1. Identify the Roadmap

If a feature slug is provided:
- Read `roadmap/.state/<feature-slug>.json`
- Read `roadmap/<feature-slug>.md`

If no slug is provided:
- List all `.json` files in `roadmap/.state/` (excluding `_global.json`)
- If exactly one exists, use it
- If multiple exist, list them and ask the user to specify
- If none exist, suggest running `/on-prepare` first

### 2. Read Current State

From the state file:
- Identify `current_phase`
- Check which phases are `complete`, `in-progress`, or `not-started`
- Check which steps in the current phase are done vs remaining

### 3. Analyze the Codebase

Before planning:
- Read the project's `CLAUDE.md`, `README.md`, and documentation
- Use Glob to survey the directory structure
- Read source files relevant to the current phase's work
- Understand existing patterns, conventions, and architecture

### 4. Produce the Detailed Plan

For the current/next phase, enhance each step with:

#### Step Detail

For each step in the current phase:
1. **Detailed description**: What exactly needs to be done (implementation-level specificity)
2. **Files to create**: Full paths of new files with a one-line description of each
3. **Files to modify**: Full paths of existing files with what changes are needed
4. **Dependencies**: Other steps this depends on (beyond what `conflicts_with` captures)
5. **Parallelism**: Confirm or refine the `parallel`/`exclusive`/`conflicts_with` annotations
6. **Estimated scope**: Small (< 30 min), Medium (30-60 min), or Large (60+ min, consider splitting)
7. **Agent focus**: Which agents in the pipeline are most important for this step (e.g., "security-heavy" or "test-heavy")

#### Update the Roadmap

Write the detailed plan back into `roadmap/<feature-slug>.md` by adding detail under each step. Do NOT remove existing content -- augment it.

### 5. Create Feature Branch

If a feature branch does not already exist for the current phase:

1. Check current git branch
2. Create branch: `git checkout -b feature/<slug>-phase-<N>`
3. Update the state file with the branch name for the current phase

If the branch already exists, switch to it.

### 6. Update State

Update `roadmap/.state/<feature-slug>.json`:
- Refine step `files` arrays with the exact file paths identified
- Refine parallelism annotations if analysis reveals new constraints
- Set `updated_at` to current timestamp

### 7. Report

Display to the user:
- Current phase and its title
- Steps ready to work on (with parallelism notes)
- Steps blocked (with reason)
- Steps already complete
- Suggested order of execution
- Branch created/checked out
- Suggest running `/on-continue` to start working

## Planning Guidelines

- **Right-size steps**: Each step should be completable in one `/on-continue` run. If a step looks too large, suggest splitting it in the roadmap.
- **File conflicts matter**: Two steps that modify the same file cannot safely run in parallel. Tag them with `conflicts_with` or shared file references.
- **Exclusive steps**: Steps that modify shared configuration (e.g., `CLAUDE.md`, `package.json`, CI configs) should be `exclusive: true` or have explicit `conflicts_with` entries.
- **Order suggestions**: Even for parallel steps, suggest an optimal order for a single session.
- **Dependencies**: If step B requires step A's output to exist, note it even if they are in the same phase.

## Important

- This command does NOT execute any steps. It only plans.
- The plan is written into the roadmap document (committed to repo).
- If the current phase is already complete, automatically advance to the next phase.
- If all phases are complete, report that the feature is done.
