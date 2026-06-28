---
name: review-qa
description: >
  Senior QA Test Engineer review. Examines test strategy, coverage targets, TDD adherence, test
  infrastructure, and mock fidelity. This reviewer runs as a lightweight second pass on every
  proposal to verify testing quality. Use alongside any other reviewer, or standalone for
  test-focused changes.
---

# Senior QA Test Engineer Review

Perform a deep technical review from the perspective of a Senior QA Test Engineer. Examine either an OpenSpec proposal (proposal.md, design.md, tasks.md, specs/) or implemented test code. Validate against `docs/TECH-STANDARDS.md` testing sections. This reviewer is designed to run as a second pass alongside any primary discipline reviewer.

## When to Use

- Any proposal or code change needs test quality verification
- Test infrastructure, fixtures, or mock setup is being modified
- The user asks for a "QA review", "test review", or "testing review"
- The `/review` router adds this as a second reviewer alongside the primary discipline
- Standalone for test-focused changes (new test files, coverage improvements, test infra)

## Execution Steps

### Step 1: Identify the target

Determine what to review:
- If the user specifies a proposal, read `openspec/changes/<proposal-name>/` artifacts (proposal.md, design.md, tasks.md, specs/)
- If the user specifies code, read the relevant test files, conftest.py, and test configuration
- If neither is specified, check for the most recent proposal in `openspec/changes/`

### Step 2: Read the standards

Read testing-relevant sections from `docs/TECH-STANDARDS.md`:
- Section 1 — Python testing standards (pytest, asyncio, coverage)
- Section 2 — Frontend testing standards (Vitest, RTL, MSW, vitest-axe)
- Section 7 — Test infrastructure and CI integration
- Any BDD scenarios in the proposal's `specs/` directory

Also read `CLAUDE.md` for project context.

### Step 3: Create the output directory

```bash
mkdir -p docs/tech-review
```

### Step 4: Perform the review

Evaluate the proposal or code against these focus areas:

**pytest Configuration**
- `asyncio_mode = "auto"` in pytest configuration (pyproject.toml or pytest.ini)
- All async tests run without manual `@pytest.mark.asyncio` decoration
- `pytest-cov` configured for coverage reporting
- `pytest-timeout` configured to prevent hanging tests

**Coverage Targets**
- 90% line coverage minimum enforced in CI
- Coverage measured per-package, not just aggregate
- Coverage report generated in CI and attached to PR
- Critical paths (error handling, cache miss, timeout) have explicit test coverage
- No `# pragma: no cover` without justification comment

**conftest.py Fixtures**
- `AsyncClient` fixture for FastAPI integration tests (uses `httpx.AsyncClient` with `ASGITransport`)
- `mock_redis` fixture providing a fake Redis (not a real Redis instance in unit tests)
- Fixtures use `yield` for setup/teardown lifecycle
- Fixtures scoped appropriately: `session` for expensive setup, `function` for test isolation
- Shared fixtures in top-level `conftest.py`, test-specific fixtures in local `conftest.py`

**Frontend Testing — Vitest + RTL + MSW**
- Vitest as test runner (not Jest) with `vitest.config.ts`
- React Testing Library (RTL) for component tests — testing behaviour, not implementation
- MSW (Mock Service Worker) for API mocking in tests and development
- MSW handlers match the OpenAPI spec responses (mock fidelity)
- `vitest-axe` for accessibility assertions on module-level components
- `@testing-library/user-event` for user interaction simulation (not `fireEvent`)

**Test Naming Convention**
- Python: `test_<behaviour>` — describes what is being tested, not the function name
  - Good: `test_returns_cached_response_when_redis_hit`
  - Bad: `test_get_apps`, `test_function_1`
- TypeScript: `it('should <behaviour>')` or `test('<behaviour>')`
- Test files mirror source structure: `src/foo.py` → `tests/test_foo.py`

**TDD Adherence**
- Tasks in `tasks.md` should specify tests before implementation
- BDD scenarios in `specs/` should map to test cases
- Red-green-refactor cycle: test written first, fails, implementation makes it pass
- Verify that the proposal includes test tasks for every feature task

**Security Test Patterns**
- Input validation tests: malformed inputs, boundary values, injection attempts
- Error leakage tests: verify 500 errors do not expose stack traces or internal paths
- Authentication tests: verify unauthorized requests receive 401/403
- Credential tests: verify SecretStr fields do not appear in logs (capture log output in test)
- CORS tests: verify cross-origin requests are handled correctly

**Mock Fidelity**
- Mock responses match the OpenAPI spec schema (not arbitrary data)
- Mock error responses test all documented error codes (400, 404, 500, 502, 503)
- Mock data covers edge cases: empty lists, maximum-size responses, unicode characters
- Integration tests verify against real API contracts (contract tests)

**Spec Testability**
- BDD scenarios in `specs/` are testable (Given/When/Then maps to test setup/action/assertion)
- Acceptance criteria are measurable and automatable
- No vague criteria like "should be fast" — use specific thresholds
- Each scenario has a clear expected outcome

### Step 5: Write the review

Write the review to `docs/tech-review/{proposal}-qa-review.md` using this exact format:

```markdown
---
reviewer: Senior QA Test Engineer
proposal: <proposal-name>
date: <YYYY-MM-DD>
status: Review Complete
---

# Senior QA Test Engineer — Review: <proposal-name>

## Summary

(2-3 sentence overall assessment. Be direct about severity.)

## Critical Findings

(Must-fix items. Use QA- prefix for finding IDs.)

### Finding QA-<N>: <Title>

- **Artifact**: (which file: design.md, tasks.md, spec.md, conftest.py, test file path, etc.)
- **Location**: (section, task number, scenario name, or line reference)
- **Issue**: (what is wrong — quote the specific TECH-STANDARDS section violated)
- **Impact**: (concrete consequences if not fixed)
- **Recommendation**: (specific fix, not vague guidance)

## Recommendations

(Should-fix improvements. Same structure as findings.)

### Recommendation QA-<N>: <Title>

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
| TECH-STANDARDS — pytest asyncio_mode=auto | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS — 90% coverage minimum | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS — conftest.py fixtures (AsyncClient, mock_redis) | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS — Vitest + RTL + MSW frontend testing | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS — Test naming (test_<behaviour>) | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS — TDD (tests before implementation) | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS — Security test patterns | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS — Mock fidelity (matches OpenAPI spec) | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS — BDD scenario testability | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS — vitest-axe accessibility assertions | Met / Partial / Not Met / N/A | |
```

### Step 6: Report results

After writing the review file, report to the user:
- Number of critical findings and recommendations
- Top 2-3 most important issues
- Overall test quality assessment
- Path to the full review file
