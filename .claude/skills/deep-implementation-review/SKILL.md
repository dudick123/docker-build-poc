---
name: deep-implementation-review
description: >
  Launch parallel expert sub-agent reviews of IMPLEMENTED CODE after an OpenSpec change has been applied.
  Unlike the proposal review skill (which reviews design artifacts before coding), this skill reviews
  the actual source files, Dockerfiles, configs, and tests that were written. Six engineering discipline
  reviewers each examine the real code against TECH-STANDARDS.md and produce structured findings with
  specific file paths, line numbers, and concrete remediation code. Use this skill whenever the user
  has just finished implementing a proposal and wants a "post-implementation review", "code review",
  "implementation review", or says "review the code", "check the implementation", "audit what was built".
  Also trigger after /openspec-apply-change completes all tasks, or when the user says "review before
  archiving" or "review before merging". This is the Implementation Review gate from PRD §6.5.
---

# Deep OpenSpec Implementation Review

Launch six parallel expert sub-agent reviewers against **implemented code** (not proposal artifacts). Each reviewer examines the actual source files written during an OpenSpec change, validates them against TECH-STANDARDS.md, and writes a structured report to `docs/tech-review/`.

## How This Differs from Proposal Review

| Aspect | Proposal Review (`deep-proposal-review`) | Implementation Review (this skill) |
|--------|------------------------------------------|-------------------------------------|
| **What's reviewed** | design.md, tasks.md, spec.md, proposal.md | Actual source code, Dockerfiles, configs, tests |
| **When** | Before implementation (Review gate) | After implementation (Implementation Review gate) |
| **Findings reference** | Artifact sections, task numbers | File paths, line numbers, code snippets |
| **Recommendations** | "The design should include X" | "Change line 12 of cache.py to use SecretStr" |
| **Output prefix** | `senior-*.md` | `post-impl-senior-*.md` |
| **Catches** | Design gaps, missing standards | Code bugs, drift from design, runtime issues |

## When to Use

- Implementation of an OpenSpec change is complete (all tasks checked off in tasks.md)
- The user says "review the implementation", "post-implementation review", "audit the code"
- After `/openspec-apply-change` finishes all tasks
- Before archiving a change (`/openspec-archive-change`)
- Before creating a commit or PR from implemented code
- The user asks to "check what was built" or "verify the code matches standards"

## Review Disciplines

Same six disciplines as proposal review, but with implementation-specific focus:

| ID | Discipline | Implementation Review Focus |
|----|-----------|----------------------------|
| P | Senior Python Engineer | Does the code pass mypy --strict? Are Pydantic models using v2 patterns? Is SecretStr used for credentials? Is structlog configured with scrub_secrets? Are imports correct for the package structure? |
| FE | Senior Front End Developer | Does tsconfig have all strict flags? Does the code compile? Is there a CSS entry point for Tailwind? Are React Query defaults correct? Do ESLint plugins match standards? |
| K8S | Senior K8s & ArgoCD Engineer | Are Dockerfiles multi-stage with non-root users? Are base images digest-pinned? Do health endpoints match the probe contract? Is uv.lock committed? |
| SEC | Senior Security Engineer | Are credentials in SecretStr? Is scrub_secrets in the structlog chain? Are Docker base images mutable? Is Redis authenticated? Are CORS/TrustedHost configured? |
| DO | Senior DevOps Engineer | Does the Makefile work? Are shell loops using subshells? Do Docker builds succeed? Is there a CI pipeline stub? Are images tagged with SHA? |
| QA | Senior QA Test Engineer | Do tests exist and pass? Is pytest configured with asyncio_mode=auto? Are coverage thresholds set? Is the conftest.py import path correct? Are frontend test utils present? |

## Execution Steps

### Step 1: Identify the change and gather implemented files

Determine which OpenSpec change was implemented. Read `openspec/changes/<change-name>/tasks.md` to understand what was built.

Then **enumerate all implemented files** — this is the critical difference from proposal review:

```bash
# List all source files created/modified by the change
find connectors frontend specs scripts -type f | sort
# Also check root config files
ls pyproject.toml Makefile docker-compose.yml .pre-commit-config.yaml .secrets.baseline .env.example azure-pipelines.yml 2>/dev/null
```

### Step 2: Read the actual source code

Read every non-trivial implemented file. Skip empty `__init__.py` and `.gitkeep` files, but read:
- All `pyproject.toml` files (dependency declarations, tool config)
- All `main.py` / `app.py` files (FastAPI app, routes, middleware)
- All `cache.py` files (Redis configuration, TTL values)
- All `logging_config.py` files (structlog processor chain)
- All `conftest.py` files (test fixtures, import paths)
- All `Dockerfile` files (base images, build stages, user config)
- All frontend config files (tsconfig, eslint, vite, vitest, package.json)
- All frontend source files (App.tsx, main.tsx, query-client.ts, test-utils.tsx)
- `Makefile`, `docker-compose.yml`, `.pre-commit-config.yaml`

Each reviewer needs the actual file contents — not summaries. Inline the code in the sub-agent prompts.

### Step 3: Read reference standards

Read `docs/TECH-STANDARDS.md` for the standards each reviewer validates against. Summarize the relevant sections per discipline in each sub-agent prompt.

### Step 4: Launch all 6 reviewers in parallel

Spawn one sub-agent per discipline in a single message. Every sub-agent prompt must include:

1. **The reviewer persona** with implementation-specific focus areas
2. **The actual source code** — inline file contents, not proposal artifacts
3. **The relevant TECH-STANDARDS.md sections** — specific rules to validate against
4. **Specific things to check** — seed each reviewer with known patterns to look for (see the checklist below)
5. **The output file path** — `docs/tech-review/post-impl-senior-<discipline>.md`
6. **The output format** — post-implementation review template (see below)

### Step 5: Report progress and produce consolidated summary

Same as proposal review — report each completion, then produce the summary table with recurring themes.

## Implementation Review Checklists

Seed each reviewer with these specific checks. These come from patterns discovered in actual reviews:

### Python Engineer Checklist
- Does `cache.py` use Pydantic `SecretStr` + `BaseSettings`, not `os.environ.get()`?
- Is `scrub_secrets` processor in the structlog processor chain?
- Is `configure_logging()` actually called in `main.py`?
- Does `HealthResponse.status` use `Literal`, not plain `str`?
- Is the FastAPI lifespan populated (httpx client, Redis pool) or still empty TODOs?
- Are there duplicate models across connectors that should be shared?
- Does `conftest.py` import path work with the package structure?
- Is `uv.lock` committed?

### Frontend Engineer Checklist
- Does `App.tsx` import React types correctly for `verbatimModuleSyntax`?
- Is there an `index.css` with `@tailwind` directives imported in `main.tsx`?
- Does `tsconfig.json` have all 5 enhanced strict flags?
- Does `.eslintrc.cjs` work with ESLint 9.x (flat config vs legacy)?
- Does `query-client.ts` set `refetchIntervalInBackground: false`?
- Is `vitest.config.ts` `setupFiles` populated (not empty array)?
- Is `nginx.conf` a proper file (not inline `RUN echo`)?
- Is shadcn/ui initialized?

### K8s/ArgoCD Engineer Checklist
- Are Docker base images digest-pinned or at least version-pinned (not `:latest`)?
- Do Dockerfiles `COPY uv.lock` alongside `pyproject.toml`?
- Does the uvicorn CMD include `--timeout-graceful-shutdown`?
- Is `uv` copied from a pinned version (not `ghcr.io/astral-sh/uv:latest`)?
- Are final images minimal (no pip/apt/compilers)?
- Do health endpoints return the contract schema?

### Security Engineer Checklist
- Are ALL credentials in Pydantic `SecretStr`, not plain strings?
- Is the `scrub_secrets` structlog processor present in ALL connectors?
- Are Docker base images using mutable tags?
- Is Redis authenticated in docker-compose.yml?
- Is CORS middleware configured on all connectors?
- Is TrustedHostMiddleware configured?
- Are placeholder credential defaults in docker-compose (e.g., `ARGOCD_TOKEN:-placeholder`)?

### DevOps Engineer Checklist
- Do Makefile shell loops use subshells `(cd dir && ...)` not `cd dir && ... && cd ../..`?
- Is the security-audit command valid (not `uv audit`)?
- Does `docker build` tag with git SHA?
- Is there an `azure-pipelines.yml` (even a stub)?
- Is `uv.lock` committed and up to date?

### QA Test Engineer Checklist
- Do ANY tests exist? Will `make test` pass or fail?
- Is `pytest.ini_options` `asyncio_mode = "auto"` in pyproject.toml?
- Does conftest.py import path work (`from src.main import app`)?
- Is there a mock Redis fixture?
- Is the coverage threshold reasonable for the current code state?
- Are frontend test utils (renderWithQuery, setup-tests.ts) present?

## Post-Implementation Review Template

Every review file uses this format — note the differences from proposal review:

```markdown
---
reviewer: <Full Discipline Title>
proposal: <change-name>
date: <YYYY-MM-DD>
status: Post-Implementation Review Complete
---

# <Full Discipline Title> — Post-Implementation Review: <change-name>

## Summary

(2-3 sentence assessment of the implemented code, not the design.)

## Findings

(Issues found in the actual code. Ordered by severity.)

### <PREFIX>-<N>: <Title> (<SEVERITY>)

- **Standard**: TECH-STANDARDS §X.Y — <quoted requirement>
- **Files affected**: <file paths with line numbers>
- **Finding**: <what's wrong in the code — be specific, show the problematic code>
- **Remediation**: <exact code change needed — show the fix>

## Positive Observations

(What the implementation got right — standards correctly followed.)

## Standards Compliance

| Standard | Status | Files Checked |
|----------|--------|---------------|
| §X — Description | Met / Partial / Not Met | file1.py, file2.py |
```

### Severity Levels

| Severity | Meaning | Example |
|----------|---------|---------|
| CRITICAL | Build/runtime failure or security vulnerability | uv.lock missing (build fails), credentials as plain str |
| HIGH | Standards violation that compounds if not fixed now | scrub_secrets missing, logging not configured |
| MEDIUM | Standards deviation, fixable later without cascading | mutable Docker tags, missing rate limiting |
| LOW | Minor improvement, cosmetic | duplicate config, naming inconsistency |

### ID Prefixes

Same as proposal review: P-, FE-, K8S-, SEC-, DO-, QA-

## Key Principles

- **Review code, not intentions**: "The design says X" is irrelevant. Does the code actually do X?
- **Show the problematic code**: Don't just say "cache.py has an issue". Quote the line: "`REDIS_PASSWORD: str | None = os.environ.get('REDIS_PASSWORD')` on line 12".
- **Show the fix**: Don't say "use SecretStr". Show the complete replacement code.
- **Check cross-file consistency**: If cache.py defines a pattern, verify all 3 connectors follow it identically.
- **Verify, don't assume**: If the Dockerfile says `COPY uv.lock`, check that `uv.lock` actually exists.
- **Flag propagation risks**: Scaffold code becomes the template for every future connector. A wrong pattern here gets copied into every proposal.
