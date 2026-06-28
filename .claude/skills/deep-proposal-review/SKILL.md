---
name: deep-proposal-review
description: >
  Launch parallel expert sub-agent reviews of an OpenSpec proposal (or any design/spec/task artifact set).
  Six engineering discipline reviewers each produce a structured markdown review with findings, recommendations,
  and standards compliance. Use this skill whenever the user asks for a "deep review", "expert review",
  "discipline review", "proposal review", "tech review", or wants multiple engineering perspectives on a
  proposal, design document, spec, or set of implementation tasks. Also trigger when the user references
  the review gate process from PRD section 6.5, or asks to "run the review disciplines" or "get expert
  feedback" on any change. This skill is central to the project's quality process — use it proactively
  when a new proposal is created or when implementation PRs are ready for review.
---

# Deep OpenSpec Proposal Review

Launch six parallel expert sub-agent reviewers against an OpenSpec proposal (or any set of design/spec/task artifacts). Each reviewer writes a structured markdown report to `docs/tech-review/`.

## When to Use

- A new OpenSpec proposal is ready for review (the **Review** gate in PRD §6.2)
- Implementation code is ready for merge (the **Implementation Review** gate in PRD §6.2)
- The user asks for "deep review", "expert review", "discipline review", or "tech review"
- The user references PRD §6.5 (Discipline Reviews)
- A cross-cutting change touches multiple layers (API spec, deployment, frontend, security)

## Review Disciplines

| ID | Discipline | Focus Areas |
|----|-----------|-------------|
| P | Senior Python Engineer | Code patterns, async correctness, Pydantic models, error handling, dependency management (`uv`), structlog, type safety, mypy strict |
| FE | Senior Front End Developer | React architecture, TypeScript strictness, React Query config, accessibility (WCAG AA), bundle performance, error boundaries, testing (Vitest/RTL/MSW) |
| K8S | Senior Kubernetes & ArgoCD Engineer | Deployment manifests, probes, resource sizing, RBAC, ArgoCD sync policy, HPA/PDB, Kustomize, container hardening |
| SEC | Senior Security Engineer | Credential handling, TLS, CORS/CSP, input validation, container hardening, NetworkPolicy, secret detection, supply chain (SBOM, dependency audit) |
| DO | Senior DevOps Engineer | CI/CD pipeline, image promotion, rendered manifests, Makefile completeness, Docker builds, observability, rollback procedures |
| QA | Senior QA Test Engineer | Test strategy, coverage targets, TDD adherence, test infrastructure, mock fidelity, spec-conformance tests, security test patterns |

## Execution Steps

### Step 1: Identify the proposal and gather artifacts

Read the active OpenSpec change directory. The standard location is `openspec/changes/<change-name>/`. Gather:

- `proposal.md` — what and why
- `design.md` — how (architecture decisions)
- `tasks.md` — implementation checklist
- `specs/<spec-name>/spec.md` — requirements and BDD scenarios

If the user specifies a different artifact set (e.g., a PR diff, a design doc outside openspec), adapt accordingly.

Also read the reference standards that reviewers validate against:

- `docs/TECH-STANDARDS.md` — the enforceable technical standards
- `docs/PRD-gitops-dashboard.md` — the product requirements (for context)
- `CLAUDE.md` — project overview and key decisions

### Step 2: Determine which reviewers are required

Use the review gate matrix from PRD §6.5:

| Proposal Type | P | FE | K8S | SEC | DO | QA |
|--------------|---|----|----|-----|----|----|
| New connector | Yes | — | Yes | Yes | Yes | Yes |
| Frontend module | — | Yes | — | Yes | — | Yes |
| API spec change | Yes | Yes | — | Yes | — | Yes |
| Deployment / manifest change | — | — | Yes | Yes | Yes | — |
| CI/CD pipeline change | — | — | — | Yes | Yes | Yes |
| Cross-cutting | Yes | Yes | Yes | Yes | Yes | Yes |
| Repo scaffolding | Yes | Yes | Yes | Yes | Yes | Yes |

Default to all six unless the proposal clearly scopes to a single layer. When in doubt, include the reviewer — the cost of an unnecessary review is low compared to a missed finding.

### Step 3: Create the output directory

```bash
mkdir -p docs/tech-review
```

### Step 4: Launch all reviewers in parallel

Spawn one sub-agent per discipline, all in a single message to maximize parallelism. Each sub-agent runs in the background.

Every sub-agent prompt MUST include:

1. **The reviewer persona** — discipline name, seniority, and focus areas from the table above
2. **The full proposal artifacts** — inline the content of proposal.md, design.md, tasks.md, and spec.md (or summarize if too large, but prefer full content)
3. **The relevant TECH-STANDARDS.md sections** — summarize the standards sections that apply to this discipline. Be specific: quote rule sets, config requirements, naming conventions, security controls. The reviewer validates against these.
4. **The output file path** — `docs/tech-review/senior-<discipline>.md`
5. **The output format** — the standard review template (see below)
6. **Explicit instruction to think deeply** — not surface-level checks. Ask: "what could go wrong?", "what's missing?", "what would I do differently?", "does this align with the standards?"

### Step 5: Report progress as reviews complete

As each background agent completes, report a brief summary to the user:
- Reviewer name
- Number of critical findings
- Top 2-3 issues
- How many reviews remain

### Step 6: Produce a consolidated summary

After all reviews complete, produce a summary table:

```
| Reviewer | Critical | Recommendations | Standards Compliance |
|----------|----------|-----------------|---------------------|
| ...      | ...      | ...             | ...                 |
```

Then list the **top recurring themes** — issues flagged by multiple reviewers carry the most weight.

## Standard Review Template

Every review file MUST use this format:

```markdown
---
reviewer: <Full Discipline Title>
proposal: <change-name>
date: <YYYY-MM-DD>
status: Review Complete
---

# <Full Discipline Title> — Review: <change-name>

## Summary

(2-3 sentence overall assessment. Be direct about severity.)

## Critical Findings

(Must-fix before implementation. Use sequential IDs with the discipline prefix.)

### Finding <PREFIX>-<N>: <Title>

- **Artifact**: (which file: design.md, tasks.md, spec.md, proposal.md)
- **Location**: (section, task number, or line reference)
- **Issue**: (what's wrong — be specific, quote the standards)
- **Impact**: (what happens if not fixed — concrete consequences)
- **Recommendation**: (specific fix — not vague "consider doing X")

## Recommendations

(Should-fix improvements. Same structure as findings.)

### Recommendation <PREFIX>-<N>: <Title>

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
| (specific standard from TECH-STANDARDS.md) | Met / Partial / Not Met / N/A | (details) |
```

### ID Prefixes

| Discipline | Prefix |
|-----------|--------|
| Senior Python Engineer | P |
| Senior Front End Developer | FE |
| Senior K8s & ArgoCD Engineer | K8S |
| Senior Security Engineer | SEC |
| Senior DevOps Engineer | DO |
| Senior QA Test Engineer | QA |

## Key Principles for Effective Reviews

These make the difference between useful reviews and noise:

- **Quote the standard**: Every finding must reference the specific TECH-STANDARDS.md section it violates. "Missing X" is weak. "TECH-STANDARDS §12.6 mandates digest-pinned base images; the proposal uses mutable `python:3.14-slim`" is actionable.
- **Concrete recommendations**: "Consider improving security" is useless. "Add `--requirepass ${REDIS_PASSWORD}` to the Redis command in docker-compose.yml" is a fix someone can apply in 30 seconds.
- **Think about propagation**: Scaffold decisions propagate to every future proposal. A wrong TTL constant in a cache.py template means every connector inherits it. Flag these amplification risks.
- **Cross-reference other disciplines**: If the Python reviewer notices a Docker issue, they should still flag it even though K8s/Security will also catch it. Redundancy across reviewers is signal, not noise — it confirms severity.
- **Severity calibration**: "Critical" means the implementation will violate an enforceable standard or create a security/reliability risk. "Recommendation" means the implementation works but is suboptimal. Don't inflate severity.

## Example Invocation

When the user says something like "run the discipline reviews on the repo-scaffolding proposal":

1. Read `openspec/changes/repo-scaffolding/` artifacts
2. Read `docs/TECH-STANDARDS.md` for reference standards
3. Determine all 6 reviewers are needed (scaffolding is cross-cutting)
4. Launch 6 sub-agents in parallel, each writing to `docs/tech-review/senior-<name>.md`
5. Report progress as each completes
6. Produce consolidated summary with recurring themes
