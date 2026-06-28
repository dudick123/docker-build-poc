# Implementation Plan: CI Container Build Pipeline v2

**Project:** PRD-2026-CI-BUILD-001 v0.4.0
**Status:** Planning
**Last Updated:** 2026-06-28

---

## Approach

The deliverable is a set of ADO shared YAML templates. The work is broken into **8 sequential phases**, each scoped to one or two tightly coupled files. This keeps each change proposal and agent context small and independently testable.

**Guiding principles:**

- One phase = one proposal = one set of tasks
- Each phase has a clear validation target (what proves it works)
- Later phases depend on earlier ones completing — do not parallelize
- All open questions are resolved — no phase is blocked

---

## Key Architectural Decisions

Decisions made during planning that differ from the original PRD v0.3.0:

- **Pattern A only** — all runtimes use self-contained multi-stage Dockerfiles. Pattern B (pre-build on agent) is eliminated. The pipeline agent needs Docker + BuildKit only.
- **Spring Boot two-invocation build** — a `test-export` stage in the Dockerfile allows BuildKit to extract test results to the agent for ADO test result publishing before the final image is built.
- **Spring Boot: Maven + Gradle** — both build tools supported; detected from project files.
- **Version-from-file tagging** — version tags derived from project metadata files (pom.xml, package.json, etc.), not Git tags. Version tag on main fails if it already exists in ACR.
- **Feature branch version tags** — `<version>-<short-sha>` suffix on non-main branches.
- **Go versioning** — `VERSION` file in repo root; if absent, version tag is skipped.
- **npm caching** — Azure Artifacts npm feed via BuildKit `--secret`. Auth token never in build args or image layers. See `docs/NPM-CACHING-PATTERN.md`.
- **Notification channels** — Teams webhook via per-tenant variable group; non-sensitive config via pipeline YAML variables.
- **ACR cache** — per-tenant scope (`<tenantName>/<appName>:buildcache`).
- **Security scan trigger** — ADO pipeline completion trigger (security scan pipeline owns the definition).
- **Scan results** — advisory-only for v1; Kyverno enforcement is the future gate.

---

## Dependency Map

```
container-build-v2.yml              ← Phase 1: all later phases depend on this
  └── steps/setup.yml               ← Phase 2
  └── steps/dockerfile-lint.yml     ← Phase 3
  └── steps/docker-build.yml        ← Phase 3
  └── steps/runtime/
        go.yml                      ← Phase 4
        python.yml                  ← Phase 4
        springboot.yml              ← Phase 5
        angular.yml                 ← Phase 6
        react.yml                   ← Phase 6
  └── steps/sbom-sign-publish.yml
        [sign & attest]             ← Phase 7
        [publish & notify]          ← Phase 8
```

---

## Phases

### Phase 1 — Template Skeleton + Tenant Interface

**File:** `container-build-v2.yml`

**Scope:**

- Full parameter block (`tenantName`, `appName`, `runtimeType`, `dockerfilePath`, `buildContext`, `dryRun`)
- Stage structure (Stages 1–5) with correct dependency conditions
- Runtime dispatch scaffolding (`${{ if }}` blocks for all 5 runtimes — stubs only, filled in by later phases)
- `dryRun` skip conditions wired on Stages 3 and 4

**Validates:** Template loads in ADO without errors; stage structure and conditions are correct; runtime dispatch stubs are present and syntactically valid.

**PRD refs:** FR-2.1, FR-2.2, section 6.2, section 7

---

### Phase 2 — Setup & Validation

**File:** `steps/setup.yml`

**Scope:**

- Resolve all tool versions from `platform-tool-versions` ADO variable group (Docker/BuildKit, Syft, Cosign, Hadolint, Azure Artifacts npm registry URL)
- Validate `tenantName` and `appName` naming convention (lowercase alphanumeric + hyphens)
- Validate `runtimeType` against allowlist (`angular`, `react`, `springboot`, `python`, `go`)
- Validate that `dockerfilePath` exists relative to build context
- Validate that the runtime step template file exists at `steps/runtime/<runtimeType>.yml`
- Fail Stage 1 with clear error messages on any validation failure

**Validates:** Invalid parameters fail Stage 1 with the correct message; valid parameters pass through cleanly.

**PRD refs:** FR-1.1, FR-1.2, FR-1.3, FR-2.3, FR-2.4, FR-2.5, FR-2.6

---

### Phase 3 — Build Core

**Files:** `steps/dockerfile-lint.yml`, `steps/docker-build.yml`

**Scope:**

- `dockerfile-lint.yml`: Hadolint execution; ERROR level findings fail the build; WARNING level findings reported only
- `docker-build.yml`: BuildKit build (rootless); OCI label injection (`image.source`, `image.created`, `image.revision`, `image.title`); `GIT_COMMIT_SHA` build arg; ACR layer cache (`--cache-from` / `--cache-to`) scoped per `<tenantName>/<appName>:buildcache`; image held locally (not pushed); full image digest captured and carried as a pipeline variable

**Validates:** `dryRun=true` against a simple Go or Python Dockerfile; image builds locally; digest variable is populated; image is not pushed to ACR.

**PRD refs:** FR-4.1–FR-4.7

---

### Phase 4 — Pattern A Runtimes (Go, Python)

**Files:** `steps/runtime/go.yml`, `steps/runtime/python.yml`

**Scope:**

- `go.yml`: validated pass-through; asserts `go.mod` exists in build context; fails Stage 1 if absent; reads `VERSION` file if present for version tag detection
- `python.yml`: validated pass-through; advisory check for `requirements.txt`, `pyproject.toml`, or `poetry.lock` (warn only, does not block); reads version from `pyproject.toml` or `setup.cfg`
- Both templates are no-ops at Stage 2 — full build runs inside the Dockerfile

**Validates:** Pattern A end-to-end with `dryRun=true`; `go.yml` fails correctly when `go.mod` is absent.

**PRD refs:** FR-3.1, Appendix B sections 15.5, 15.6

---

### Phase 5 — Spring Boot (Pattern A + Two-Invocation BuildKit)

**File:** `steps/runtime/springboot.yml`

**Scope:**

- Detect build tool: Maven (`pom.xml` present) or Gradle (`gradlew` present); fail Stage 1 if neither found
- Detect version from project file:
  - Maven: `<version>` element in `pom.xml`
  - Gradle: `version` property in `build.gradle` or `gradle.properties`
- Validate that the Dockerfile contains a `test-export` stage (`FROM scratch AS test-export`); fail Stage 2 with message directing tenant to platform reference Dockerfile if absent
- Two-invocation BuildKit sequence:
  1. `docker build --target test-export --output type=local,dest=./test-results .`
  2. ADO `PublishTestResults` task reads from `./test-results`
  3. Test failure blocks the pipeline — final image MUST NOT build with a failing test suite
  4. `docker build --target final .` — BuildKit cache hit on all layers from step 1
- Inject Azure Artifacts registry URL and npm_token secret are not applicable here — Spring Boot uses Docker layer cache for Maven/Gradle dependencies

**Validates:** Test pass → final image built; test failure → pipeline blocked at test publish step; test results visible in ADO Tests tab.

**PRD refs:** FR-3.2, FR-3.3, FR-3.4, Appendix B section 15.4

---

### Phase 6 — Pattern A Frontend Runtimes (Angular, React)

**Files:** `steps/runtime/angular.yml`, `steps/runtime/react.yml`

**Scope:**

- Both: validated pass-throughs — no pre-build steps execute on the agent
- Both: read version from `package.json` `version` field for version tag detection
- Both: inject `NPM_REGISTRY` build arg (from `platform-tool-versions`) and `npm_token` BuildKit secret (`System.AccessToken`) into `docker build` invocation — enables Azure Artifacts npm caching inside the Dockerfile
- `angular.yml`: advisory check that `package.json` exists; warn if `ng` is not referenced in build scripts
- `react.yml`: detect Next.js SSR (no `output: export` in `next.config.js`) and emit warning — SSR apps require Node runtime in final image, not nginx
- Neither template modifies the Dockerfile or build context

**Validates:** Image builds from source entirely inside Docker; npm dependencies resolved via Azure Artifacts feed; version read from `package.json`.

**PRD refs:** FR-3.1, FR-3.5, Appendix B sections 15.2, 15.3

**Prerequisite:** Azure Artifacts npm feed URL present in `platform-tool-versions`.

---

### Phase 7 — Sign & Attest

**File:** `steps/sbom-sign-publish.yml` (sign & attest portion)

**Scope:**

- Retrieve Cosign private key from Azure Key Vault via ADO Key Vault task at Stage 3 start
- Generate SBOM using Syft in CycloneDX JSON format from the locally held image
- Publish SBOM as a named pipeline artifact (retained for audit)
- Sign the image digest using Cosign (`cosign sign` on `sha256:<digest>`, not a tag)
- Attach SBOM as OCI attestation (`cosign attest`)
- Verify Cosign signature immediately after signing — block Stage 4 if verification fails
- Key MUST NOT be stored in pipeline variables or any tenant-accessible location

**Validates:** ACR artifacts present after a full run: `<image>:<sha>`, `<image>:<sha>.sig`, `<image>:<sha>.att`; signature verification passes; SBOM artifact retained in pipeline.

**PRD refs:** FR-6.1–FR-6.4, FR-7.1–FR-7.6, section 6.4, 6.6

**Hard dependencies:** Cosign key pair in Azure Key Vault (D-2); Shared ACR provisioned (D-1).

---

### Phase 8 — Publish & Notify

**File:** `steps/sbom-sign-publish.yml` (publish & notify portion)

**Scope:**

- Push image to ACR at `<tenantName>/<appName>` only after SBOM generated, signed, and signature verified
- Push primary tag (full 40-char Git SHA) and alias tag (`<branch>-<short-sha>`) in same operation
- Push version tag per version-from-file rules:
  - Main branch: push `<version-from-file>`; fail pipeline if tag already exists in ACR
  - Feature branches: push `<version-from-file>-<short-sha>`
  - Go with no `VERSION` file: skip version tag
- Assert `latest` tag is not being pushed — fail pipeline before push if tag logic produces `latest`
- Verify post-push: ACR-returned digest matches build-time digest; mismatch fails pipeline
- Publish image provenance summary as pipeline artifact and as ADO PR comment
- Output full image reference (`<acr-host>/<tenant>/<app>@sha256:<digest>`) as named ADO pipeline output variable
- Post build summary to ADO PR; include note that Trivy/Nexus/Fortify runs in separate pipeline (advisory-only)
- Send Teams webhook notification via per-tenant variable group (`tenant-<tenantName>-notifications`)
- Trigger downstream security scan pipeline via ADO pipeline completion trigger
- On `dryRun=true`: post PR comment indicating dry run, no ACR push, no scan triggered

**Validates:** SM-2 (image + sig + att in ACR), SM-3 (SHA tag present), SM-4 (no `latest`), SM-5 (provenance in PR comment); version tag fails on duplicate on main; Teams notification fires.

**PRD refs:** FR-5.1–FR-5.4, FR-8.1–FR-8.7, FR-9.1–FR-9.3, FR-10.1–FR-10.4, section 6.3

**Hard dependencies:** ADO PR comment API access (D-8); per-tenant ADO service connection (D-7); per-tenant notifications variable group.

---

## Infrastructure Prerequisites by Phase

| Phase | Prerequisite |
|---|---|
| Phase 3+ | `platform-tool-versions` variable group provisioned (D-4) |
| Phase 3+ | Shared ACR provisioned with per-tenant cache scope (D-1) |
| Phase 6+ | Azure Artifacts npm feed URL present in `platform-tool-versions` |
| Phase 7+ | Cosign key pair in Azure Key Vault (D-2) |
| Phase 8 | Per-tenant ADO service connection with `<tenantName>/*` push scope (D-7) |
| Phase 8 | ADO PR comment API access for pipeline service principal (D-8) |
| Phase 8 | Per-tenant notifications variable group (`tenant-<tenantName>-notifications`) |

---

## Validation Ladder

Each phase can be tested in isolation before the next begins:

```
Phase 1 ── template loads, stage structure visible in ADO
Phase 2 ── bad params → Stage 1 failure with correct message
Phase 3 ── dryRun=true, simple Dockerfile → local image + digest captured
Phase 4 ── dryRun=true, go/python repos → pass-through; go.mod check fires
Phase 5 ── springboot repo → test results in ADO Tests tab; test fail blocks
Phase 6 ── angular/react repo → image built entirely in Docker; npm from Azure Artifacts
Phase 7 ── full run → .sig and .att in ACR; SBOM artifact retained
Phase 8 ── full run → PR comment, version tag, output variable, scan triggered
```

---

## Rollout Strategy

**Phase 1 pilot:** single net-new tenant in the current PI. Use `dryRun=true` for initial validation before enabling full sign + publish. This tenant POC satisfies SM-8 (dryRun validated) and produces the reference implementation for existing tenant migration.

**Existing tenant migration:** separate migration guide required covering Dockerfile changes (multi-stage Pattern A), new `azure-pipelines.yml` `extends:` structure, version tag convention, and `test-export` stage requirement for Spring Boot. Migration guide is a deliverable after the POC completes.
