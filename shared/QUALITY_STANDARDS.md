# Quality Standards

These standards define the quality bar for all work produced by on-loop agents. The bar is set for **regulated financial environments** and **critical infrastructure**.

## Code Quality

### Must Have
- All public functions/methods have clear, descriptive names
- No dead code, no commented-out code blocks
- No hardcoded secrets, credentials, or PII
- All error paths explicitly handled (no bare `except`, no swallowed errors)
- Input validation at all system boundaries
- Consistent code style (enforced by linter configuration)
- No known CVEs in dependencies

### Should Have
- Functions under 50 lines; files under 500 lines
- Cyclomatic complexity under 10 per function
- Type annotations (where language supports them)
- Structured logging with correlation IDs

## Testing Quality

### Must Have
- Unit tests for all business logic
- Integration tests for all API endpoints and data flows
- Tests cover both happy path and error paths
- No tests that depend on external services without mocks/stubs
- Tests are deterministic (no flaky tests)
- Test names describe the behavior being tested

### Should Have
- Code coverage above 80% for new code
- Edge case coverage (empty inputs, boundary values, concurrent access)
- Performance/load tests for critical paths
- E2E tests for critical user journeys (Playwright where applicable)

## Security Quality

### Must Have
- OWASP Top 10 mitigations verified
- No SQL injection, XSS, CSRF, command injection vectors
- Authentication and authorization on all protected endpoints
- Secrets loaded from environment or secret manager (never from code)
- TLS for all network communication
- Input sanitization and output encoding
- Rate limiting on public endpoints
- Security headers configured (CSP, HSTS, X-Frame-Options, etc.)

### Should Have
- STRIDE threat model for new features
- Dependency vulnerability scan (no critical/high CVEs)
- Least-privilege IAM roles/policies
- Audit logging for all state-changing operations
- Data classification labels (public, internal, confidential, restricted)

## Documentation Quality

### Must Have
- README with setup instructions, usage examples, and architecture overview
- CLAUDE.md with project conventions and agent instructions
- API documentation for all public endpoints
- Environment variable documentation
- Clear error messages that guide the user toward resolution

### Should Have
- Architecture decision records (ADRs) for significant choices
- Mermaid diagrams for system architecture and data flows
- Runbook for common operational tasks
- CHANGELOG maintained

## Build & CI Quality

### Must Have
- Reproducible builds (pinned dependencies, lockfiles)
- Linter and formatter configured and enforced in CI
- Tests run in CI on every PR
- Security scanning in CI (dependency audit, SAST)
- Build fails on lint errors, test failures, or critical security findings

### Should Have
- Makefile or task runner with standard targets (build, test, lint, security)
- Docker build with minimal base image and non-root user
- CI caching for dependencies
- Branch protection rules documented

## Review Quality

### Must Have
- All code changes reviewed before merge
- Review checks: correctness, security, performance, naming, error handling
- No TODOs without linked issues
- No suppressed warnings without documented justification

### Should Have
- Review checklist followed consistently
- Patterns and anti-patterns documented for the project
- Knowledge sharing notes for non-obvious decisions
