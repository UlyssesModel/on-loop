---
name: on-test
description: Generate and run tests for the specified target using the testing agent
user_invocable: true
argument: target file, directory, or description
---

# /on-test

Standalone test generation using the testing agent.

## Usage

```
/on-test <file or directory to test>
/on-test <description of what to test>
```

## Instructions

1. Read the user's target argument.

2. Generate a session name: `YYYYMMDD_HHMMSS_test-<slugified-target>` (e.g., `20260426_143052_test-src-api-handlers`)

3. Create a session directory `.on-loop/sessions/<session-name>/`:
   - `state.json` with phase `"TEST"` and the target as prompt
   - Empty `agent-notes/` directory
   - Update `.on-loop/index.json` (create if missing)

4. If the target is a file or directory path:
   - Read the source code to understand what needs testing
   - Identify the testing framework already in use (if any)

5. Dispatch the **testing agent** (`agents/testing.md`):
   - Provide the target and source code context
   - Provide existing test patterns in the project for consistency

6. When complete, update `.on-loop/index.json` session status to `"complete"`, then:
   - Display test results (pass/fail counts)
   - Display any issues found
   - Show the test files created

## Notes

- This command runs only the testing agent
- Tests follow the project's existing test conventions
- Both happy path and error path tests are generated
- The testing agent will also run the tests and report results
