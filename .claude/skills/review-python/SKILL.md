---
name: review-python
description: >
  Senior Python Engineer review of an OpenSpec proposal or implemented code. Examines async patterns,
  Pydantic v2 models, FastAPI lifespan, httpx client usage, structlog configuration, Redis caching,
  dependency management (uv), mypy strict compliance, and type safety. Use when working on connector
  proposals, Python code changes, or when the user asks for a Python-focused review.
---

# Senior Python Engineer Review

Perform a deep technical review from the perspective of a Senior Python Engineer. Examine either an OpenSpec proposal (proposal.md, design.md, tasks.md, specs/) or implemented Python code. Validate against `docs/TECH-STANDARDS.md` sections 1, 3, and 4.

## When to Use

- A new or updated connector proposal is ready for review
- Python code changes (connectors, shared libraries) need review
- The user asks for a "Python review" or "backend review"
- The `/review` router delegates to this skill

## Execution Steps

### Step 1: Identify the target

Determine what to review:
- If the user specifies a proposal, read `openspec/changes/<proposal-name>/` artifacts (proposal.md, design.md, tasks.md, specs/)
- If the user specifies code, read the relevant Python files
- If neither is specified, check for the most recent proposal in `openspec/changes/`

### Step 2: Read the standards

Read the following sections from `docs/TECH-STANDARDS.md`:
- Section 1: Python Code Style (Connectors) — PEP compliance, formatting, linting, type annotations, naming, imports, docstrings
- Section 3: Dependency Management — uv requirements, lockfile strategy, pyproject.toml structure
- Section 4: FastAPI Application Patterns — lifespan, middleware, error handling, Redis caching, httpx clients, structlog

Also read `CLAUDE.md` for project-level context (planned tech stack, architecture decisions).

### Step 3: Create the output directory

```bash
mkdir -p docs/tech-review
```

### Step 4: Perform the review

Evaluate the proposal or code against these focus areas. For each, check both the proposal artifacts AND (if available) the implemented code:

**Dependency Management**
- uv is used (never pip). Check for `uv sync --frozen` in CI, `uv.lock` committed, `pyproject.toml` with `requires-python = ">=3.14"`
- No `requirements.txt` files — uv uses `pyproject.toml` + `uv.lock`

**Pydantic v2 Models**
- `SecretStr` used for all credentials (API tokens, passwords, connection strings)
- `model_config = ConfigDict(frozen=True)` on immutable models
- Pydantic `BaseSettings` for environment configuration with `env_prefix`
- No `dict()` — use `model_dump()`. No `json()` — use `model_dump_json()`

**structlog Configuration**
- `scrub_secrets` processor configured to redact SecretStr fields
- JSON renderer in production, console renderer in development
- Bound loggers with context (connector name, environment, request_id)
- No secrets or credential values in log messages

**FastAPI Lifespan**
- `@asynccontextmanager` lifespan function populates `app.state` with shared resources (httpx client, Redis pool)
- No global mutable state — all shared state lives on `app.state`
- Graceful shutdown closes httpx clients and Redis connections

**httpx Client Usage**
- `httpx.AsyncClient` with explicit timeouts (`timeout=httpx.Timeout(connect=5.0, read=30.0, write=10.0, pool=5.0)`)
- Client created in lifespan, stored on `app.state`, closed on shutdown
- No `requests` library — httpx only

**Redis Caching**
- `orjson` serialization (not json module)
- 30-minute uniform TTL across all connectors (TECH-STANDARDS §4)
- Cache key convention: `{connector}:{resource}:{params}`
- No persistence — Redis is ephemeral cache only

**asyncio Correctness**
- No blocking I/O in async functions (no `time.sleep`, no synchronous HTTP calls)
- `asyncio.TaskGroup` for concurrent operations (not `gather` with bare exceptions)
- Proper exception handling in async contexts

**mypy Strict Compliance**
- `mypy --strict` must pass with zero errors
- No `Any` except at true integration boundaries
- Complete type annotations on all functions and methods
- `TYPE_CHECKING` imports for annotation-only imports

**Ruff Configuration**
- All required rule sets enabled (E, F, W, I, UP, S, B, A, C4, PT, SIM, TCH, DTZ, PIE, RSE, RET, FBT, ASYNC, RUF)
- Line length 99
- Config in `pyproject.toml` under `[tool.ruff]` — no separate ruff.toml

### Step 5: Write the review

Write the review to `docs/tech-review/{proposal}-python-review.md` using this exact format:

```markdown
---
reviewer: Senior Python Engineer
proposal: <proposal-name>
date: <YYYY-MM-DD>
status: Review Complete
---

# Senior Python Engineer — Review: <proposal-name>

## Summary

(2-3 sentence overall assessment. Be direct about severity.)

## Critical Findings

(Must-fix items. Use P- prefix for finding IDs.)

### Finding P-<N>: <Title>

- **Artifact**: (which file: design.md, tasks.md, spec.md, proposal.md, or source file path)
- **Location**: (section, task number, or line reference)
- **Issue**: (what is wrong — quote the specific TECH-STANDARDS section violated)
- **Impact**: (concrete consequences if not fixed)
- **Recommendation**: (specific fix, not vague guidance)

## Recommendations

(Should-fix improvements. Same structure as findings.)

### Recommendation P-<N>: <Title>

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
| TECH-STANDARDS §1.1 — PEP 8/257/484 | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §1.2 — Python 3.14 minimum | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §1.3 — Ruff formatter (99 chars) | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §1.4 — Ruff linter (all rule sets) | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §1.5 — mypy --strict | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §3 — uv dependency management | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §4.1 — FastAPI lifespan | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §4.2 — httpx timeouts | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §4.3 — Redis orjson + 30min TTL | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §4.4 — structlog + scrub_secrets | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §4.5 — Pydantic SecretStr for creds | Met / Partial / Not Met / N/A | |
```

### Step 6: Report results

After writing the review file, report to the user:
- Number of critical findings and recommendations
- Top 2-3 most important issues
- Overall standards compliance assessment
- Path to the full review file
