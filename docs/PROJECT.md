# CI Container Build Pipeline v2 — Project Context

**PRD:** PRD-2026-CI-BUILD-001 v0.4.0 | **Owner:** Platform Engineering | **Companion:** PRD-2026-CI-RENDER-001

---

## What This Is

A shared ADO YAML pipeline template that standardises container image builds across all tenant teams on the AKS multi-tenant platform. Triggered on PR merge to main. Outputs a signed, SBOM-attested image in the shared ACR — the handoff point to a separate security scan pipeline (Trivy/Nexus/Fortify, out of scope here).

All runtimes use self-contained multi-stage Dockerfiles (Pattern A). The pipeline agent requires Docker and BuildKit only — no language toolchains are managed on the agent image.

---

## Pipeline Flow

```
PR merge to main
  → Stage 1: Setup (tool resolution, parameter validation)
  → Stage 2: Build (runtime validation, Dockerfile lint, docker build)
  → Stage 3: Sign & Attest (Syft SBOM, Cosign sign, cosign attest) — skipped if dryRun
  → Stage 4: Publish (ACR push, digest verify, provenance summary) — skipped if dryRun
  → Stage 5: Notify (PR comment, Teams webhook, trigger security scan pipeline)
```

---

## Template Structure

```
platform-templates/
  container-build-v2.yml       ← tenant entry point
  steps/
    setup.yml
    dockerfile-lint.yml
    docker-build.yml
    sbom-sign-publish.yml
    runtime/
      angular.yml              ← Pattern A: pass-through validator + npm caching injection
      react.yml                ← Pattern A: pass-through validator + npm caching injection
      springboot.yml           ← Pattern A: two-invocation BuildKit + test result export
      python.yml               ← Pattern A: pass-through validator
      go.yml                   ← Pattern A: pass-through validator (go.mod check)
```

---

## Runtime Patterns

All runtimes follow **Pattern A** — the full build toolchain runs inside a multi-stage Dockerfile. The pipeline agent only needs Docker (BuildKit). Tenant teams control all toolchain versions through their own Dockerfile.

| Runtime | runtimeType | Pre-build on Agent | Build Toolchain |
|---|---|---|---|
| Angular | `angular` | None | Node version in Dockerfile `FROM` |
| React | `react` | None | Node version in Dockerfile `FROM` |
| Spring Boot | `springboot` | None (test results extracted via BuildKit) | JDK version in Dockerfile `FROM` |
| Python | `python` | None | Python version in Dockerfile `FROM` |
| Go | `go` | None | Go version in Dockerfile `FROM` |

**Spring Boot special case:** The pipeline runs two `docker build` invocations — the first targets a `test-export` stage to extract test results to the agent for ADO test result publishing; the second builds the final image. BuildKit layer caching ensures the second invocation is near-instant.

---

## Tenant Interface

```yaml
extends:
  template: templates/container-build-v2.yml@platform-templates
  parameters:
    tenantName: payments          # lowercase alphanumeric + hyphens only
    appName: payment-processor
    runtimeType: springboot       # angular | react | springboot | python | go
    dockerfilePath: ./Dockerfile
    buildContext: .
    dryRun: false                 # true = build only, no sign/push
```

All platform controls (Cosign key, ACR endpoint, tag convention) are locked inside the base template — not overridable by tenant parameters.

---

## Image Naming & Tagging

- **Repository:** `<acr>/<tenantName>/<appName>`
- **Primary tag:** full 40-char Git SHA (immutable, used in manifests)
- **Version tag (main):** version detected from project file — fails if tag already exists (forces version bump)
- **Version tag (feature branches):** `<version>-<short-sha>` — always unique, safe to overwrite
- **Alias tag:** `<branch>-<short-sha>` (human navigation only, never in manifests)
- **`latest` is prohibited** — enforced as a pipeline assertion, not a convention

Version detection by runtime:
- Spring Boot (Maven): `<version>` element in `pom.xml`
- Spring Boot (Gradle): `version` in `build.gradle` or `gradle.properties`
- Angular / React: `version` field in `package.json`
- Python: `version` in `pyproject.toml` or `setup.cfg`
- Go: content of `VERSION` file in repo root; if absent, version tag is skipped

---

## npm Dependency Caching

Angular and React builds use the platform **Azure Artifacts npm feed** as a persistent proxy cache. The feed URL is provided by the platform via `platform-tool-versions`. The auth token is injected via BuildKit `--secret` — it never appears in image layers or build history.

See `docs/NPM-CACHING-PATTERN.md` for full implementation detail.

---

## Security & Provenance

- **Signing:** Cosign key-based (private key in Azure Key Vault, retrieved at sign time via ADO Key Vault task)
- **SBOM:** Syft in CycloneDX JSON format, attached as OCI attestation via `cosign attest`
- **ACR artifacts per build:** `<image>:<sha>` + `<image>:<sha>.sig` + `<image>:<sha>.att`
- **Digest carry-through:** every downstream step references `sha256:<digest>`, never a tag
- **No `--build-arg` secrets:** npm auth token via BuildKit `--secret`; runtime secrets via ESO + Azure Key Vault only

---

## Key Constraints

- Tool versions from `platform-tool-versions` ADO variable group — no hard-coded versions
- BuildKit required; legacy `docker build` not permitted
- Hadolint errors block build; warnings report only
- ACR layer cache via `--cache-from`/`--cache-to`, scoped per `<tenantName>/<appName>`; image NOT pushed until Stage 4
- Tenant teams have no direct ACR push credentials — all images go through this pipeline
- Agent requires Docker (BuildKit) only — no language toolchains managed on the agent image
- Tenant teams own all build toolchain versions through their Dockerfile `FROM` statements

---

## Notifications

- **PR comment:** build status, image digest, signing status, SBOM status, security scan note
- **Teams webhook:** per-tenant variable group (`tenant-<tenantName>-notifications`) holds `TEAMS_WEBHOOK_URL`
- **Non-sensitive config:** pipeline YAML variables (email distro, notify-on-success flag)

---

## Hard Dependencies

| Dependency | Status |
|---|---|
| Shared ACR provisioned (per-tenant cache scope) | Required |
| Cosign key pair in Azure Key Vault | Required |
| ADO platform templates repository | Required |
| `platform-tool-versions` variable group | Required |
| Per-tenant ADO service connection (`<tenantName>/*` push scope) | Required |
| Azure Artifacts npm feed (platform-managed) | Required for Angular/React |
| Platform agent image (Docker + BuildKit only) | Required |
| Security scan pipeline (Trivy/Nexus/Fortify) | Required for SM-7 |

---

## Out of Scope

Security scanning (Trivy, Nexus, Fortify), base image curation, Kustomize overlay updates, ACR retention policy, Kyverno admission enforcement, multi-arch builds, Cloud Native Buildpacks.

## Deferred / Future

Keyless Cosign signing (Azure Workload Identity + Fulcio), Kargo-based image promotion, ARM multi-architecture builds, self-service tenant onboarding automation, Maven/Gradle dependency caching via Azure Artifacts.
