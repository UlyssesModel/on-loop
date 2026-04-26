# Security Agent Notes

## Summary

The v0.3.0 changes introduce git worktree isolation, persistent session logs committed to the repository, and shell hooks for change tracking. The codebase is an instruction/configuration plugin (markdown and JSON), not executable application code -- so the attack surface is indirect: it prescribes shell commands that Claude Code agents will execute, and it determines what data gets committed to git history. The primary concerns are: (1) shell injection vectors in hooks.json, (2) sensitive data committed in session logs, (3) path traversal via unsanitized branch slugs, and (4) destructive force-flag usage in cleanup commands.

## OWASP Top 10 Review

### A01: Broken Access Control
- No issues found. This is a local CLI plugin with no network-accessible endpoints or multi-user access model.

### A02: Cryptographic Failures
- No issues found. No cryptographic operations are performed.

### A03: Injection
- [HIGH] Shell injection via hooks.json -- see Finding 1 below.
- [MEDIUM] Branch slug injection into shell commands -- see Finding 2 below.

### A04: Insecure Design
- [HIGH] Persistent session logs committed to repo with security findings -- see Finding 3 below.

### A05: Security Misconfiguration
- [MEDIUM] Force flags in cleanup commands -- see Finding 4 below.

### A06: Vulnerable Components
- No issues found. No third-party dependencies; the plugin only requires git, python3, and gh CLI.

### A07: Auth Failures
- Not applicable. No authentication system.

### A08: Data Integrity Failures
- [LOW] No integrity validation on index.json or state.json. A corrupted or tampered index.json could cause the orchestrator to misroute sessions.

### A09: Logging Failures
- [HIGH] Security agent notes (which may contain vulnerability details, exploit paths, and remediation guidance) are committed to the repository as part of the session audit log -- see Finding 3.

### A10: SSRF
- Not applicable. No URL fetching operations.

## STRIDE Analysis

### Git Worktree Lifecycle
| Threat | Risk | Mitigation |
|--------|------|------------|
| Spoofing | LOW | Worktrees are local; no remote identity involved |
| Tampering | MEDIUM | No integrity checks on session state files; any process with filesystem access can modify state.json or index.json to influence orchestrator behavior |
| Repudiation | LOW | changes.log provides an audit trail of file operations |
| Information Disclosure | HIGH | Session logs including security.md are committed to repo -- see Finding 3 |
| Denial of Service | LOW | Orphaned worktrees could consume disk; mitigated by /on-loop:clear |
| Elevation of Privilege | LOW | No privilege boundaries exist in this system |

### hooks.json Shell Commands
| Threat | Risk | Mitigation |
|--------|------|------------|
| Spoofing | LOW | Hooks run locally under user's identity |
| Tampering | MEDIUM | Hook commands are defined in committed JSON; modifying hooks.json in a PR could inject malicious commands that run on all future sessions |
| Repudiation | LOW | Hook output is not logged |
| Information Disclosure | LOW | Hook reads index.json and state.json; no secrets involved |
| Denial of Service | LOW | Malformed index.json could cause python3 to error, but hook failures are non-fatal |
| Elevation of Privilege | MEDIUM | hooks.json commands execute arbitrary shell/python -- a supply chain attack modifying this file would get code execution in the user's shell context |

## Findings

### [HIGH] Finding 1: Shell Injection via Unsanitized Variables in hooks.json

- **Location**: `/Users/charmalloc/dev/on-loop/hooks/hooks.json:14`
- **Description**: The `on-loop-change-tracker` hook interpolates `$CLAUDE_FILE_PATH` directly into a shell command that appends to `changes.log`. The variable `$SESSION_DIR` (derived from python3 parsing of index.json) is also used unquoted in the file-existence test and the echo/append command. If `CLAUDE_FILE_PATH` or the session directory path contains shell metacharacters (backticks, `$()`, semicolons, pipes, etc.), they will be interpreted by the shell.

  Specific vulnerable patterns in the hook:
  ```
  if [ -f "$SESSION_DIR/changes.log" ]   # SESSION_DIR from python, used in test
  echo "..." >> "$SESSION_DIR/changes.log"  # append with interpolated values
  ```
  
  While `SESSION_DIR` is a UUID-based path (low risk), `$CLAUDE_FILE_PATH` is an environment variable set by Claude Code reflecting the file the agent just wrote -- if an agent writes to a path containing metacharacters (e.g., a file with a backtick in its name), the shell will execute the embedded command.

  Additionally, the python3 one-liner on line 7 uses `open('.on-loop/index.json')` with unquoted f-string interpolation of session data, which could be exploited if session metadata (like branch name) contains Python injection payloads.

- **Impact**: Arbitrary command execution in the user's shell context during hook execution. The risk is mitigated by the fact that file paths are typically controlled by the coding agent (trusted), but the attack surface exists if a malicious plan.md or prompt manipulates the agent into writing to a crafted file path.

- **Remediation**:
  1. Quote all variable expansions in shell commands (already done for `$SESSION_DIR` in some places but not consistently).
  2. Sanitize `$CLAUDE_FILE_PATH` before interpolation -- strip or reject paths containing shell metacharacters.
  3. Consider writing the hook logic as a standalone script file rather than inline shell in JSON, which would be easier to audit and maintain.
  4. Use `printf '%s\n'` instead of `echo` for appending to changes.log to avoid echo interpretation issues.

- **Reference**: CWE-78 (OS Command Injection), OWASP A03:2021

### [HIGH] Finding 2: Path Traversal and Shell Injection via Branch Slug

- **Location**: `/Users/charmalloc/dev/on-loop/agents/orchestrator.md:81-88`, `/Users/charmalloc/dev/on-loop/commands/on-loop.md:46-49`
- **Description**: The branch slug is derived from the user's prompt via "slugification" (lowercase, replace spaces/special chars with hyphens, truncate to 50 chars). However, the slugification rules are described in natural language without a strict character whitelist. The slug is then interpolated directly into shell commands:
  ```
  git worktree add .claude/worktrees/<branch-slug> on-loop/<branch-slug>
  git worktree remove .claude/worktrees/<slug> --force
  rm -rf .claude/worktrees/<branch-slug>
  ```
  
  If the slugification is incomplete (e.g., does not strip `../`, semicolons, or backticks), an attacker-controlled prompt could produce a slug that:
  - Traverses outside `.claude/worktrees/` (e.g., slug `../../etc` leading to `rm -rf .claude/worktrees/../../etc`)
  - Injects shell commands via metacharacters

  The instruction says "replace spaces/special chars with hyphens" but does not define "special chars" exhaustively. An LLM interpreting this instruction may not catch all dangerous characters.

- **Impact**: Directory traversal could cause `rm -rf` to delete files outside the intended worktree directory. Shell metacharacters in the slug could lead to arbitrary command execution.

- **Remediation**:
  1. Define a strict character whitelist for slugs: `[a-z0-9-]` only.
  2. Add an explicit validation step: `if [[ ! "$BRANCH_SLUG" =~ ^[a-z0-9][a-z0-9-]*$ ]]; then echo "Invalid slug"; exit 1; fi`
  3. Reject slugs containing `..`, `/`, or any character outside the whitelist.
  4. Add this validation as a documented pre-condition in both `orchestrator.md` and `on-loop.md`.

- **Reference**: CWE-22 (Path Traversal), CWE-78 (OS Command Injection)

### [HIGH] Finding 3: Security Findings Committed to Repository History

- **Location**: `/Users/charmalloc/dev/on-loop/agents/orchestrator.md:103,188-189`, `/Users/charmalloc/dev/on-loop/commands/on-loop.md:105-106`, `/Users/charmalloc/dev/on-loop/shared/COMMUNICATION_PROTOCOL.md:1-6`
- **Description**: The v0.3.0 design commits `.on-loop/sessions/<session-id>/` to the repository as a "persistent audit log." This directory includes `agent-notes/security.md`, which contains detailed security vulnerability findings (OWASP analysis, STRIDE threat models, specific file:line locations, exploitation details, and remediation steps).

  Once committed, this security data becomes part of the permanent git history. Even if the session directory is later deleted, the data persists in git history and is visible to anyone with repository access. For open-source repositories or repos with broad contributor access, this means:
  - Detailed vulnerability descriptions are exposed before remediation is confirmed
  - Specific file paths and line numbers of vulnerabilities are documented
  - Exploitation impact analysis is available to potential attackers

  The `.on-loop/` directory is NOT in `.gitignore` (only `.claude/worktrees/` is gitignored). The GIT phase explicitly stages the session directory.

- **Impact**: Information disclosure of security vulnerabilities to anyone with repository access. In the window between commit and remediation (or indefinitely if remediation is incomplete), attackers have a detailed roadmap of exploitable weaknesses. This is particularly concerning for the "retry exhausted" case where security findings are recorded as TODOs and the code ships with known vulnerabilities documented in the same commit.

- **Remediation**:
  1. **Do NOT commit `agent-notes/security.md` to the repository.** Add `.on-loop/sessions/*/agent-notes/security.md` to `.gitignore`.
  2. Alternatively, add all of `.on-loop/` to `.gitignore` and keep session logs as local-only artifacts (reverting the v0.3.0 "persistent session" design for security-sensitive content).
  3. If audit logs must be committed, redact security findings to summary-level only (e.g., "2 HIGH findings identified and remediated" without specifics).
  4. At minimum, the PR body should reference security findings by severity count only, not by detailed description.

- **Reference**: CWE-200 (Exposure of Sensitive Information), OWASP A09:2021, SOC2 CC6.1 (Logical Access Controls)

### [MEDIUM] Finding 4: Destructive Force Flags in Worktree Cleanup

- **Location**: `/Users/charmalloc/dev/on-loop/agents/orchestrator.md:203-204`, `/Users/charmalloc/dev/on-loop/commands/on-loop-clear.md:36-41`
- **Description**: Both COMPLETE phase and `/on-loop:clear` use `git worktree remove --force` as a fallback, followed by `rm -rf .claude/worktrees/` (or the specific slug directory). The `--force` flag bypasses git's safety checks for uncommitted changes, and `rm -rf` provides no confirmation or dry-run option.

  In `/on-loop:clear`, the command `rm -rf .claude/worktrees/` removes the entire worktrees directory, which could affect concurrent sessions if multiple worktrees are active.

- **Impact**: Data loss of uncommitted work in worktrees. If combined with the path traversal issue in Finding 2, the `rm -rf` could affect directories outside the intended scope.

- **Remediation**:
  1. Before `--force` removal, log what uncommitted changes would be lost.
  2. In `/on-loop:clear`, only remove worktrees that match `on-loop/` branch patterns, not the entire directory.
  3. Add a confirmation step or `--force` flag requirement for the clear command itself (it already has `--include-logs` for session deletion).
  4. Ensure `rm -rf` targets are validated to be under `.claude/worktrees/` using `realpath` to resolve symlinks and prevent traversal.

- **Reference**: CWE-459 (Incomplete Cleanup)

### [MEDIUM] Finding 5: Supply Chain Risk via hooks.json Modification

- **Location**: `/Users/charmalloc/dev/on-loop/hooks/hooks.json`
- **Description**: `hooks.json` defines shell commands that execute automatically on Claude Code events (`Stop`, `PostToolUse`). These hooks run with the user's full shell privileges. A malicious pull request modifying `hooks.json` could inject arbitrary code that executes silently on every file write operation (via the `PostToolUse` hook) or on agent stop (via the `Stop` hook).

  There is no integrity verification, signing, or review gate specifically for hook modifications.

- **Impact**: A compromised or malicious contribution modifying `hooks.json` gains persistent code execution in the developer's shell context, triggered automatically by normal Claude Code usage.

- **Remediation**:
  1. Add `hooks/hooks.json` to a CODEOWNERS file requiring explicit security review for modifications.
  2. Document in CLAUDE.md that hooks.json changes require manual security review.
  3. Consider moving hook logic to separate script files that are easier to audit than inline shell in JSON strings.

- **Reference**: CWE-829 (Inclusion of Functionality from Untrusted Control Sphere)

### [LOW] Finding 6: User Prompt Stored in Multiple Locations Without Sanitization

- **Location**: `/Users/charmalloc/dev/on-loop/agents/orchestrator.md:110,148`, `/Users/charmalloc/dev/on-loop/shared/COMMUNICATION_PROTOCOL.md:49`
- **Description**: The user's original prompt is stored verbatim in `state.json` (full text) and `index.json` (first 100 chars). These files are committed to the repository. If the prompt contains sensitive information (API keys, internal URLs, customer names, PII), it becomes part of permanent git history.

- **Impact**: Low risk of sensitive data exposure if users include confidential information in their prompts.

- **Remediation**:
  1. Document that prompts are committed to the repo, so users know not to include sensitive data.
  2. Consider storing only a hash or truncated/sanitized version of the prompt in committed files.

- **Reference**: CWE-532 (Insertion of Sensitive Information into Log File)

## Compliance Notes

- **SOC2 CC6.1 (Logical Access Controls)**: Finding 3 exposes security vulnerability details in committed files accessible to all repo collaborators. This violates the principle of restricting access to security-sensitive information on a need-to-know basis.
- **SOC2 CC7.2 (Monitoring)**: The session log and changes.log provide good audit trail for operations. However, hook output is not logged (no observability into hook execution failures).
- **NIST 800-53 SI-11 (Error Handling)**: Force-flag usage in cleanup (Finding 4) could mask errors. Errors during worktree removal should be logged to the session audit trail.
- **GDPR**: If user prompts contain PII, Finding 6 applies. Prompts are stored in committed JSON files with no data minimization or retention policy.
- **PCI-DSS**: Not directly applicable to this configuration plugin.

## Dependency Audit

No third-party dependencies. The plugin relies only on:
- `git` (system)
- `python3` (system, used for UUID generation and JSON parsing in hooks)
- `gh` (GitHub CLI, used for PR creation)

No lockfiles to verify. No known CVEs applicable.

## Decisions

- Finding 3 (security findings committed to repo) is classified HIGH because it creates an information disclosure channel for vulnerability details. While the intent of "persistent audit logs" is sound for SOC2 compliance, security-specific findings should be excluded or redacted before commit.
- Findings 1 and 2 (shell injection vectors) are classified HIGH because they describe command injection paths, even though exploitation requires specific conditions (malicious file paths or insufficiently sanitized slugs). The risk is elevated because the instructions are interpreted by an LLM, which may not implement sanitization consistently.
- No CRITICAL findings were identified. The attack surface is indirect (instructions interpreted by agents, not directly executable code), which reduces the severity compared to equivalent issues in application code.

## Recommendations for Next Agent

- **Documentation agent**: Document that `.on-loop/sessions/` content is committed to the repo and should not contain sensitive data. Add a warning about prompt content being persisted.
- **Build agent**: Add `hooks/hooks.json` to a CODEOWNERS-equivalent review requirement. Consider adding a CI check that validates branch slugs against `^[a-z0-9][a-z0-9-]*$`.
- **Coding agent**: Implement the slug validation whitelist from Finding 2's remediation. Refactor hooks.json inline shell to standalone scripts. Add `.on-loop/sessions/*/agent-notes/security.md` to `.gitignore`.
