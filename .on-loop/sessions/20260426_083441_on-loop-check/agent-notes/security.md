# Security Agent Notes

## Summary

Audited the `/on-loop-check` command file and associated changes (version bumps, CLAUDE.md update) for security vulnerabilities. The deliverable is a markdown instruction file interpreted by Claude Code, not executable code. The security surface is therefore the shell commands that the interpreting LLM will execute based on these instructions.

Overall security posture: **PASS with 2 MEDIUM findings and 1 LOW finding**. No CRITICAL or HIGH issues. The three items flagged by the testing agent have been evaluated and are addressed below.

## OWASP Top 10 Review

### A01: Broken Access Control
- [OK] The command relies on `gh auth status` as a prerequisite gate. Access to GitHub resources is governed by the user's existing `gh` CLI authentication scope. No privilege escalation vectors identified.

### A02: Cryptographic Failures
- [OK] No cryptographic operations performed. No tokens, keys, or secrets are stored or transmitted by the command itself.

### A03: Injection
- [MEDIUM] Branch name injection -- see Finding 1 below. The regex `^[a-zA-Z0-9._/-]+$` is present and applied before shell usage. It is adequate but has a minor gap worth documenting.
- [OK] PR number input validated against `^[0-9]+$` -- sufficient for numeric-only input.
- [OK] No SQL, LDAP, or template injection surfaces exist.

### A04: Insecure Design
- [OK] Single-attempt coding agent dispatch prevents retry loops. Version bump only on full success. Explicit STOP conditions on pending checks, pre-existing failures, and missing PRs.

### A05: Security Misconfiguration
- [OK] No debug flags, default credentials, or unnecessary features exposed.

### A06: Vulnerable Components
- [OK] The command uses `gh` CLI (system-installed). No new dependencies introduced. No package.json or lockfile changes.

### A07: Auth Failures
- [OK] Authentication is delegated to `gh auth status`. The command does not implement its own auth.

### A08: Data Integrity Failures
- [OK] Version bump uses explicit string matching and idempotency checks. JSON files are read-then-write.

### A09: Logging Failures
- [MEDIUM] CI log handling -- see Finding 2 below. The instruction to not persist `--log-failed` output is present, but the mitigation relies on LLM compliance rather than a technical control.

### A10: SSRF
- [OK] No URL construction from user input. All GitHub API calls go through `gh` CLI which handles URL construction internally.

## STRIDE Analysis

### /on-loop-check Command (overall)

| Threat | Risk | Mitigation |
|--------|------|------------|
| Spoofing | LOW | Auth delegated to `gh auth status`; user identity is the local shell user |
| Tampering | LOW | Version bump writes are idempotent and target specific files; git commit provides audit trail |
| Repudiation | LOW | All changes are committed to git with Co-Authored-By attribution |
| Information Disclosure | MEDIUM | CI failed logs may contain secrets; mitigated by instruction to not persist (see Finding 2) |
| Denial of Service | LOW | Single attempt limit on coding agent dispatch prevents infinite loops; pending check detection prevents busy-waiting |
| Elevation of Privilege | LOW | No privilege boundaries crossed; all operations run as the local user with their existing `gh` token scope |

## Findings

### [MEDIUM] Finding 1: Branch Name Regex Allows Characters That Could Cause Shell Parsing Issues

- **Location**: `commands/on-loop-check.md:46` (regex), `commands/on-loop-check.md:50` (usage in `gh pr list --head <branch>`)
- **Description**: The branch validation regex `^[a-zA-Z0-9._/-]+$` correctly blocks most injection characters (semicolons, backticks, pipes, ampersands, dollar signs, quotes, spaces). However, the regex allows `.` and `/` which could theoretically form path traversal sequences like `../`. While this is not exploitable in the `gh pr list --head <branch>` context (the value is passed as a CLI argument to `gh`, not used in file system operations), it is worth noting for defense-in-depth. Additionally, the branch name is not shown to be quoted in the `gh pr list` command template on line 50 (`gh pr list --head <branch>`), though the `gh` CLI handles this safely.
- **Impact**: LOW in practice. The `gh` CLI treats the `--head` value as a Git ref filter, not a shell-interpolated string. No command injection is achievable with the allowed character set. Path traversal via `../` has no effect in this context.
- **Remediation**: No code change required. For defense-in-depth, the instruction could explicitly note that the branch argument should be passed as a quoted shell argument (e.g., `gh pr list --head "$BRANCH"`). The existing regex is sufficient for preventing command injection.
- **Reference**: CWE-78 (OS Command Injection), OWASP A03

### [MEDIUM] Finding 2: CI Log Secret Leakage Relies on Instructional (Not Technical) Control

- **Location**: `commands/on-loop-check.md:145-146` (log retrieval), `commands/on-loop-check.md:272` (Important section warning)
- **Description**: The `gh run view <run-id> --log-failed` command can return CI logs that contain leaked secrets (environment variables, tokens printed by failing tests, etc.). The command instructions say "pass as context, do NOT persist to a file -- logs may contain leaked secrets." This is the correct mitigation for a markdown instruction file, but it is an instructional control, not a technical one. The interpreting LLM could hypothetically write the output to a file. The existing session `changes.log` is append-only and committed to the repo, so any accidental persistence there would be particularly problematic.
- **Impact**: MEDIUM. If CI logs containing secrets were persisted to a file within the git worktree and subsequently committed, those secrets would enter the git history. The instruction clearly states not to do this, and the coding agent dispatch passes logs as inline context only.
- **Remediation**: No code change required for PASS. The instructional control is appropriate for the medium (markdown command files interpreted by an LLM). The warning appears in two places (Step 5a.3 and the Important section), which provides adequate reinforcement. For additional hardening in a future version, consider adding an explicit instruction like: "After the coding agent completes, verify that no files matching `*.log` or containing CI output were created in the worktree before running `git add`."
- **Reference**: CWE-532 (Insertion of Sensitive Information into Log File), OWASP A09

### [LOW] Finding 3: `git add -A` in Regression Fix Could Stage Unintended Files

- **Location**: `commands/on-loop-check.md:151` (`git add -A`)
- **Description**: The regression fix commit uses `git add -A` which stages all changes in the worktree, including untracked files. If the coding agent creates temporary files, debug output, or if other untracked files exist in the worktree, they would be committed. This could inadvertently commit sensitive files (`.env`, credentials, debug dumps) or unrelated changes.
- **Impact**: LOW. The worktree is a controlled environment created by the on-loop orchestrator. The coding agent is instructed to fix only the regression. The risk of stray sensitive files appearing in a fresh worktree is low but non-zero.
- **Remediation**: Replace `git add -A` with explicit file staging. The command instructions should direct the coding agent to report which files it modified, then stage only those files. For example:
  ```bash
  git add <specific-files-modified-by-coding-agent>
  git commit -m "Fix CI regression: <brief description>"
  ```
  Alternatively, add an instruction to run `git diff --stat` before `git add -A` and verify only expected files are changed.
- **Reference**: CWE-219 (Storage of File with Sensitive Data Under Web Root -- analogous: committing sensitive data), OWASP A05

## Compliance Notes

- **SOC2**: Access controls delegated to `gh` CLI auth. Audit logging via git commits with Co-Authored-By attribution. No gaps identified.
- **PCI-DSS**: Not applicable -- no cardholder data handled.
- **NIST 800-53**: Input validation (SI-10) addressed via branch regex. Least privilege (AC-6) maintained by using existing user credentials only. Audit trail (AU-3) via git history.
- **GDPR**: Not applicable -- no PII processed by this command.

## Dependency Audit

No new dependencies introduced. The command uses only:
- `gh` CLI (system-installed, user-managed)
- `git` (system-installed)

No package.json, lockfile, or dependency manifest changes in this deliverable.

## Decisions

- The branch validation regex `^[a-zA-Z0-9._/-]+$` is **sufficient** for preventing command injection. The allowed character set cannot produce shell metacharacters. Documented as MEDIUM for awareness, not for remediation.
- The instructional control against persisting CI logs is **adequate** for the medium (LLM-interpreted markdown). Technical enforcement is not possible in a markdown instruction file.
- The `git add -A` issue is **LOW risk** in the controlled worktree context but should be addressed in a future iteration for defense-in-depth.
- **Security gate: PASS** -- No CRITICAL or HIGH findings.

## Recommendations for Next Agent

- Documentation agent should note the security considerations (branch validation, CI log handling, explicit file staging) in any user-facing documentation for `/on-loop-check`
- Build agent does not need to add security scanning for this deliverable (no executable code introduced)
- If a future version adds executable code (e.g., a shell script wrapper), re-audit for command injection at that time
