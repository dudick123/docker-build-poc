---
name: review-devops
description: >
  Senior DevOps Engineer review. Examines CI/CD pipeline, Makefile correctness, Docker builds,
  image tagging, observability configuration, and deployment workflow. Use for CI/CD changes,
  Makefile updates, or Docker build modifications.
---

# Senior DevOps Engineer Review

Perform a deep technical review from the perspective of a Senior DevOps Engineer. Examine either an OpenSpec proposal (proposal.md, design.md, tasks.md, specs/) or implemented CI/CD artifacts (Makefile, azure-pipelines.yml, Dockerfiles, build scripts). Validate against `docs/TECH-STANDARDS.md` sections 8, 9, 10, and 11.

## When to Use

- A proposal includes CI/CD pipeline changes, Makefile updates, or Docker build modifications
- Build, test, or deployment workflow changes need review
- The user asks for a "DevOps review", "CI/CD review", "pipeline review", or "Makefile review"
- The `/review` router delegates to this skill

## Execution Steps

### Step 1: Identify the target

Determine what to review:
- If the user specifies a proposal, read `openspec/changes/<proposal-name>/` artifacts (proposal.md, design.md, tasks.md, specs/)
- If the user specifies files, read the relevant Makefile, pipeline configs, Dockerfiles, and build scripts
- If neither is specified, check for the most recent proposal in `openspec/changes/`

### Step 2: Read the standards

Read the following from `docs/TECH-STANDARDS.md`:
- Section 8: Makefile Standards — required targets, conventions, subshell safety
- Section 9: CI/CD Pipeline Standards — Azure DevOps pipeline structure, stages, gates
- Section 10: Image Tagging and Promotion — SHA tagging, promotion workflow
- Section 11: Observability — Prometheus metrics endpoint, structured logging, health endpoints

Also read `CLAUDE.md` for project context (Azure DevOps Pipelines, AKS, environment promotion order).

### Step 3: Create the output directory

```bash
mkdir -p docs/tech-review
```

### Step 4: Perform the review

Evaluate the proposal or artifacts against these focus areas:

**Makefile Targets (23 Required)**
- Verify all 23 required targets are defined (per TECH-STANDARDS §8):
  - `help` — prints all targets with descriptions
  - `install` — `uv sync` for development dependencies
  - `install-prod` — `uv sync --frozen --no-dev` for production
  - `format` — `ruff format`
  - `format-check` — `ruff format --check`
  - `lint` — `ruff check`
  - `lint-fix` — `ruff check --fix`
  - `typecheck` — `mypy --strict`
  - `test` — `pytest` with coverage
  - `test-unit` — unit tests only
  - `test-integration` — integration tests only
  - `coverage` — coverage report
  - `security-scan` — `detect-secrets` + `uv audit`
  - `docker-build` — build container image
  - `docker-run` — run container locally
  - `docker-push` — push to registry
  - `clean` — remove build artifacts, caches, virtual environments
  - `generate-api` — generate API client code from OpenAPI specs
  - `dev` — start development server with hot reload
  - `pre-commit` — run all checks (format-check, lint, typecheck, test, security-scan)
  - `ci` — full CI pipeline (install, pre-commit, docker-build)
  - `kustomize-build` — render Kustomize manifests
  - `validate-manifests` — validate rendered manifests

**Makefile Subshell Safety**
- Loops that must fail-fast use `set -e` in subshells or `&&` chaining
- No bare `for ... ; do ... ; done` without error propagation
- `.PHONY` declared for all non-file targets
- Variables use `?=` for overridable defaults (e.g., `REGISTRY ?= myregistry.azurecr.io`)

**Docker SHA Tagging**
- Images tagged with Git SHA: `$(REGISTRY)/$(IMAGE):$(GIT_SHA)`
- No mutable tags (`latest`, `stable`) used in deployments
- `GIT_SHA` derived from `git rev-parse --short HEAD`
- Promotion copies the image (re-tags), does not rebuild
- Image manifest is identical across environments for the same SHA

**uv sync --frozen in CI**
- CI runs `uv sync --frozen` — never `uv sync` (which resolves dependencies)
- `uv.lock` committed and treated as source of truth
- Lock file changes require explicit review (dependency update PR)
- No `pip install` anywhere in CI or Dockerfiles

**azure-pipelines.yml**
- Pipeline stages: Build → Test → Security Scan → Docker Build → Deploy (per environment)
- Environment promotion order: DEV-East, DEV-West, STAGE-East, STAGE-West, PROD-East, PROD-West
- Manual approval gates before STAGE and PROD
- Pipeline variables reference Azure Key Vault secrets, not inline values
- Trigger configuration: CI on main branch, PR validation on feature branches

**Prometheus Metrics Endpoint**
- `/metrics` endpoint exposed on each connector
- Standard metrics: request count, request duration histogram, error rate, cache hit/miss ratio
- Metrics labelled with connector name, environment, and region
- Prometheus scrape annotations on Kubernetes pods

**Lockfile Strategy**
- `uv.lock` for Python dependencies — committed, frozen in CI
- `package-lock.json` or `pnpm-lock.yaml` for frontend — committed, frozen in CI
- Lock file updates are separate PRs with dependency audit
- No lock file regeneration during builds

**Build Reproducibility**
- Same Git SHA produces identical artifacts regardless of when or where built
- No network fetches during Docker build (dependencies cached in builder stage)
- Build args limited to `GIT_SHA` and `BUILD_DATE` — no environment-specific args
- Multi-platform builds if required (amd64 for AKS)

### Step 5: Write the review

Write the review to `docs/tech-review/{proposal}-devops-review.md` using this exact format:

```markdown
---
reviewer: Senior DevOps Engineer
proposal: <proposal-name>
date: <YYYY-MM-DD>
status: Review Complete
---

# Senior DevOps Engineer — Review: <proposal-name>

## Summary

(2-3 sentence overall assessment. Be direct about severity.)

## Critical Findings

(Must-fix items. Use DO- prefix for finding IDs.)

### Finding DO-<N>: <Title>

- **Artifact**: (which file: design.md, tasks.md, Makefile, azure-pipelines.yml, Dockerfile, etc.)
- **Location**: (section, task number, target name, or line reference)
- **Issue**: (what is wrong — quote the specific TECH-STANDARDS section violated)
- **Impact**: (concrete consequences if not fixed)
- **Recommendation**: (specific fix, not vague guidance)

## Recommendations

(Should-fix improvements. Same structure as findings.)

### Recommendation DO-<N>: <Title>

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
| TECH-STANDARDS §8.1 — 23 required Makefile targets | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §8.2 — Subshell loop safety | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §8.3 — .PHONY declarations | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §9.1 — Pipeline stages (Build→Test→Scan→Deploy) | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §9.2 — Environment promotion order | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §9.3 — Manual approval gates | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §10.1 — Docker SHA tagging | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §10.2 — No mutable tags in deployments | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §10.3 — Promotion re-tags, no rebuild | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §11.1 — /metrics Prometheus endpoint | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §11.2 — Structured JSON logging | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS — uv sync --frozen in CI | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS — Build reproducibility | Met / Partial / Not Met / N/A | |
```

### Step 6: Report results

After writing the review file, report to the user:
- Number of critical findings and recommendations
- Top 2-3 most important issues
- Overall standards compliance assessment
- Path to the full review file
