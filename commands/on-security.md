---
name: on-security
description: Run a security audit on the specified target using the security agent
user_invocable: true
argument: target file, directory, or description
---

# /on-security

Standalone security audit using the security agent.

## Usage

```
/on-security <file or directory to audit>
/on-security <description of what to review>
/on-security                              # audit entire project
```

## Instructions

1. Read the user's target argument. If no argument, audit the entire project.

2. Generate a session name: `YYYYMMDD_HHMMSS_security-<slugified-target>` (e.g., `20260426_143052_security-src-auth`)

3. Create a session directory `.on-loop/sessions/<session-name>/`:
   - `state.json` with phase `"SECURITY"` and the target as prompt
   - Empty `agent-notes/` directory
   - Update `.on-loop/index.json` (create if missing)

4. Dispatch the **security agent** (`agents/security.md`):
   - Provide the target scope
   - Provide the session directory path
   - Provide project context (dependencies, configuration, architecture)

5. When complete, display:
   - Summary of security posture
   - OWASP Top 10 findings (if any)
   - STRIDE analysis results
   - All findings sorted by severity (CRITICAL -> LOW)
   - Compliance notes
   - Dependency audit results

6. Update `.on-loop/index.json` session status to `"complete"`.

7. The findings are left in the session's `agent-notes/security.md`.

## Notes

- This command runs only the security agent — it does NOT modify code
- The security agent reviews but never fixes code
- Use the findings to guide manual remediation or feed into `/on-loop`
- Covers OWASP Top 10, STRIDE threat modeling, and compliance checks
