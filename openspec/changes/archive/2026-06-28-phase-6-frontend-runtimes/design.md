## Context

`angular.yml` and `react.yml` are Phase 1 stubs executing in the Build stage's single job, after `docker-build.yml` has already run. Both are Pattern A — no compilation on the agent. The full frontend build (npm ci + framework build) runs inside the multi-stage Dockerfile.

The key constraint for frontend runtimes is npm authentication. Angular and React Dockerfiles use `--mount=type=secret,id=npm_token` to authenticate to the Azure Artifacts npm feed during `npm ci`. This secret must be present in the `docker build` invocation. Since `docker-build.yml` runs before the runtime templates, npm credentials must be injected there — not in a secondary docker build.

The architectural decision established in `docs/NPM-CACHING-PATTERN.md`:
- `--build-arg NPM_REGISTRY=...` passes the registry URL (non-secret, safe as a build arg)
- `--secret id=npm_token,env=SYSTEM_ACCESSTOKEN` passes the ADO pipeline token in-memory via BuildKit secret
- `System.AccessToken` provides read access to Azure Artifacts feeds in the same ADO organisation without additional service connections

Stakeholders: platform engineering (owns templates), Angular and React tenant teams (receive advisory checks and version tag behavior), security (npm token hygiene).

## Goals / Non-Goals

**Goals:**
- `docker-build.yml`: inject `NPM_REGISTRY` build arg and `npm_token` BuildKit secret into every docker build
- `angular.yml`: advisory checks (package.json presence, ng in scripts), version extraction from package.json, emit `ANGULAR_VERSION`
- `react.yml`: advisory checks (package.json presence, Next.js SSR detection), version extraction from package.json, emit `REACT_VERSION`
- All: no agent-side npm, no Angular CLI, no React/Next.js toolchain on agent
- All: parameter values via env: block (injection-safe)

**Non-Goals:**
- Running `npm ci`, `ng build`, `next build`, or any Node.js command on the agent
- Validating the Dockerfile structure beyond what Hadolint already enforces
- Managing Azure Artifacts feed configuration or package uploads
- Supporting Yarn, pnpm, or Bun (npm only in scope)
- Detecting all Next.js configuration formats (only `.js` and `.mjs` extensions checked)

## Decisions

### Decision 1: npm credentials injected in docker-build.yml, not in runtime templates

The `NPM_REGISTRY` build arg and `npm_token` secret are added to `docker-build.yml`'s `docker build` invocation, not to a second invocation in the runtime templates. They are passed for all runtimes — Go, Python, and Spring Boot Dockerfiles silently ignore them.

**Rationale:** `docker-build.yml` runs before runtime templates. If Angular/React ran their own `docker build`, the `docker-build.yml` build would fail for any Dockerfile that references the npm feed (no credentials provided). Adding credentials once in `docker-build.yml` avoids a second build, eliminates digest divergence, and ensures the image captured by `IMAGE_DIGEST` (from `docker-build.yml`) is the authenticated build. The NPM-CACHING-PATTERN.md establishes this as the approved pattern.

**Alternative rejected:** Runtime templates running a second `docker build` with npm credentials (similar to Spring Boot's `buildFinalImage`). Rejected because (1) it requires two full builds for Angular/React, (2) the digest from `docker-build.yml` would differ from the runtime template's build due to timestamp labels, and (3) `docker-build.yml` would already have failed before the runtime template runs.

### Decision 2: SYSTEM_ACCESSTOKEN provides npm auth — no dedicated service connection

The ADO pipeline token (`System.AccessToken`) is mapped to `SYSTEM_ACCESSTOKEN` in `docker-build.yml`'s `env:` block and passed to BuildKit via `--secret id=npm_token,env=SYSTEM_ACCESSTOKEN`. This token has read access to Azure Artifacts feeds in the same ADO organisation.

**Rationale:** No additional service connection or per-tenant configuration is needed. `System.AccessToken` is always available in ADO pipelines. The token is scoped to the pipeline's organisation, which is appropriate for reading from the shared npm feed.

### Decision 3: angular.yml and react.yml are advisory-only — no hard failures

All checks in `angular.yml` and `react.yml` emit warnings, not errors. Missing `package.json`, missing `ng` in scripts, or an SSR Next.js app without `output: export` — all produce `##vso[task.logissue type=warning]` and continue. The pipeline does not fail at the runtime template step.

**Rationale:** Pattern A means the Dockerfile is the authoritative build definition. If `package.json` is in a subdirectory within the Docker context, or if the tenant uses a non-standard Angular config, the Dockerfile build will succeed. Failing Stage 2 on an advisory check would block valid builds. The goal is visibility, not enforcement.

**Exception:** If `package.json` is absent AND version extraction fails, `ANGULAR_VERSION` / `REACT_VERSION` is emitted as empty string — this is not an error, just skips version tagging in Phase 8.

### Decision 4: Version extracted via grep/sed from package.json — no node or jq required

```bash
grep -oP '"version"\s*:\s*"\K[^"]+' package.json | head -1
```

This Perl-compatible regex extracts the value of the `"version"` key without invoking Node.js, `jq`, or any JSON parser. It's reliable for standard `package.json` files where `"version"` appears in the top-level object.

**Rationale:** Pattern A — no Node toolchain on agent. `jq` is not guaranteed on all ADO agent images. The Perl regex is available in standard `grep -P` on Linux agents and is sufficient for well-formed `package.json` files.

### Decision 5: Next.js SSR detection is heuristic — grep on next.config.js/mjs

If `next` appears in `package.json` dependencies/devDependencies, the app is assumed to be Next.js. If `next.config.js` or `next.config.mjs` does not contain the text `output.*export`, a warning is emitted that SSR apps require a Node.js runtime in the final image rather than nginx.

**Rationale:** The platform reference React Dockerfile uses nginx for SPA apps. SSR (server-rendered) Next.js apps cannot be served by nginx — they need `node server.js`. The heuristic catches the common misconfiguration where a tenant copies the SPA Dockerfile for an SSR app. False positives (e.g., a Next.js app with `output: 'export'` in an unconventional config location) produce a warning but don't fail the build.

## Risks / Trade-offs

- **`System.AccessToken` scope** — The token has access to all Azure Artifacts feeds in the ADO organisation, not just the platform npm feed. This is the standard ADO behaviour for pipeline tokens and is acceptable for a shared platform. If tighter scoping is needed, a dedicated service connection would be required.
- **npm credentials added to all runtimes** — Go, Python, and Spring Boot builds will receive `SYSTEM_ACCESSTOKEN` in their environment and pass `--build-arg NPM_REGISTRY` to docker build. The secret is ignored by BuildKit for steps that don't mount it; the build arg is ignored by Docker for Dockerfiles without `ARG NPM_REGISTRY`. No security risk, minor overhead.
- **grep-based package.json parsing** — A `"version"` key inside a nested object (e.g., in `dependencies`) could match before the top-level `version`. Mitigation: `head -1` takes the first match; in well-formed `package.json` the top-level `version` is typically the first occurrence. Document that tenants must use a standard `package.json` layout.
- **Next.js SSR detection false negatives** — If `output: 'export'` is set in a non-standard location (e.g., `next.config.ts`, a dynamic config), the check won't detect it and will emit a false SSR warning. Mitigation: warning is advisory; the Docker build and pipeline succeed regardless.

## Open Questions

None. All decisions align with the approved npm caching pattern in `docs/NPM-CACHING-PATTERN.md` and the Phase 6 scope in the implementation plan.
