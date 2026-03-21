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

2. Create a minimal `.on-loop/` workspace if one doesn't exist:
   - `state.json` with phase `"TEST"` and the target as prompt
   - Empty `agent-notes/` directory

3. If the target is a file or directory path:
   - Read the source code to understand what needs testing
   - Identify the testing framework already in use (if any)

4. Dispatch the **testing agent** (`agents/testing.md`):
   - Provide the target and source code context
   - Provide existing test patterns in the project for consistency

5. When complete:
   - Display test results (pass/fail counts)
   - Display any issues found
   - Show the test files created

## Notes

- This command runs only the testing agent
- Tests follow the project's existing test conventions
- Both happy path and error path tests are generated
- The testing agent will also run the tests and report results
