---
name: review-k8s
description: >
  Senior Kubernetes and ArgoCD Engineer review. Examines Dockerfiles, container hardening, health probes,
  resource sizing, deployment strategy, Kustomize manifests, ArgoCD RBAC and sync policy. Use for
  infrastructure proposals, Dockerfile changes, or Kubernetes manifest changes.
---

# Senior Kubernetes & ArgoCD Engineer Review

Perform a deep technical review from the perspective of a Senior Kubernetes and ArgoCD Engineer. Examine either an OpenSpec proposal (proposal.md, design.md, tasks.md, specs/) or implemented infrastructure artifacts (Dockerfiles, K8s manifests, Kustomize overlays, ArgoCD configs). Validate against `docs/TECH-STANDARDS.md` sections 5, 6, and 12.

## When to Use

- A proposal includes Dockerfiles, Kubernetes manifests, or ArgoCD configuration
- Infrastructure or deployment changes need review
- The user asks for a "K8s review", "Kubernetes review", "infra review", or "ArgoCD review"
- The `/review` router delegates to this skill

## Execution Steps

### Step 1: Identify the target

Determine what to review:
- If the user specifies a proposal, read `openspec/changes/<proposal-name>/` artifacts (proposal.md, design.md, tasks.md, specs/)
- If the user specifies files, read the relevant Dockerfiles, manifests, and Kustomize configs
- If neither is specified, check for the most recent proposal in `openspec/changes/`

### Step 2: Read the standards

Read the following from `docs/TECH-STANDARDS.md`:
- Section 5: Docker and Container Standards — multi-stage builds, base images, non-root, labels
- Section 6: Kubernetes Deployment Standards — probes, resources, strategy, PDB, HPA
- Section 12: Security and Hardening — container security context, ArgoCD RBAC

Also read `CLAUDE.md` for project context (Kustomize not Helm, AKS clusters, environment promotion order).

### Step 3: Create the output directory

```bash
mkdir -p docs/tech-review
```

### Step 4: Perform the review

Evaluate the proposal or artifacts against these focus areas:

**Multi-Stage Dockerfiles**
- Builder stage and runtime stage separated
- Runtime image is minimal (distroless or slim variant)
- `uv.lock` and `pyproject.toml` COPY'd before source code for layer caching
- `uv sync --frozen --no-dev` in build stage — no dev dependencies in production image
- Final `COPY --from=builder` copies only the virtual environment and application code

**Base Image Pinning**
- Base images pinned by digest, not mutable tag (e.g., `python:3.14-slim@sha256:...`)
- No `latest` tags anywhere
- Base image digest documented in a comment with the human-readable tag for reference

**Non-Root Container Execution**
- `USER 1000:1000` set in Dockerfile
- Application files owned by UID 1000
- No `chmod 777` or world-writable directories
- `runAsNonRoot: true` in Kubernetes securityContext
- `readOnlyRootFilesystem: true` where possible

**Health Probes**
- `/healthz` endpoint for liveness probe
- `/readyz` endpoint for readiness probe (checks downstream dependencies: Redis, upstream API reachability)
- Startup probe configured to avoid killing slow-starting containers
- Probe timing: `initialDelaySeconds`, `periodSeconds`, `failureThreshold` appropriately set
- Liveness probe does NOT check downstream dependencies (to avoid cascading restarts)

**Probe Timing**
- Startup probe: generous `failureThreshold * periodSeconds` to handle cold start
- Liveness probe: `periodSeconds: 10`, `failureThreshold: 3` (30s to detect stuck process)
- Readiness probe: `periodSeconds: 5`, `failureThreshold: 2` (10s to remove from service)

**Graceful Shutdown**
- `terminationGracePeriodSeconds` set (default 30s, adjust per connector)
- FastAPI `--timeout-graceful-shutdown` flag configured to match
- SIGTERM handler drains in-flight requests before exit
- PreStop hook if needed for load balancer drain time

**Deployment Strategy**
- `maxSurge: 1` and `maxUnavailable: 0` for zero-downtime rolling updates
- PodDisruptionBudget (PDB) configured: `minAvailable: 1` for single-replica connectors
- Deployments use `revisionHistoryLimit` to control ReplicaSet retention

**HPA (Horizontal Pod Autoscaler)**
- HPA configured with CPU and memory targets
- `minReplicas` and `maxReplicas` set appropriately per environment
- Scaling behaviour tuned to avoid flapping (stabilization windows)

**Kustomize Structure**
- `base/` contains common resources
- `overlays/{dev,stage,prod}/` contain environment-specific patches
- No Helm charts — Kustomize only (project decision in CLAUDE.md)
- `kustomization.yaml` uses `resources:`, `patches:`, and `configMapGenerator:`

**ArgoCD Configuration**
- Application manifests reference the correct repo, path, and target revision
- Sync policy: `automated` with `selfHeal: true` and `prune: true` for non-prod
- RBAC: connectors have read-only ArgoCD tokens scoped to their projects
- Sync waves used for dependency ordering if needed

**Resource Sizing**
- CPU and memory requests and limits set on all containers
- Requests based on observed baseline, limits provide headroom
- No unbounded resources (missing limits)
- Resource quotas considered for namespace-level governance

### Step 5: Write the review

Write the review to `docs/tech-review/{proposal}-k8s-review.md` using this exact format:

```markdown
---
reviewer: Senior Kubernetes & ArgoCD Engineer
proposal: <proposal-name>
date: <YYYY-MM-DD>
status: Review Complete
---

# Senior Kubernetes & ArgoCD Engineer — Review: <proposal-name>

## Summary

(2-3 sentence overall assessment. Be direct about severity.)

## Critical Findings

(Must-fix items. Use K8S- prefix for finding IDs.)

### Finding K8S-<N>: <Title>

- **Artifact**: (which file: design.md, tasks.md, spec.md, Dockerfile, deployment.yaml, etc.)
- **Location**: (section, task number, or line reference)
- **Issue**: (what is wrong — quote the specific TECH-STANDARDS section violated)
- **Impact**: (concrete consequences if not fixed)
- **Recommendation**: (specific fix, not vague guidance)

## Recommendations

(Should-fix improvements. Same structure as findings.)

### Recommendation K8S-<N>: <Title>

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
| TECH-STANDARDS §5.1 — Multi-stage Dockerfile | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §5.2 — Digest-pinned base images | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §5.3 — Non-root UID 1000 | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §5.4 — uv.lock COPY for layer caching | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §6.1 — /healthz + /readyz probes | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §6.2 — Startup/liveness/readiness timing | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §6.3 — terminationGracePeriodSeconds | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §6.4 — maxSurge/maxUnavailable strategy | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §6.5 — PodDisruptionBudget | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §6.6 — HPA configuration | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §6.7 — Resource requests and limits | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §12 — ArgoCD RBAC and sync policy | Met / Partial / Not Met / N/A | |
```

### Step 6: Report results

After writing the review file, report to the user:
- Number of critical findings and recommendations
- Top 2-3 most important issues
- Overall standards compliance assessment
- Path to the full review file
