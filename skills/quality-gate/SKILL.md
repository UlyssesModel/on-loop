---
name: quality-gate
description: Defines pass/fail criteria for phase transitions in the on-loop pipeline
---

# Quality Gate

This skill defines the quality criteria that must be met before transitioning between phases.

## Gate: SPEC → PLAN

**Check**: Architect agent notes exist and contain a valid specification.

| Criteria | Required |
|----------|----------|
| `.on-loop/agent-notes/architect.md` exists | Yes |
| Specification has functional requirements | Yes |
| Specification has non-functional requirements | Yes |
| Security considerations documented | Yes |
| Architecture diagram present | Recommended |

**On fail**: Cannot proceed. Report to user.

## Gate: PLAN → CODE

**Check**: Plan is written and actionable.

| Criteria | Required |
|----------|----------|
| `.on-loop/plan.md` has content | Yes |
| Tasks are listed with assignments | Yes |
| Constraints documented | Yes |

**On fail**: Orchestrator rewrites plan.

## Gate: CODE → TEST

**Check**: Coding agent completed implementation.

| Criteria | Required |
|----------|----------|
| `.on-loop/agent-notes/coding.md` exists | Yes |
| Files created/modified listed in `changes.log` | Yes |
| No CRITICAL issues self-reported | Yes |
| Code compiles/parses without errors | Yes |

**On fail**: Cannot proceed. Coding agent must resolve.

## Gate: TEST → SECURITY

**Check**: All tests pass.

| Criteria | Required |
|----------|----------|
| `.on-loop/agent-notes/testing.md` exists | Yes |
| All tests pass (zero failures) | Yes |
| Happy path tests exist | Yes |
| Error path tests exist | Yes |

**On fail (retries remain)**: Transition back to CODE with test feedback.
**On fail (retries exhausted)**: Record TODO, proceed to SECURITY.

## Gate: SECURITY → DOC/BUILD

**Check**: No critical security vulnerabilities.

| Criteria | Required |
|----------|----------|
| `.on-loop/agent-notes/security.md` exists | Yes |
| No CRITICAL findings | Yes |
| No unmitigated HIGH findings | Yes |
| OWASP Top 10 review completed | Yes |

**On fail (retries remain)**: Transition back to CODE with security findings.
**On fail (retries exhausted)**: Record TODO, proceed to DOC/BUILD.

## Gate: DOC + BUILD → REVIEW

**Check**: Both documentation and build agents completed.

| Criteria | Required |
|----------|----------|
| `.on-loop/agent-notes/documentation.md` exists | Yes |
| `.on-loop/agent-notes/build.md` exists | Yes |
| README exists or was updated | Recommended |
| CI configuration exists | Recommended |

**On fail**: Warn but proceed to REVIEW (doc/build issues are non-blocking).

## Gate: REVIEW → GIT

**Check**: Reviewer approved the code.

| Criteria | Required |
|----------|----------|
| `.on-loop/agent-notes/reviewer.md` exists | Yes |
| Verdict is `APPROVE` | Yes |
| No CRITICAL issues | Yes |

**On fail (retries remain)**: Transition back to CODE with review feedback.
**On fail (retries exhausted)**: Record TODO, proceed to GIT with warnings.

## Gate: GIT → COMPLETE

**Check**: Git operations completed successfully.

| Criteria | Required |
|----------|----------|
| All changed files staged and committed | Yes |
| Branch pushed to origin | Yes |
| PR created via `gh pr create` | Yes |
| `pr_url` set in `state.json` | Yes |

**On fail**: Report git error to user. The user can retry with `/on-loop-resume --from=GIT`.

## Retry Budget Summary

| Transition | Max Retries | Trigger |
|-----------|-------------|---------|
| TEST → CODE | 3 | Test failures |
| SECURITY → CODE | 2 | CRITICAL/HIGH findings |
| REVIEW → CODE | 2 | REQUEST_CHANGES verdict |

Total maximum code iterations: 1 (initial) + 3 + 2 + 2 = **8**

## Exhausted Retry Behavior

When retries are exhausted:
1. Record all unresolved issues as TODOs in `state.json`
2. Log the decision: `[timestamp] orchestrator DECISION retry_exhausted — <details>`
3. Advance to the next phase
4. The COMPLETE summary will prominently display all TODOs
