# Implementation Report: Container Build Pipeline v2

**Document ID:** IMPL-REPORT-2026-CI-BUILD-001
**Date:** 2026-06-28
**PRD Reference:** PRD-2026-CI-BUILD-001 v0.4.0
**Status:** All 8 phases implemented; ADO end-to-end validation pending live environment

---

## Summary

Eight sequential implementation phases delivered the full ADO shared pipeline template system defined in PRD-2026-CI-BUILD-001. The deliverable is a composable YAML template set at `platform-templates/` that tenant pipelines consume via `extends:`. All 14 capability specifications were authored and synced to `openspec/specs/` across the phases. 1,034 lines of ADO YAML were written across 11 template files.

---

## Deliverables

### Template Files

| File | Lines | Description |
|---|---|---|
| `platform-templates/container-build-v2.yml` | 163 | Base template — tenant entry point |
| `platform-templates/steps/setup.yml` | 111 | Tool resolution + parameter validation |
| `platform-templates/steps/dockerfile-lint.yml` | 43 | Hadolint Dockerfile linting |
| `platform-templates/steps/docker-build.yml` | 56 | BuildKit image build + OCI labels |
| `platform-templates/steps/sbom-sign-publish.yml` | 396 | SBOM, signing, attestation, publish, notify |
| `platform-templates/steps/runtime/go.yml` | 27 | Go pass-through validator |
| `platform-templates/steps/runtime/python.yml` | 44 | Python pass-through validator |
| `platform-templates/steps/runtime/springboot.yml` | 101 | Spring Boot two-invocation BuildKit |
| `platform-templates/steps/runtime/angular.yml` | 42 | Angular validator |
| `platform-templates/steps/runtime/react.yml` | 51 | React/Next.js validator |
| `platform-templates/test/tenant-go-dryrun.yml` | — | Tenant reference pipeline (dry-run, Go) |
| **Total** | **1,034** | |

### Capability Specifications (openspec/specs/)

14 capability specs authored across the 8 phases:

`tenant-interface` · `stage-structure` · `runtime-dispatch` · `tool-version-resolution` · `parameter-validation` · `dockerfile-linting` · `buildkit-image-build` · `go-runtime-validation` · `python-runtime-validation` · `springboot-runtime-validation` · `angular-runtime-validation` · `react-runtime-validation` · `image-signing` · `sbom-generation` · `acr-publish` · `pipeline-notify`

---

## Phase-by-Phase Summary

### Phase 1 — Template Skeleton & Tenant Interface

**Goal:** Establish the base template skeleton so tenant pipelines can load the template from day one without compilation errors.

**What was built:**
- `container-build-v2.yml` with the complete six-parameter tenant interface: `tenantName`, `appName`, `runtimeType`, `dockerfilePath`, `buildContext`, `dryRun`
- Five-stage pipeline structure: `Setup → Build → SignAndAttest → Publish → Notify` with correct `dependsOn` chains
- `dryRun` skip conditions on Stages 3 and 4
- Runtime dispatch scaffolding (`${{ if eq(parameters.runtimeType, ...) }}` blocks for all five runtimes)
- Stub versions of all referenced step templates so the template compiles without missing-file errors
- Tenant reference pipeline at `platform-templates/test/tenant-go-dryrun.yml`

**Capabilities introduced:** `tenant-interface`, `stage-structure`, `runtime-dispatch`

---

### Phase 2 — Setup & Validation

**Goal:** Replace the Setup stage stub with real tool-version resolution and parameter validation so every failure produces a clear Stage 1 error before any build work begins.

**What was built in `steps/setup.yml`:**
- Tool-version resolution from the `platform-tool-versions` ADO variable group; each tool version emitted as a named step output variable (`HADOLINT_VERSION`, `SYFT_VERSION`, `COSIGN_VERSION`, `ACR_HOST`, `NPM_REGISTRY_URL`)
- `tenantName` / `appName` validation: lowercase alphanumeric + hyphens only; fails Stage 1 with the parameter name and rejected value
- `runtimeType` allowlist check: `angular | react | springboot | python | go`; fails Stage 1 listing the invalid value and the full allowed set
- `dockerfilePath` existence assertion: resolves `<buildContext>/<dockerfilePath>` on the agent filesystem; fails Stage 1 with the resolved path
- Runtime template existence check: asserts `steps/runtime/<runtimeType>.yml` is present in the template repo

**Key design decision:** All `${{ parameters.xxx }}` template expressions are only ever in `env:` blocks of bash steps — never inline in script bodies. This convention (established here) was enforced across all subsequent phases to prevent template injection.

**Capabilities introduced:** `tool-version-resolution`, `parameter-validation`

---

### Phase 3 — Build Core

**Goal:** Replace the Dockerfile-lint and docker-build stubs with real Hadolint linting and a rootless BuildKit image build that captures the image digest for downstream stages.

**What was built:**

`steps/dockerfile-lint.yml`:
- Runs Hadolint at the pinned version from `$(resolveTools.HADOLINT_VERSION)`
- ERROR-level findings fail the build immediately
- WARNING-level findings are printed to the build log but do not block
- Picks up `.hadolint.yaml` from the build context if present

`steps/docker-build.yml`:
- Rootless BuildKit build with `DOCKER_BUILDKIT=1`
- Four OCI image labels injected: `source`, `created`, `revision`, `title`
- `GIT_COMMIT_SHA` passed as `--build-arg` (full 40-char SHA)
- ACR layer cache: `--cache-from` / `--cache-to` scoped to `<acrHost>/<tenantName>/<appName>:buildcache` with `mode=max`
- Image held locally only — not pushed to ACR
- Full image digest captured via `--iidfile` and emitted as `IMAGE_DIGEST` step output variable

**Capabilities introduced:** `dockerfile-linting`, `buildkit-image-build`

---

### Phase 4 — Go & Python Runtime Validators

**Goal:** Replace Go and Python runtime stubs with validated pass-throughs that assert project structure and detect version metadata for downstream tag generation.

**Pattern A** (used by all runtimes): the entire build toolchain runs inside the Dockerfile; no language toolchain is installed on the pipeline agent.

**`steps/runtime/go.yml`:**
- Hard assertion: `go.mod` must exist at the build context root; fails Stage 2 with a directive to the tenant if absent
- Optional version read: reads first non-empty line of `VERSION` file if present; emits as `GO_VERSION` output variable (empty if no file — version tag skipped in Phase 8)

**`steps/runtime/python.yml`:**
- Advisory check: warns if none of `requirements.txt`, `pyproject.toml`, or `poetry.lock` is found (does not block)
- Optional version read: parses `version` from `pyproject.toml` (`[project]` or `[tool.poetry]`) or `setup.cfg` (`[metadata]`); emits as `PYTHON_VERSION` output variable

**Capabilities introduced:** `go-runtime-validation`, `python-runtime-validation`

---

### Phase 5 — Spring Boot Runtime

**Goal:** Implement the Spring Boot runtime, which requires the most agent-side orchestration: a two-invocation BuildKit sequence that extracts and publishes test results before building the final image.

**`steps/runtime/springboot.yml`:**
- Build tool detection: prefers Gradle wrapper (`gradlew`); falls back to Maven (`pom.xml`); hard fails Stage 2 if neither found
- Version extraction: `version` from `build.gradle` / `gradle.properties` (Gradle) or `<version>` from `pom.xml` (Maven); emits as `SPRINGBOOT_VERSION` output variable
- Dockerfile assertion: verifies the Dockerfile contains `AS test-export`; fails Stage 2 with reference to the platform Dockerfile guide if absent
- **Two-invocation BuildKit:**
  1. `docker build --target test-export --output type=local,dest=./test-results` — extracts JUnit XML from the `test-export` stage
  2. `PublishTestResults@2` reads `./test-results/**/*.xml`; `failTaskOnFailedTests: true` hard-gates the final build on a passing test suite
  3. `docker build --target final` — BuildKit layer cache makes this near-instant; final image held locally

**Capability introduced:** `springboot-runtime-validation`

---

### Phase 6 — Angular & React Frontend Runtimes

**Goal:** Implement Angular and React validators and extend `docker-build.yml` to inject npm Azure Artifacts credentials into all BuildKit invocations, routing package resolution through the platform feed.

**`steps/runtime/angular.yml`:**
- Advisory check: warns if `package.json` is not found in the build context
- Advisory check: warns if `ng` does not appear in any `package.json` scripts entry
- Emits `ANGULAR_VERSION` from `package.json` `version` field

**`steps/runtime/react.yml`:**
- Advisory check: warns if `package.json` is not found
- Next.js detection: if `"next"` is in `package.json` dependencies, checks for `output: export` in `next.config.js` / `next.config.mjs`; warns if absent (indicates SSR which may not produce a static build artifact)
- Emits `REACT_VERSION` from `package.json` `version` field

**`steps/docker-build.yml` additions:**
- `npmRegistryUrl` parameter added; passed as `--build-arg NPM_REGISTRY=$NPM_REGISTRY_URL`
- `--secret id=npm_token,env=SYSTEM_ACCESSTOKEN` added to the BuildKit invocation — auth token is never stored in a build arg or image layer
- These flags are harmless for non-npm runtimes (Go/Python/Spring Boot Dockerfiles that do not reference `ARG NPM_REGISTRY` or the npm secret simply ignore them)

**Key design decision:** npm credentials are injected universally in `docker-build.yml` rather than conditionally in the Angular/React runtime templates. This avoids a second `docker build` invocation for frontend runtimes and keeps the credential pattern in one place.

**Capabilities introduced:** `angular-runtime-validation`, `react-runtime-validation`

---

### Phase 7 — Sign & Attest (Stage 3)

**Goal:** Implement the `signAndAttest` phase of `sbom-sign-publish.yml`, generating a CycloneDX SBOM, signing the image manifest with Cosign, attaching the SBOM as an OCI attestation, and emitting the manifest digest for Stage 4.

**`steps/sbom-sign-publish.yml` — `signAndAttest` phase:**

| Step | Tool | What it does |
|---|---|---|
| Download Syft | `curl` | Fetches Syft at pinned version from GitHub releases |
| Download Cosign | `curl` | Fetches Cosign binary at pinned version |
| Retrieve keys | `AzureKeyVault@2` | Downloads `cosign-private-key` and `cosign-public-key` from platform AKV; keys never stored in variable groups |
| Generate SBOM | `/tmp/syft` | Scans locally held Docker image; outputs `sbom.cdx.json` in CycloneDX JSON format |
| Publish SBOM | `PublishPipelineArtifact@1` | Retains SBOM as pipeline artifact `sbom-<tenant>-<app>` |
| Push + sign + attest + verify | `docker`, `cosign` | Pushes image under short-SHA tag; captures manifest digest; signs digest; attaches SBOM attestation; verifies signature; cleans up key files via `trap ... EXIT`; emits `MANIFEST_DIGEST` |

**Key design decisions:**
- Cosign cannot sign an image that isn't in a registry: Stage 3 pushes the image under the short-SHA tag first to obtain the manifest digest, then signs the digest (not a tag).
- `IMAGE_DIGEST` from `--iidfile` (config digest) ≠ manifest digest needed for Cosign. `MANIFEST_DIGEST` is captured from `docker inspect` after the ACR push.
- Key files are written to `/tmp/cosign.key` and `/tmp/cosign.pub` with a `trap 'rm -f ...' EXIT` to guarantee cleanup whether the step succeeds or fails.

**`container-build-v2.yml` additions:** SignAndAttest stage call passes `acrHost`, `imageDigest`, `syftVersion`, `cosignVersion` from upstream stage outputs.

**Capabilities introduced:** `sbom-generation`, `image-signing`

---

### Phase 8 — Publish & Notify (Stages 4 & 5)

**Goal:** Complete the delivery loop — push the full tag set to ACR, verify digest integrity, surface provenance to the PR, and notify via Teams.

**`steps/sbom-sign-publish.yml` — `publish` phase:**

| Step | What it does |
|---|---|
| `assertTags` | Computes all candidate tags (full SHA, alias, version); asserts none equals `latest`; fails before any push if assertion fails |
| `publish` | Pushes full 40-char SHA (primary tag); verifies manifest digest matches Stage 3 `MANIFEST_DIGEST`; pushes `<branch>-<short-sha>` alias tag; pushes version tag conditionally (main branch: fails if tag exists; non-main: appends `-<short-sha>` for uniqueness; skips if version empty); emits `IMAGE_REF` output variable |
| `writeProvenance` | Writes `provenance.json` using `jq`; fields: `imageRef`, `manifestDigest`, `tags[]`, `sbomArtifact`, `cosignStatus`, `pipelineRunId`, `acrRepository`, `gitCommit` |
| `PublishPipelineArtifact@1` | Publishes `provenance.json` as `provenance-<tenant>-<app>` |

**`steps/sbom-sign-publish.yml` — `notify` phase:**

| Step | What it does |
|---|---|
| `postPrComment` | Skips gracefully if `BUILD_REASON != PullRequest`; constructs Markdown summary table (status, runtime, digest, ACR ref, SBOM name, Cosign status, security scan advisory); posts via ADO REST API using `System.AccessToken` |
| `setBuildTag` | Sets ADO pipeline run build tag to `MANIFEST_DIGEST` (or `dryrun-<buildId>` on dry runs) |
| `notifyTeams` | Reads `$(TEAMS_WEBHOOK_URL)` from per-tenant variable group; skips with warning if unset; sends Adaptive Card via `curl`; non-blocking |

**`container-build-v2.yml` additions:**
- Publish job uses `${{ if/elseif }}` job-level `variables:` block to select the correct runtime version output variable at compile time and pass it as `$(RUNTIME_VERSION)` to the publish template
- Notify stage condition changed from `succeededOrFailed()` to `in(dependencies.Publish.result, 'Succeeded', 'Failed', 'SucceededWithIssues', 'Skipped')` to correctly run when Publish is skipped (dry run)

**Key implementation note:** YAML block scalar + bash heredoc content conflict prevented using Python heredocs in ADO YAML (heredoc lines at column 0 terminate the YAML literal block scalar). All JSON operations were implemented using `jq`, which is available on ADO Ubuntu agents.

**Capabilities introduced:** `acr-publish`, `pipeline-notify`

---

## Tenant Usage

A tenant pipeline that wants to use the template:

```yaml
# tenant app repo: azure-pipelines.yml
trigger:
  branches:
    include:
      - main

extends:
  template: templates/container-build-v2.yml@platform-templates
  parameters:
    tenantName: payments
    appName: payment-processor
    runtimeType: springboot
    dockerfilePath: Dockerfile
    buildContext: .
```

Six parameters exposed to tenants. All platform controls (ACR host, Cosign key, tag convention, tool versions) are locked inside the base template.

---

## Image Tagging Convention

| Tag | Format | Condition |
|---|---|---|
| Primary (immutable) | `<full-40-char-sha>` | Always; used in Kustomize manifests |
| Alias | `<branch>-<short-sha>` | Always; human navigation only |
| Version (main) | `<version>` | Non-empty version string on main branch; fails if tag already exists |
| Version (non-main) | `<version>-<short-sha>` | Non-empty version string on non-main branch; always safe |

`latest` tag is prohibited and enforced as a pipeline assertion before any push executes.

---

## Pipeline Output Artifacts

Every successful (non-dry-run) pipeline run produces:

| Artifact | Location | Contents |
|---|---|---|
| SBOM | ADO pipeline artifact `sbom-<tenant>-<app>` | CycloneDX JSON; all image packages including base |
| Cosign signature | ACR `<image>:<sha>.sig` | Detached signature of manifest digest |
| SBOM attestation | ACR `<image>:<sha>.att` | SBOM attached as OCI attestation |
| Provenance | ADO pipeline artifact `provenance-<tenant>-<app>` | JSON: digest, tags, artifact locations, git commit |
| Build tag | ADO pipeline run | Manifest digest (for run-to-image correlation) |

---

## Hard Dependencies (not in this repo)

| Dependency | Consumed by |
|---|---|
| Shared ACR | Stages 3, 4 — image push, cache |
| `platform-tool-versions` ADO variable group | Stage 1 — all tool version pins + ACR host + npm feed URL |
| Cosign key pair in Azure Key Vault | Stage 3 — signing |
| ADO platform templates repository | All tenants — `extends:` target |
| Per-tenant ADO service connection (ACR push) | Stages 3, 4 — scoped to `<tenantName>/*` |
| Per-tenant variable group `tenant-<name>-notifications` | Stage 5 — Teams webhook URL |
| `COSIGN_AKV_SERVICE_CONNECTION` + `COSIGN_KEY_VAULT_NAME` | Stage 3 — `AzureKeyVault@2` task |

---

## Open Items

### ADO End-to-End Validation (requires live environment)

All code-level implementation tasks are complete. The following validations require a running ADO organization with the hard dependencies above provisioned:

**Phase 7 tasks (10.1–10.5):**
- Queue a `dryRun=false` pipeline; confirm ACR contains `<image>:<sha>`, `<image>:<sha>.sig`, `<image>:<sha>.att`; verify `cosign verify` exits zero; confirm SBOM artifact is present
- Confirm Stage 4 does not run if `cosign verify` fails
- Confirm `dryRun=true` skips Stage 3 entirely
- Confirm Cosign key files are absent from the agent filesystem after Stage 3
- Confirm `MANIFEST_DIGEST` is accessible in Stage 4

**Phase 8 tasks (12.1–12.10):**
- Full tag push verification (full SHA, alias, version tags present in ACR)
- Digest verification passes after primary tag push
- `latest` tag absent from ACR
- `IMAGE_REF` output variable resolves in Stage 5
- `provenance-<tenant>-<app>` artifact present
- PR comment appears on triggering PR
- Teams notification received when webhook is configured
- ADO build tag set to manifest digest
- Dry run: Stages 3 and 4 skipped; PR comment indicates dry run
- Version tag collision on main branch fails with bump-version error

### Not Implemented (deferred per PRD non-goals)

- Security scanning (Trivy, Nexus, Fortify) — separate downstream pipeline
- Kustomize overlay updates after publish
- Kyverno admission enforcement policy
- Keyless Cosign signing (OIDC / Azure Workload Identity) — see PRD Appendix C

---

## Phase Archive Locations

All change records, specs, designs, and tasks are preserved at:

```
openspec/changes/archive/
  2026-06-28-phase-1-template-skeleton-tenant-interface/
  2026-06-28-phase-2-setup-validation/
  2026-06-28-phase-3-build-core/
  2026-06-28-phase-4-pattern-a-runtimes/
  2026-06-28-phase-5-pattern-b-runtimes/
  2026-06-28-phase-6-frontend-runtimes/
  2026-06-28-phase-7-sign-attest/
  2026-06-28-phase-8-publish-notify/
```
