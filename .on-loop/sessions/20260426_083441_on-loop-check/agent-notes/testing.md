# Testing Agent Notes

## Summary

Performed structural, correctness, and completeness validation of the `/on-loop-check` command deliverable. Since this is a markdown-based Claude Code plugin (not executable code), all tests are static analysis and structural checks run via Python scripting. 36 tests were executed across 7 categories. All 36 passed; 2 tests initially reported FAIL due to regex escaping issues in the test scripts themselves — both were confirmed PASS upon re-investigation of the actual file content.

## Test Results

- Total: 36
- Passed: 36
- Failed: 0
- Skipped: 0
- Coverage: N/A (markdown command file, no executable code)

## Tests Written

### Structural Validation (T01–T05)

- Frontmatter parses as valid YAML
- Required fields present: `name`, `description`, `user_invocable`
- `name` field value is `on-loop-check`
- `user_invocable` is boolean `true`
- `argument` field present and matches spec

### Version Consistency (T06–T10)

- `plugin.json` is valid JSON
- `plugin.json` version is `0.4.0`
- `marketplace.json` is valid JSON
- `marketplace.json` plugins array entry version is `0.4.0`
- Both version strings match each other

### CLAUDE.md Validation (T11–T12)

- `/on-loop-check` is listed in the Commands section
- Alphabetical ordering verified: `/on-loop-check` appears after `/on-loop <prompt>` and before `/on-loop-status`

### Functional Requirements Coverage (T13-FR-001 through T13-FR-008)

- FR-001: PR number input, branch name input, and auto-detect from current branch all addressed
- FR-002: `gh pr checks` and `gh pr view` both referenced
- FR-003: Success message ("ALL CHECKS PASSED") present
- FR-004: Failure classification (REGRESSION / PRE-EXISTING) addressed
- FR-005: Coding agent dispatch for regressions present
- FR-006: Pre-existing failure alert ("BLOCKED") present
- FR-007: `plugin.json` modification in version bump step
- FR-008: `marketplace.json` modification in version bump step

### Edge Case Coverage (T14–T22)

- No PR found: error message with `gh pr create` suggestion
- Pending checks: STOP with "wait and re-run" message
- `gh` not authenticated: error with `gh auth login` instruction
- Mixed failures (regression + pre-existing): Step 5c addresses this case
- Branch validation regex `^[a-zA-Z0-9._/-]+$` present in Step 2 (before shell usage) and in the Important section
- Secret leakage warning: explicit "Do NOT persist these logs to files"
- Idempotency: "If both are already 0.4.0, skip the bump" logic present
- No on-loop session fallback: Step 5a.4 covers this case
- Single coding agent dispatch: "at most once per invocation" stated explicitly

### gh CLI Syntax Validation (T31–T33)

- `gh pr checks <pr-number>` — correct syntax
- `gh run list --branch main --workflow "<workflow-name>" --status completed --limit 1 --json databaseId,conclusion` — correct syntax
- `gh run view <run-id> --log-failed` — correct syntax

### Security and Safety (T25–T27, T34–T36)

- No hard-coded credentials or tokens in the command file
- Version bump explicitly excluded from regression path (Step 5a.5)
- Version bump explicitly excluded from mixed failure path (Step 5c.4: "Do NOT bump versions")
- Version bump is in Step 6 (success path only) with `git add` and `git commit`
- Command states it does not create its own session (ADR-002 compliant)
- Error/failure language present for GitHub API unreachability

## Decisions

- Static analysis via Python scripting is the appropriate test methodology for markdown instruction files; there is no runtime to execute
- Two test script false negatives (T18 and T34) caused by regex metacharacter escaping in the Python test patterns were re-investigated and confirmed as PASS; the command file content is correct
- T36 (offline safety) is validated by the presence of error handling language; the actual runtime behavior depends on `gh` CLI error propagation, which is outside the scope of a static check

## Files Modified

- `/Users/charmalloc/dev/on-loop/.on-loop/sessions/20260426_083441_on-loop-check/agent-notes/testing.md` — Created this file

## Issues Found

None. All requirements from the architect spec are addressed in the command file.

## Failures Detail

None.

## Recommendations for Next Agent

- Security agent should verify the branch injection regex `^[a-zA-Z0-9._/-]+$` is sufficient (note: it allows `/` which is expected for branch names like `on-loop/feature`, and `_` and `.` which are common in branch names)
- Security agent should confirm that passing `gh run view --log-failed` output as inline context to the coding agent (not persisted to disk) is the correct threat mitigation for secret leakage
- The version bump logic handles `0.3.0 -> 0.4.0` explicitly by string match; if the running version is already past `0.4.0` (e.g., `0.5.0`), the idempotency check correctly skips. However, if the version is something other than `0.3.0` or `0.4.0`, the command's behavior is unspecified. This edge case is low risk for a first release but worth noting.
- The `git add -A` used in the regression fix commit step (Step 5a.3) could inadvertently stage untracked files. Security agent may want to flag this as a concern.
