---
name: review
description: >
  Smart review router — automatically selects the right 1-2 expert reviewers for the current
  OpenSpec proposal or code change. Examines the proposal type and affected files to pick the
  primary discipline reviewer plus a QA pass. Use this as the default review command — just say
  "review this" or "review the proposal" and it picks the right experts. This is the standard
  review workflow for all proposals and implementations in this project.
---

# Smart Review Router

Automatically select and launch the right expert reviewers for the current proposal or code change. This is the default entry point for all reviews — it examines the target, picks the primary discipline reviewer, adds a QA pass, and runs both in parallel.

## When to Use

- The user says "review this", "review the proposal", "run a review", or similar
- A new OpenSpec proposal is ready for review
- Code changes are ready for review and the user has not specified a particular discipline
- This is the standard review workflow — use it unless the user explicitly requests a specific reviewer

## Execution Steps

### Step 1: Identify the proposal or change

Read the active proposal or infer the target from context:

1. Check if the user specified a proposal name. If so, read `openspec/changes/<proposal-name>/`.
2. If not specified, list `openspec/changes/` and identify the most recent non-archived proposal.
3. If no proposal exists, look at the current Git diff or recently changed files to determine what is being reviewed.

Read the proposal artifacts:
- `proposal.md` — what and why (contains the impact section)
- `design.md` — how (architecture, file structure)
- `tasks.md` — implementation checklist
- `specs/` — BDD scenarios and requirements

### Step 2: Determine the primary reviewer

Examine the proposal's impact section, file paths, and design to classify the change:

| Signal | Primary Reviewer |
|--------|-----------------|
| Files under `connectors/` or Python source code | `/review-python` |
| Files under `frontend/` or React/TypeScript code | `/review-frontend` |
| Dockerfiles, Kubernetes manifests, Kustomize overlays, ArgoCD configs | `/review-k8s` |
| Credentials, TLS, CORS, CSP, secret management, auth | `/review-security` |
| CI/CD pipelines, Makefile, Docker build workflow, image tagging | `/review-devops` |

**Cross-cutting proposals** (touching multiple layers):
- Count the files or tasks per layer
- Pick the layer with the most changes as the primary reviewer
- If roughly equal between Python and Frontend, prefer `/review-python` (backend-first project)

**When in doubt**: If the proposal type is ambiguous, select `/review-python` for backend-leaning changes or `/review-frontend` for UI-leaning changes. The QA pass covers testing regardless.

### Step 3: Always add QA as second reviewer

Every review includes `/review-qa` as the second pass. The QA reviewer:
- Validates test strategy, coverage targets, and TDD adherence
- Checks mock fidelity and spec testability
- Is lightweight and complements any primary discipline

The only exception: if the user explicitly asks for a single specific reviewer, respect that and skip the QA pass.

### Step 4: Create the output directory

```bash
mkdir -p docs/tech-review
```

### Step 5: Launch both reviewers in parallel

Spawn two sub-agents simultaneously. Each sub-agent receives:

1. **The reviewer skill instructions** — copy the full skill body from the appropriate SKILL.md
2. **The proposal artifacts** — inline the content of proposal.md, design.md, tasks.md, and specs/
3. **The relevant TECH-STANDARDS.md sections** — summarize the standards for each discipline
4. **The output file path**:
   - Primary: `docs/tech-review/{proposal}-{discipline}-review.md`
   - QA: `docs/tech-review/{proposal}-qa-review.md`

Both agents write their review files independently. There is no dependency between them.

### Step 6: Report results

After both reviews complete, produce a consolidated summary:

```
## Review Summary: <proposal-name>

### Reviewers
| Reviewer | Output File | Critical | Recommendations |
|----------|------------|----------|-----------------|
| <Primary> | docs/tech-review/{proposal}-{discipline}-review.md | N | N |
| QA | docs/tech-review/{proposal}-qa-review.md | N | N |

### Top Issues
1. (Most important finding across both reviews)
2. (Second most important)
3. (Third most important)

### Recurring Themes
(Issues flagged by both reviewers — these carry the most weight)

### Next Steps
- Address critical findings before implementation
- Consider recommendations during implementation
- Re-review after critical findings are resolved
```

## Reviewer Selection Examples

| Proposal | Primary | Why |
|----------|---------|-----|
| `argocd-connector` | review-python | Connector = Python service |
| `dashboard-app-status` | review-frontend | Frontend module |
| `repo-scaffolding` | review-python | Cross-cutting, but more Python infra than frontend |
| `container-hardening` | review-k8s | Dockerfile and K8s manifest focus |
| `credential-rotation` | review-security | Security-sensitive credential handling |
| `ci-pipeline-setup` | review-devops | CI/CD pipeline configuration |
| `network-connector` | review-python | Connector = Python service (even though it queries K8s) |

## Edge Cases

- **Pure documentation changes**: Skip review, inform the user that doc-only changes do not require a discipline review.
- **OpenAPI spec changes only**: Use `/review-python` as primary (specs drive backend code generation).
- **Multiple proposals**: Ask the user which proposal to review. Do not review multiple proposals in one invocation.
- **No proposal found**: Ask the user to specify what to review. Do not guess.
