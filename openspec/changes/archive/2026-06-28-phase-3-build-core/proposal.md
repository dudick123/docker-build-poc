## Why

Phases 1 and 2 established the pipeline skeleton and parameter validation; every tenant pipeline now reaches the Build stage with verified inputs but no real work happens there — both `steps/dockerfile-lint.yml` and `steps/docker-build.yml` are stubs. Phase 3 replaces those stubs with the full build core: Hadolint Dockerfile linting followed by a rootless BuildKit image build that captures the full image digest for all downstream signing, publishing, and verification steps.

## What Changes

- **Replace stub** `platform-templates/steps/dockerfile-lint.yml` with Hadolint execution:
  - Run Hadolint at the version pinned in `$(Setup.resolveTools.HADOLINT_VERSION)` via the Phase 2 output variable
  - ERROR-level findings fail the build immediately
  - WARNING-level findings are printed to the build log but do not block
  - Hadolint configuration sourced from `.hadolint.yaml` in the build context if present; otherwise platform defaults apply

- **Replace stub** `platform-templates/steps/docker-build.yml` with a rootless BuildKit build:
  - Build with `DOCKER_BUILDKIT=1` using the version from `$(Setup.resolveTools.DOCKER_BUILDKIT_VERSION)`
  - Inject four OCI-standard image labels: `org.opencontainers.image.source`, `org.opencontainers.image.created`, `org.opencontainers.image.revision`, `org.opencontainers.image.title`
  - Pass `GIT_COMMIT_SHA` as a `--build-arg` (full 40-character SHA from `$(Build.SourceVersion)`)
  - Enable ACR layer cache: `--cache-from` and `--cache-to` scoped to `<acrHost>/<tenantName>/<appName>:buildcache`
  - Hold image locally only — do NOT push to ACR (push is Phase 8)
  - Capture the full `sha256:<digest>` of the built image and emit it as a step output variable `IMAGE_DIGEST` for downstream sign/publish steps

## Capabilities

### New Capabilities

- `dockerfile-linting`: Hadolint execution against the tenant Dockerfile — which finding severity levels block vs. warn, how the Hadolint version is consumed, and `.hadolint.yaml` config file discovery.
- `buildkit-image-build`: The rootless BuildKit build invocation — OCI label injection, `GIT_COMMIT_SHA` build arg, ACR layer cache configuration, local-only image hold, and digest capture as a pipeline output variable.

### Modified Capabilities

## Impact

- **Modified files:** `platform-templates/steps/dockerfile-lint.yml`, `platform-templates/steps/docker-build.yml`
- **No other template files change** — the base template (`container-build-v2.yml`) already calls both step templates; no wiring changes needed
- **Consumes Phase 2 outputs:** `$(stageDependencies.Setup.Setup.outputs['resolveTools.DOCKER_BUILDKIT_VERSION'])` and `$(stageDependencies.Setup.Setup.outputs['resolveTools.HADOLINT_VERSION'])` — Phase 2 must be deployed before Phase 3 is testable
- **Produces for Phase 7:** `IMAGE_DIGEST` step output variable — Phases 7 and 8 (sign, attest, publish) depend on this value being set and accurate
- **Infrastructure prerequisite:** Shared ACR must be provisioned and the per-tenant service connection must have `<tenantName>/<appName>:buildcache` cache scope for layer caching (D-1 from implementation plan); build succeeds without cache on first run
- **Agent requirement:** Agent image must have Docker with BuildKit support (rootless mode); no additional toolchain needed for Pattern A runtimes
