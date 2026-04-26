# Security Agent Notes

## Summary

Audited `/on-loop-debug-fix` (new command) and `/on-loop-check` (modified polling behavior) for security vulnerabilities. The deliverables are markdown instruction files interpreted by Claude, not executable code. The security surface area is significant due to log ingestion of potentially secret-containing data, .env handling, docker/kubectl operations, and untrusted input processing. Found one HIGH finding (incomplete log redaction) and several MEDIUM findings. No CRITICAL findings.

## OWASP Top 10 Review

### A01: Broken Access Control
- [OK] The command reuses existing session context and does not create new access paths. kubectl uses current context only. No privilege escalation vectors identified.

### A02: Cryptographic Failures
- [HIGH] Log redaction regex is incomplete. See Finding HIGH-001 below.

### A03: Injection
- [MEDIUM] The command instructs "Never interpolate log content into shell commands without proper quoting" (line 396 of on-loop-debug-fix.md) but does not specify HOW to quote. The kubectl label selector in Step 3b uses `APP_NAME=$(basename "$REPO_ROOT")` which is safe for typical repo names but could be problematic if the repo root directory name contains shell metacharacters. See Finding MEDIUM-001.
- [OK] Docker compose commands use `--env-file` with a variable, not string interpolation of log content.

### A04: Insecure Design
- [MEDIUM] No rate limiting on docker restart operations. A malicious or pathological error loop could cause rapid container restarts. See Finding MEDIUM-002.

### A05: Security Misconfiguration
- [OK] .env handling is well-designed: never copied, never symlinked, always referenced via --env-file or source from repo root.
- [OK] The command explicitly states never to persist raw logs.

### A06: Vulnerable Components
- [N/A] No new dependencies introduced. These are markdown instruction files.

### A07: Auth Failures
- [OK] No authentication mechanisms introduced. Relies on existing gh, kubectl, docker auth.

### A08: Data Integrity Failures
- [OK] Commit messages include source attribution. Git history provides integrity trail.

### A09: Logging Failures
- [MEDIUM] `source "$REPO_ROOT/.env"` in Steps 3d and 3e exports ALL .env variables into the shell environment. If the Claude session logs shell commands or environment state, secrets from .env could leak into conversation context. See Finding MEDIUM-003.

### A10: SSRF
- [MEDIUM] Thanos and Loki endpoints are read from .env (THANOS_QUERY_URL, LOKI_URL). If an attacker can control these environment variables, they could point queries to arbitrary endpoints. Low risk in practice since .env is developer-controlled. No additional mitigation needed beyond the existing trust model.

## STRIDE Analysis

### /on-loop-debug-fix (Log Discovery Mode)

| Threat | Risk | Mitigation |
|--------|------|------------|
| Spoofing | LOW | Relies on existing kubectl/docker auth; no new auth surfaces |
| Tampering | LOW | Git commit integrity; no data stores modified |
| Repudiation | LOW | Commit messages include source and complexity; git history is audit trail |
| Information Disclosure | HIGH | Incomplete log redaction allows secrets to pass through to agent context (HIGH-001) |
| Denial of Service | MEDIUM | No limit on docker restart cycles (MEDIUM-002) |
| Elevation of Privilege | LOW | Uses current kubectl context; no privilege changes |

### /on-loop-debug-fix (Prompt Mode)

| Threat | Risk | Mitigation |
|--------|------|------------|
| Spoofing | LOW | User input is from the invoking user |
| Tampering | LOW | No data modification outside git commits |
| Repudiation | LOW | Commit trail |
| Information Disclosure | LOW | User-provided text, no log ingestion by default |
| Denial of Service | LOW | Single agent dispatch for trivial; bounded by complexity tier |
| Elevation of Privilege | LOW | No privilege changes |

### /on-loop-check (Polling Loop)

| Threat | Risk | Mitigation |
|--------|------|------------|
| Spoofing | LOW | Uses gh CLI auth |
| Tampering | LOW | No data modification in polling path |
| Repudiation | LOW | Reports status to user |
| Information Disclosure | MEDIUM | Failed test logs may contain secrets; instruction says not to persist, but relies on agent compliance |
| Denial of Service | LOW | Bounded at 20 polls / 10 minutes; timeout exits cleanly |
| Elevation of Privilege | LOW | No privilege changes |

## Findings

### [HIGH-001] Incomplete Log Redaction Regex — Secret Leakage Risk

- **Location**: `commands/on-loop-debug-fix.md:169-180` (Step 5: Log Redaction)
- **Description**: The redaction regex covers three patterns: (1) keyword=value pairs for password/secret/token/key/auth/credential, (2) Bearer tokens, (3) Basic auth headers. However, it misses several common secret formats that frequently appear in infrastructure logs:
  - **JWT tokens**: `eyJ...` base64-encoded tokens that do not follow the `key=value` pattern. JWTs appear in logs as bare strings (e.g., in URL query parameters, request headers logged without the `Bearer` prefix, or error messages like `invalid token: eyJhbG...`).
  - **Connection strings**: `postgres://user:password@host/db`, `mongodb://user:pass@host`, `redis://:password@host`, `amqp://user:pass@host`. These contain embedded credentials not matching keyword=value patterns.
  - **AWS-style keys**: `AKIA[0-9A-Z]{16}` (access key IDs) and long base64 secret keys.
  - **Private keys / PEM blocks**: `-----BEGIN (RSA |EC |)PRIVATE KEY-----` blocks that may appear in misconfiguration error messages.
  - **Hex/base64 API keys**: Many services use bare hex or base64 strings as API keys (e.g., Stripe `sk_live_...`, GitHub `ghp_...`, `ghs_...`).
  - **Query parameters with tokens**: URLs like `?token=abc123&api_key=xyz` where the token is in a URL query string, not a `key=value` shell-style assignment.
- **Impact**: Secrets present in infrastructure logs in these formats would pass through redaction unmodified and be included in agent context. Since agent context may be logged, cached, or transmitted, this creates a secret exposure risk.
- **Remediation**: Add the following redaction patterns to Step 5:
  - JWT: `eyJ[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}` -> `[REDACTED-JWT]`
  - Connection strings: `(?i)(postgres|mysql|mongodb|redis|amqp|mssql)://[^\s]+` -> `[REDACTED-CONNECTION-STRING]`
  - AWS access keys: `AKIA[0-9A-Z]{16}` -> `[REDACTED-AWS-KEY]`
  - PEM blocks: `-----BEGIN[^-]*PRIVATE KEY-----[\s\S]*?-----END[^-]*PRIVATE KEY-----` -> `[REDACTED-PRIVATE-KEY]`
  - Known vendor prefixes: `(sk_live_|sk_test_|ghp_|ghs_|gho_|github_pat_|xoxb-|xoxp-|SG\.)[A-Za-z0-9_-]+` -> `[REDACTED-API-KEY]`
  - URL query tokens: strip query parameters from logged URLs entirely, or redact `[?&](token|key|secret|password|auth|api_key|access_token|apikey)=[^\s&]+`
- **Reference**: CWE-532 (Insertion of Sensitive Information into Log File), CWE-200 (Exposure of Sensitive Information)

### [MEDIUM-001] Insufficient Command Injection Guidance for Log Content

- **Location**: `commands/on-loop-debug-fix.md:396` and general pattern throughout
- **Description**: The command states "Never interpolate log content into shell commands without proper quoting" but does not define what "proper quoting" means or provide a safe pattern. Claude agents executing these instructions may still construct shell commands using log-derived values (e.g., service names extracted from logs used in `docker compose up -d --build <service>`). If a log line contains a service name with shell metacharacters or newlines, naive interpolation could lead to command injection.
- **Impact**: Low probability but high impact. An attacker who can control log output (e.g., by naming a k8s pod or docker service with shell metacharacters) could potentially inject commands.
- **Remediation**: Add explicit safe patterns:
  - "Always use double-quoted variables: `$VAR`, never unquoted `$VAR` or backtick-interpolated values"
  - "For values derived from log content, validate against `^[a-zA-Z0-9._-]+$` before using in shell commands"
  - "Never use `eval` or `sh -c` with log-derived content"
- **Reference**: CWE-78 (OS Command Injection)

### [MEDIUM-002] Unbounded Docker Restart Cycles

- **Location**: `commands/on-loop-debug-fix.md:271-274` (Step 8b, TRIVIAL dispatch)
- **Description**: After a TRIVIAL fix, the command restarts docker services with `docker compose up -d --build <affected-service>`. There is no guard against repeated restart cycles if the fix does not resolve the issue and the command is invoked again. While each invocation is user-triggered, an automated retry loop (e.g., from the orchestrator during TEST phase, as described in the architecture diagram) could cause rapid container restarts.
- **Impact**: Rapid container restarts can cause data loss (if containers have non-persisted state), service instability, and resource exhaustion on the host.
- **Remediation**: Add a guard: "Before restarting a service, check if it was restarted by this command in the last 60 seconds (via a timestamp marker). If so, skip the restart and report that the service was recently restarted."
- **Reference**: CWE-400 (Uncontrolled Resource Consumption)

### [MEDIUM-003] .env Sourcing Exposes All Secrets to Shell Environment

- **Location**: `commands/on-loop-debug-fix.md:106-107` (Step 3d), `commands/on-loop-debug-fix.md:121-122` (Step 3e)
- **Description**: The command uses `set -a; source "$REPO_ROOT/.env"; set +a` to export all .env variables into the shell environment for Thanos and Loki endpoint detection. This exports ALL variables from .env (database passwords, API keys, etc.) into the environment, not just the specific variables needed (THANOS_QUERY_URL, LOKI_URL). If Claude logs the shell environment or if other commands in the same shell session leak environment variables, all .env secrets are exposed.
- **Impact**: Over-broad secret exposure. Only THANOS_QUERY_URL and LOKI_URL are needed, but all .env variables become available.
- **Remediation**: Replace `source "$REPO_ROOT/.env"` with targeted variable extraction:
  ```bash
  THANOS_QUERY_URL=$(grep -E '^THANOS_QUERY_URL=' "$REPO_ROOT/.env" 2>/dev/null | cut -d= -f2-)
  ```
  This reads only the specific variable needed without polluting the environment.
- **Reference**: CWE-522 (Insufficiently Protected Credentials), Principle of Least Privilege

## Compliance Notes

- **SOC2**: Log redaction (HIGH-001) is a gap for SOC2 CC6.1 (logical and physical access controls) -- secrets in agent context could be considered unauthorized information disclosure. The .env handling is well-designed for SOC2 compliance.
- **PCI-DSS**: If any logs contain cardholder data (PAN, CVV), the current redaction regex would not catch them. Recommend adding PAN redaction (`\b[0-9]{13,19}\b` with Luhn validation) if PCI scope is relevant. N/A for this plugin currently.
- **NIST 800-53**: SC-28 (Protection of Information at Rest) -- the "never persist logs" instruction is good. SI-10 (Information Input Validation) -- command injection guidance (MEDIUM-001) is a gap.
- **GDPR**: If logs contain PII (email addresses, names, IP addresses), the current redaction regex would not catch them. Consider adding email redaction (`[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}` -> `[REDACTED-EMAIL]`) and IP address redaction. Relevance depends on deployment context.

## Dependency Audit

- No new runtime dependencies introduced. Both deliverables are markdown instruction files.
- External tool dependencies (docker, kubectl, gh, curl) are assumed pre-installed and are not managed by this plugin.

## Decisions

- The overall security design is sound: .env never copied, logs never persisted, redaction applied before agent processing, docker recommendations are advisory-only.
- The on-loop-check polling loop is well-bounded (20 cycles, 10 minutes max) with clean timeout behavior. No resource exhaustion risk.
- The branch name validation in on-loop-check (`^[a-zA-Z0-9._/-]+$`) is good defense against injection.

## Recommendations for Next Agent

- Documentation agent should document the log redaction patterns and their limitations in user-facing docs
- Documentation agent should note that users with PCI or GDPR requirements may need to add additional redaction patterns
- Build agent should consider adding a pre-commit hook or CI check that scans for common secret patterns in committed files (defense-in-depth for the .env protection)
- The HIGH-001 finding (incomplete redaction) should be remediated before merge -- it is the only finding that could block this PR from a security gate perspective

## Gate Decision

**PASS with conditions**: No CRITICAL findings. One HIGH finding (HIGH-001: incomplete log redaction) has clear remediation steps and is mitigated by the fact that logs are processed in-memory only and not persisted. The existing redaction is a good foundation but needs expansion. The HIGH finding is conditionally passed because: (1) the redaction is defense-in-depth (logs are already not persisted), (2) the remediation is additive (adding patterns, not redesigning), and (3) the risk is bounded to the Claude conversation context. However, the coding agent should add the additional redaction patterns before final merge.
