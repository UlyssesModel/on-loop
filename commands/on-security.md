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

2. Create a minimal `.on-loop/` workspace if one doesn't exist:
   - `state.json` with phase `"SECURITY"` and the target as prompt
   - Empty `agent-notes/` directory

3. Dispatch the **security agent** (`agents/security.md`):
   - Provide the target scope
   - Provide project context (dependencies, configuration, architecture)

4. When complete, display:
   - Summary of security posture
   - OWASP Top 10 findings (if any)
   - STRIDE analysis results
   - All findings sorted by severity (CRITICAL → LOW)
   - Compliance notes
   - Dependency audit results

5. The findings are left in `.on-loop/agent-notes/security.md`.

## Notes

- This command runs only the security agent — it does NOT modify code
- The security agent reviews but never fixes code
- Use the findings to guide manual remediation or feed into `/on-loop`
- Covers OWASP Top 10, STRIDE threat modeling, and compliance checks
