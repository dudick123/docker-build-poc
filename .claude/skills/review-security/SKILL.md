---
name: review-security
description: >
  Senior Security Engineer review. Examines credential handling, TLS configuration, CORS/CSP headers,
  input validation, container hardening, secret detection, and supply chain security. Use for any
  proposal that touches credentials, authentication, Docker images, or network configuration.
---

# Senior Security Engineer Review

Perform a deep technical review from the perspective of a Senior Security Engineer. Examine either an OpenSpec proposal (proposal.md, design.md, tasks.md, specs/) or implemented code and configuration. Validate against `docs/TECH-STANDARDS.md` security-related sections throughout the document.

## When to Use

- A proposal touches credentials, authentication, or authorization
- Changes involve TLS, CORS, CSP headers, or network configuration
- Docker images, container security contexts, or secret management are modified
- The user asks for a "security review" or "sec review"
- The `/review` router delegates to this skill

## Execution Steps

### Step 1: Identify the target

Determine what to review:
- If the user specifies a proposal, read `openspec/changes/<proposal-name>/` artifacts (proposal.md, design.md, tasks.md, specs/)
- If the user specifies code, read the relevant source files, Dockerfiles, and configuration
- If neither is specified, check for the most recent proposal in `openspec/changes/`

### Step 2: Read the standards

Read security-relevant sections from `docs/TECH-STANDARDS.md`:
- Section 1.5 — Ruff S (bandit) rules, noqa suppression requiring review approval
- Section 2.14 — Frontend security patterns (XSS, localStorage, CSP)
- Section 4 — FastAPI middleware (CORS, TrustedHostMiddleware), error handling (no secrets in responses)
- Section 5 — Container hardening (non-root, read-only filesystem)
- Section 7 — Secret management (External Secrets Operator, Azure Key Vault)
- Section 12 — Security and hardening (comprehensive security controls)

Also read `CLAUDE.md` for project context (credential flow, architecture boundaries).

### Step 3: Create the output directory

```bash
mkdir -p docs/tech-review
```

### Step 4: Perform the review

Evaluate the proposal or code against these focus areas:

**Credential Handling — Pydantic SecretStr**
- ALL credentials (API tokens, passwords, connection strings, Redis auth) use `Pydantic.SecretStr`
- SecretStr prevents accidental logging — `repr()` returns `'**********'`
- Credentials loaded via `BaseSettings` from environment variables, never hardcoded
- No credential values in default parameters, docstrings, or comments

**structlog scrub_secrets Processor**
- `scrub_secrets` processor configured in structlog pipeline
- Processor detects and redacts SecretStr fields in log events
- Test coverage verifies secrets do not appear in log output
- No manual `.get_secret_value()` calls in log statements

**CORS Configuration**
- Explicit allowed origins — no wildcard `*` in production
- `allow_methods` restricted to `GET` only (dashboard is read-only)
- `allow_credentials: false` (no cookies or auth headers from browser)
- CORS middleware added via FastAPI, not a reverse proxy (to keep config in code)

**TrustedHostMiddleware**
- `TrustedHostMiddleware` configured with explicit allowed hosts
- Prevents host header injection attacks
- Configured per-environment (dev allows localhost, prod restricts to cluster DNS)

**Redis Authentication**
- Redis connection uses password authentication (`--requirepass`)
- Password sourced from SecretStr environment variable
- Redis connection string does not appear in logs or error responses
- TLS for Redis if crossing network boundaries

**Secret Detection — detect-secrets**
- `.secrets.baseline` file committed to the repository
- `detect-secrets` runs in CI pre-commit hook
- Known false positives documented in the baseline with justification
- No high-entropy strings in source code without baseline entry

**.gitignore Secret Patterns**
- `.env`, `.env.*` files in `.gitignore`
- `*.pem`, `*.key`, `*.p12` in `.gitignore`
- `credentials.json`, `service-account.json` in `.gitignore`
- No committed secret files in repository history

**CSP Headers**
- `Content-Security-Policy` header configured on frontend nginx
- `script-src 'self'` — no inline scripts, no eval
- `style-src 'self' 'unsafe-inline'` only if Tailwind requires it
- `connect-src` restricted to known connector Service DNS names
- `frame-ancestors 'none'` — no embedding

**Error Response Security**
- Error responses do not leak internal details (stack traces, file paths, database errors)
- FastAPI exception handlers return structured error responses with safe messages
- 500 errors return a correlation ID for debugging, not the exception message
- Pydantic validation errors sanitized before returning to client (no field values)

**Container Hardening**
- No mutable Docker tags (`latest`, `stable`) — digest-pinned bases
- `runAsNonRoot: true` in securityContext
- `readOnlyRootFilesystem: true` where possible
- `allowPrivilegeEscalation: false`
- Capabilities dropped: `ALL`, add back only what is needed

**Input Validation**
- All API inputs validated by Pydantic models (path params, query params, request body)
- String inputs have max length constraints
- Enum inputs use Literal types or Python enums
- No raw user input passed to shell commands, file paths, or database queries

**Supply Chain Security**
- `uv.lock` committed and `uv sync --frozen` in CI (no resolution at build time)
- Dependencies audited — `uv audit` or equivalent in CI pipeline
- Docker base images from trusted registries only
- No `curl | bash` installation patterns in Dockerfiles

### Step 5: Write the review

Write the review to `docs/tech-review/{proposal}-security-review.md` using this exact format:

```markdown
---
reviewer: Senior Security Engineer
proposal: <proposal-name>
date: <YYYY-MM-DD>
status: Review Complete
---

# Senior Security Engineer — Review: <proposal-name>

## Summary

(2-3 sentence overall assessment. Be direct about severity.)

## Critical Findings

(Must-fix items. Use SEC- prefix for finding IDs.)

### Finding SEC-<N>: <Title>

- **Artifact**: (which file: design.md, tasks.md, spec.md, proposal.md, or source file path)
- **Location**: (section, task number, or line reference)
- **Issue**: (what is wrong — quote the specific TECH-STANDARDS section violated)
- **Impact**: (concrete consequences if not fixed)
- **Recommendation**: (specific fix, not vague guidance)

## Recommendations

(Should-fix improvements. Same structure as findings.)

### Recommendation SEC-<N>: <Title>

- **Artifact**:
- **Location**:
- **Issue**:
- **Impact**:
- **Recommendation**:

## Observations

(Nice-to-have notes, minor items, things to watch in future proposals.)

## Standards Compliance

| Standard | Status | Notes |
|----------|--------|-------|
| TECH-STANDARDS — Pydantic SecretStr for all credentials | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS — structlog scrub_secrets processor | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS — CORS explicit origins, GET-only | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS — TrustedHostMiddleware | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS — Redis authentication | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS — detect-secrets baseline | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS — .gitignore secret patterns | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS — CSP headers on frontend | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS — No secrets in error responses | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS — Container hardening (non-root, RO FS) | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS — Input validation via Pydantic | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS — Supply chain (uv.lock, digest pins) | Met / Partial / Not Met / N/A | |
```

### Step 6: Report results

After writing the review file, report to the user:
- Number of critical findings and recommendations
- Top 2-3 most important issues (prioritize credential and data exposure risks)
- Overall standards compliance assessment
- Path to the full review file
