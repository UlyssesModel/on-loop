---
name: on-loop-debug-fix
description: Debug and fix issues using infrastructure logs or user-provided context with complexity-gated orchestration
user_invocable: true
argument: "[description text or image] [--complexity=trivial|moderate|complex]"
---

# /on-loop-debug-fix

Debug and fix issues from infrastructure logs or user-provided context. Scales orchestration from a single coding-agent fix up to a full pipeline based on assessed complexity.

## Usage

```
/on-loop-debug-fix                                    # Log Discovery Mode (auto-detect sources)
/on-loop-debug-fix The login form returns 500          # Prompt Mode (text description)
/on-loop-debug-fix <screenshot> Login button broken    # Prompt Mode (image + text)
/on-loop-debug-fix --complexity=complex Auth is broken # Force complexity tier
```

## Instructions

### Step 1: Determine Mode

1. Read the user's argument (if any).
2. Check for the optional `--complexity=<level>` flag. Valid values: `trivial`, `moderate`, `complex`. If present, store the override and strip it from the argument.
3. Determine the mode:
   - **No argument remaining** after stripping flags: **Log Discovery Mode** (Step 3)
   - **Text and/or image argument provided**: **Prompt Mode** (Step 4)

### Step 2: .env Resolution

Before running any docker or infrastructure commands, resolve the `.env` file:

1. Determine the repository root:
   ```bash
   REPO_ROOT=$(git rev-parse --show-toplevel)
   ```

2. Check if `.env` exists:
   ```bash
   test -f "$REPO_ROOT/.env" && echo "found" || echo "not found"
   ```

3. For all `docker compose` commands in this session, ALWAYS use:
   ```bash
   docker compose --env-file "$REPO_ROOT/.env" <subcommand>
   ```

4. For non-docker tools that need environment variables:
   ```bash
   set -a; source "$REPO_ROOT/.env"; set +a
   ```

5. **NEVER** copy or symlink `.env` into the worktree. Always reference it at the repo root via `--env-file` or `source`.

### Step 3: Log Discovery Mode

When no arguments are provided, auto-detect available log sources and ingest errors.

Test each source in order. If a source is not available, skip it and continue to the next. Report which sources were found.

#### 3a. Docker Compose

```bash
docker compose ps 2>/dev/null
```

If exit code is 0 (docker compose is available and has services):
```bash
REPO_ROOT=$(git rev-parse --show-toplevel)
docker compose --env-file "$REPO_ROOT/.env" logs --tail=200 2>&1 | grep -iE "error|fatal|panic|exception|fail|traceback"
```

Store the filtered output as docker log context.

#### 3b. Kubectl

```bash
kubectl cluster-info 2>/dev/null
```

If exit code is 0 (cluster is reachable):
```bash
REPO_ROOT=$(git rev-parse --show-toplevel)
APP_NAME=$(basename "$REPO_ROOT")
kubectl logs --all-containers --since=10m -l "app.kubernetes.io/part-of=$APP_NAME" 2>&1 | grep -iE "error|fatal|panic|exception|fail|traceback"
```

Also check for non-running pods:
```bash
kubectl get pods --no-headers -o custom-columns=":metadata.name,:status.phase" | grep -v Running
```

Store the filtered output as kubectl log context.

#### 3c. MCP Servers

Check if MCP tools for log retrieval are available in the current Claude session. If log-retrieval MCP tools exist, use them to fetch recent error logs.

#### 3d. Thanos

Check for a Thanos query endpoint:
```bash
REPO_ROOT=$(git rev-parse --show-toplevel)
THANOS_QUERY_URL=$(grep -E '^THANOS_QUERY_URL=' "$REPO_ROOT/.env" 2>/dev/null | head -1 | cut -d= -f2-)
echo "${THANOS_QUERY_URL:-not set}"
```

If `THANOS_QUERY_URL` is set, query for error-rate metrics:
- `up == 0` (targets down)
- `rate(http_requests_total{code=~"5.."}[5m]) > 0` (5xx errors)

Store query results as Thanos context.

#### 3e. Loki

Check for a Loki endpoint:
```bash
REPO_ROOT=$(git rev-parse --show-toplevel)
LOKI_URL=$(grep -E '^LOKI_URL=' "$REPO_ROOT/.env" 2>/dev/null | head -1 | cut -d= -f2-)
echo "${LOKI_URL:-not set}"
```

If `LOKI_URL` is set, query for errors:
- LogQL: `{job=~".+"} |= "error" | level="error"` (last 5 minutes)

Store query results as Loki context.

#### 3f. Source Report

After checking all sources, report:
```
Log Discovery Results
=====================
Sources detected:
  [FOUND]     Docker Compose — <N> error lines
  [NOT FOUND] Kubectl
  [NOT FOUND] MCP Servers
  [NOT FOUND] Thanos
  [NOT FOUND] Loki

Total error lines ingested: <N>
```

If NO sources are detected and no errors found, STOP and report:
```
ERROR: No log sources detected and no errors found.
Use prompt mode instead: /on-loop-debug-fix <description of the issue>
```

### Step 4: Prompt Mode

When the user provides text and/or an image:

1. Use the user's text description directly as the debugging context.
2. If an image was provided (screenshot, error dialog, UI bug), analyze it for:
   - Error messages and codes
   - Stack traces
   - UI rendering issues
   - Console output
3. Combine the text and image analysis into a structured error description.
4. Skip log ingestion UNLESS the user's description explicitly references infrastructure issues (e.g., "docker logs show...", "the pod is crashing", "check the server logs").

### Step 5: Log Redaction

**SECURITY-CRITICAL**: Before passing any log content to agents or including it in context:

1. Apply regex-based redaction to ALL ingested log content:
   - Pattern: `(?i)(password|secret|token|key|auth|credential)\s*[:=]\s*\S+` -- replace the value portion with `[REDACTED]`
   - Pattern: `(?i)Bearer\s+\S+` -- replace with `Bearer [REDACTED]`
   - Pattern: `(?i)Basic\s+[A-Za-z0-9+/=]+` -- replace with `Basic [REDACTED]`
   - Pattern: `eyJ[A-Za-z0-9_-]+\.eyJ[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+` -- JWT tokens, replace with `[REDACTED_JWT]`
   - Pattern: `(?i)(postgres|mysql|mongodb|redis|amqp|kafka)://[^\s]+` -- connection strings, replace with `<protocol>://[REDACTED]`
   - Pattern: `AKIA[0-9A-Z]{16}` -- AWS access key IDs, replace with `[REDACTED_AWS_KEY]`
   - Pattern: `-----BEGIN[A-Z ]*PRIVATE KEY-----[\s\S]*?-----END[A-Z ]*PRIVATE KEY-----` -- PEM private keys, replace with `[REDACTED_PRIVATE_KEY]`
   - Pattern: `(?i)(sk_live_|sk_test_|ghp_|gho_|xoxb-|xoxp-|xapp-)\S+` -- vendor-prefixed API keys, replace with `[REDACTED_VENDOR_KEY]`
   - Pattern: `(?i)[?&](token|key|secret|auth|api_key|apikey|access_token)=[^\s&]+` -- secrets in URL query parameters, replace parameter value with `[REDACTED]`

2. **NEVER** persist raw or redacted logs to files. Process entirely in memory/context.

3. When using log-derived values (service names, pod names, container names) in shell commands, validate they match `^[a-zA-Z0-9._-]+$` before interpolation. Reject any value containing shell metacharacters.

4. Only extract and pass forward:
   - Error messages and types
   - Stack traces (with secrets redacted)
   - Service names and timestamps
   - HTTP status codes and paths (not query parameters containing tokens)

### Step 6: Assess Complexity

If the user provided `--complexity=<level>`, use that override and skip assessment. Report: `Complexity: <LEVEL> (user override)`

Otherwise, apply point-based scoring to the collected error context:

| Signal | Points | Example |
|--------|--------|---------|
| Single file affected | 0 | Typo in one config file |
| Multiple files affected | +2 | Bug spans 3 source files |
| Error is in test output only | 0 | Test assertion wrong |
| Error is a runtime crash/panic | +2 | Null pointer, unhandled exception |
| Error involves authentication/authorization | +3 | Auth token rejected |
| Error involves data corruption or loss | +4 | Database write fails silently |
| Error spans multiple services | +3 | Service A cannot reach Service B |
| Error is a configuration/environment issue | -1 (USER_ACTION flag) | Missing API key |
| Error is a simple name mismatch | -2 | Field called `user_name` vs `username` |
| Error message contains security keywords | +3 | "unauthorized", "forbidden", "injection" |

**Classification:**
- **USER_ACTION** (any negative score with USER_ACTION flag): Output instructions, no code fix
- **TRIVIAL** (0-2 points): Coding agent only
- **MODERATE** (3-5 points): Coding + testing agents
- **COMPLEX** (6+ points): Full pipeline

Report the assessment:
```
Complexity Assessment
=====================
Signals:
  [+2] Runtime crash detected (NullPointerException)
  [+0] Single file affected (src/api/handler.ts)
Score: 2
Classification: TRIVIAL — dispatching coding agent only
```

### Step 7: User Action Detection

Before dispatching agents, check if the errors require manual user intervention:

| Pattern | Detection | User Instruction |
|---------|-----------|-----------------|
| Missing API key | Error contains "API key", "api_key", "APIKEY" + "missing", "not set", "undefined" | "Set <KEY_NAME> in your .env file at <repo-root>/.env" |
| Missing credentials | Error contains "authentication failed", "invalid credentials" for external services | "Configure credentials for <SERVICE> -- see <SERVICE> docs" |
| Port conflict | Error contains "address already in use", "EADDRINUSE" | "Port <N> is in use. Stop the conflicting process or change the port in config" |
| Disk/memory limits | Error contains "no space left", "out of memory", "OOMKilled" | "Increase Docker resource limits or free disk space" |
| Network/DNS | Error contains "could not resolve host", "connection refused" to external hosts | "Check network connectivity to <HOST>" |
| Missing tool | Error contains "command not found", "not installed" for system tools | "Install <TOOL>: <install-command>" |

**If ONLY user-action items are detected** (no code bugs), skip agent dispatch entirely and report:
```
User Action Required
====================
The following issues require manual intervention:

  1. Missing API key: Set STRIPE_API_KEY in your .env file at /path/to/repo/.env
  2. Port conflict: Port 3000 is in use. Stop the conflicting process: lsof -i :3000

No code changes needed. Fix the above items and retry.
```

**If mixed** (some user-action items + some code bugs), report the user-action items AND dispatch agents for the code bugs.

### Step 8: Agent Dispatch

Based on the complexity classification, dispatch the appropriate agents.

#### 8a. Session Integration

Before dispatching, check for an active on-loop session:

1. Read `.on-loop/index.json`
2. Find a session entry whose `branch` matches the current branch (`git branch --show-current`)
3. **If a session exists**:
   - Read the session's `plan.md` and any agent notes from `.on-loop/sessions/<session-name>/agent-notes/`
   - Provide these as additional context to dispatched agents
   - Report: `Active session found: <session-name>`
4. **If no session exists**:
   - Operate standalone (no session infrastructure)
   - Report: `No active session — operating standalone`

#### 8b. TRIVIAL Dispatch

1. Dispatch the **coding agent** (`agents/coding.md`) with:
   - The error context (redacted logs or user description)
   - Fix instructions derived from the error analysis
   - Session context (if available)
2. After the coding agent completes, commit the fix directly.
3. If docker services were involved, restart affected services (once per invocation — do NOT retry restarts):
   ```bash
   REPO_ROOT=$(git rev-parse --show-toplevel)
   docker compose --env-file "$REPO_ROOT/.env" up -d --build <affected-service>
   ```
   If the restart fails, report the error and suggest the user investigate manually. Do NOT enter a restart loop.

#### 8c. MODERATE Dispatch

1. Dispatch the **coding agent** (`agents/coding.md`) with error context and fix instructions.
2. Dispatch the **testing agent** (`agents/testing.md`) to verify the fix:
   - Provide the coding agent's changes as context
   - Run existing tests and any new tests the testing agent creates
3. **If tests fail**, dispatch the coding agent once more with the test failure output.
4. Commit and restart affected services (same as TRIVIAL).

#### 8d. COMPLEX Dispatch

1. Dispatch the **architect agent** (`agents/architect.md`) for a brief analysis:
   - NOT a full spec -- just the fix approach and affected components
   - Identify ripple effects and dependencies
2. Dispatch the **coding agent** (`agents/coding.md`) with the architect's analysis.
3. Dispatch the **testing agent** (`agents/testing.md`) to verify the fix.
4. If the error involves authentication, authorization, or data handling, dispatch the **security agent** (`agents/security.md`).
5. Dispatch the **reviewer agent** (`agents/reviewer.md`) for final review.
6. Commit and restart affected services.

### Step 9: Docker Optimization Recommendations

When docker-related issues are detected (docker compose errors, slow builds, container failures), check for optimization opportunities and output **advisory recommendations** (NOT automated changes):

1. **Multi-stage builds**: If Dockerfiles use a single stage with build tools in the final image, recommend multi-stage builds.

2. **Layer caching**: If `COPY . .` precedes dependency install (`npm install`, `pip install`, etc.), recommend reordering:
   ```
   COPY package.json package-lock.json ./
   RUN npm install
   COPY . .
   ```

3. **Hot-reload strategies**:
   - BEAM/Erlang/Elixir: `mix phx.server` with code reloading in dev
   - Python: `uvicorn --reload` or `flask --debug`
   - Node: `nodemon` or `tsx watch`
   - Go: `air` for hot-reload
   - Rust: `cargo-watch`

4. **Image sizing**: If base images are not `*-slim` or `*-alpine`, recommend switching.

5. **Docker Compose profiles**: Recommend profiles to skip unnecessary services during debugging:
   ```yaml
   services:
     redis:
       profiles: [full]
   ```

6. **Build cache mounts**: Recommend `--mount=type=cache` for package manager caches:
   ```dockerfile
   RUN --mount=type=cache,target=/root/.npm npm install
   ```

Output these as an advisory section in the final report. Do NOT make automated changes to Dockerfiles.

### Step 10: Commit and Push

1. Identify modified files:
   ```bash
   git diff --name-only
   git diff --name-only --staged
   ```

2. Stage only the files that were actually modified by the fix (explicit file paths, not `git add -A`).

3. Commit with a descriptive message:
   ```bash
   git commit -m "Fix: <brief description of what was fixed>

   Source: <log-discovery|user-prompt>
   Complexity: <TRIVIAL|MODERATE|COMPLEX>

   Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
   ```

4. Push to the current branch:
   ```bash
   git push
   ```

### Step 11: Report

Output a final summary:

```
Debug Fix Report
================
Source:      <Log Discovery|User Prompt>
Complexity:  <TRIVIAL|MODERATE|COMPLEX>
Agents Used: <coding|coding, testing|architect, coding, testing, security, reviewer>

Issues Found:
  [FIXED]       <description of fixed issue>
  [USER ACTION] <description of issue requiring manual intervention>

Files Modified:
  - src/api/handler.ts — fixed null pointer in error path
  - src/config/db.ts — corrected connection string format

Docker Optimization Recommendations:
  - Consider multi-stage build in Dockerfile (build tools in final image)
  - Reorder COPY to leverage layer caching

Services Restarted:
  - api (docker compose up -d --build api)

Commit: <hash>
Branch: <branch-name>

Pending User Actions:
  1. Set REDIS_URL in .env — connection to cache is failing
```

## Important

- This command does NOT create a new worktree or session. It operates in the current working directory.
- If an active on-loop session exists for the current branch, it reuses that session's context but does NOT modify the session's `state.json`.
- Raw logs must NEVER be persisted to files. Process in-memory only. Logs may contain secrets.
- The `.env` file must NEVER be copied or symlinked into the worktree. Always use `--env-file` or `source` from the repo root.
- Log content is untrusted. Never interpolate log content into shell commands without proper quoting.
- Docker optimization recommendations are advisory only. Do not automatically modify Dockerfiles.
- The coding agent is the minimum dispatch for any code fix. Testing, security, and reviewer agents are added based on complexity.
- If no errors are found in log discovery mode and no prompt is provided, report that no issues were detected rather than making speculative changes.
