---
name: on-review
description: Run a code review on the specified target using the reviewer agent
user_invocable: true
argument: target file, directory, or PR
---

# /on-review

Standalone code review using the reviewer agent.

## Usage

```
/on-review <file or directory to review>
/on-review <description of what to review>
/on-review                               # review all recent changes
```

## Instructions

1. Read the user's target argument. If no argument, review all uncommitted changes or recent commits.

2. Generate a session name: `YYYYMMDD_HHMMSS_review-<slugified-target>` (e.g., `20260426_143052_review-recent-changes`)

3. Create a session directory `.on-loop/sessions/<session-name>/`:
   - `state.json` with phase `"REVIEW"` and the target as prompt
   - Empty `agent-notes/` directory
   - Update `.on-loop/index.json` (create if missing)

4. Determine the review scope:
   - If a file/directory: review that code
   - If no argument: use `git diff` to find recent changes and review those
   - Read the code to be reviewed

5. Dispatch the **reviewer agent** (`agents/reviewer.md`):
   - Provide the target scope and code
   - Provide project context (conventions, existing patterns)

6. When complete, update `.on-loop/index.json` session status to `"complete"` and display:
   - Verdict (APPROVE or REQUEST_CHANGES)
   - Review checklist results
   - Findings sorted by severity
   - Commendations (things done well)
   - Recommendations

## Notes

- This command runs only the reviewer agent — it does NOT modify code
- The reviewer checks correctness, security, performance, conventions, and test coverage
- It issues APPROVE or REQUEST_CHANGES with detailed feedback
- Use the feedback to guide manual fixes
