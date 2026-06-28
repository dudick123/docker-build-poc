# CI Container Build Pipeline v2 — Platform Architecture

**Document ID:** ARCH-2026-CI-BUILD-001
**Audience:** Platform Engineering
**Status:** Draft
**Version:** 1.0.0
**Last Updated:** 2026-06-28
**Companion PRD:** PRD-2026-CI-BUILD-001 v0.4.0

---

## Table of Contents

1. [Purpose and Scope](#1-purpose-and-scope)
2. [System Overview](#2-system-overview)
3. [Infrastructure Components](#3-infrastructure-components)
4. [Pipeline Architecture](#4-pipeline-architecture)
5. [Template Dispatch Model](#5-template-dispatch-model)
6. [Build Pattern: Multi-Stage Dockerfile (Pattern A)](#6-build-pattern-multi-stage-dockerfile-pattern-a)
7. [Spring Boot: Two-Invocation BuildKit](#7-spring-boot-two-invocation-buildkit)
8. [npm Dependency Caching via Azure Artifacts](#8-npm-dependency-caching-via-azure-artifacts)
9. [Image Naming, Tagging, and Versioning](#9-image-naming-tagging-and-versioning)
10. [Security and Provenance Model](#10-security-and-provenance-model)
11. [Tooling Reference](#11-tooling-reference)
12. [ADO Configuration Reference](#12-ado-configuration-reference)
13. [Notification Architecture](#13-notification-architecture)
14. [Security Scan Handoff](#14-security-scan-handoff)
15. [ACR Repository Structure](#15-acr-repository-structure)
16. [Tenant Onboarding Checklist](#16-tenant-onboarding-checklist)
17. [Operational Considerations](#17-operational-considerations)
18. [Future State](#18-future-state)

---

## 1. Purpose and Scope

This document describes the platform architecture for the v2 CI container build pipeline. It is written for **platform engineers** responsible for building, operating, and maintaining the shared ADO pipeline template infrastructure.

It covers infrastructure topology, pipeline template design, tooling integration, security controls, and operational responsibilities. It does not cover tenant Dockerfile authoring guidance — that belongs in the tenant-facing onboarding documentation (to be authored after the POC).

**What this pipeline delivers:**

- A signed, SBOM-attested OCI image in the shared Azure Container Registry (ACR)
- A clean handoff to the downstream security scan pipeline via ADO pipeline completion trigger
- Cryptographic build provenance traceable from a running container back to a source commit and pipeline run

**What this pipeline does not do:**

- Vulnerability scanning (Trivy, Nexus, Fortify) — separate security scan pipeline
- Kustomize config repo updates — manual for v1
- Runtime admission enforcement — Kyverno (planned, separate PRD)
- Base image curation or enforcement — deferred

---

## 2. System Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         ADO Source Repository (tenant)                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────────┐  │
│  │ Source Code  │  │  Dockerfile  │  │  azure-pipelines.yml         │  │
│  │              │  │ (multi-stage)│  │  extends: container-build-v2 │  │
│  └──────────────┘  └──────────────┘  └──────────────────────────────┘  │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │ PR merge to main
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     ADO Platform Templates Repository                   │
│                                                                         │
│  container-build-v2.yml  ──► setup.yml                                 │
│                          ──► dockerfile-lint.yml  (Hadolint)           │
│                          ──► docker-build.yml     (BuildKit)           │
│                          ──► runtime/<type>.yml   (validator)          │
│                          ──► sbom-sign-publish.yml (Syft + Cosign)    │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │ runs on
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│            ADO Self-Hosted Ephemeral Agent (Docker + BuildKit only)     │
│                                                                         │
│   Stage 1: Setup & Validate                                            │
│   Stage 2: Lint → Build (image held locally)                           │
│   Stage 3: SBOM → Sign → Verify          ◄── Azure Key Vault           │
│   Stage 4: Push → Tag → Provenance       ──► Azure Container Registry  │
│   Stage 5: PR Comment → Teams → Trigger  ──► Security Scan Pipeline    │
└─────────────────────────────────────────────────────────────────────────┘
                                │
              ┌─────────────────┼──────────────────┐
              ▼                 ▼                  ▼
    ┌──────────────┐  ┌──────────────────┐  ┌──────────────────┐
    │    Shared    │  │  Security Scan   │  │  ADO PR Comment  │
    │    ACR       │  │  Pipeline (ADO)  │  │  + Teams Notify  │
    │  (per-tenant │  │  Trivy / Nexus   │  │                  │
    │   namespace) │  │  Fortify         │  │                  │
    └──────────────┘  └──────────────────┘  └──────────────────┘
```

---

## 3. Infrastructure Components

### 3.1 Azure Container Registry (ACR)

Single shared ACR instance for all tenant builds across all environments.

| Concern | Configuration |
|---|---|
| Repository naming | `<tenantName>/<appName>` — enforced by pipeline |
| Layer cache storage | Per-tenant scope: `<tenantName>/<appName>:buildcache` |
| Push permissions | Per-tenant service connection scoped to `<tenantName>/*` only |
| Registry-wide admin | Platform engineering only — no tenant access |
| ACR SKU | Premium required for geo-replication and cache storage support |

**ACR artifacts per successful build:**

```
<acr>/<tenantName>/<appName>:<40-char-git-sha>      ← primary image
<acr>/<tenantName>/<appName>:<version>               ← version tag (main only)
<acr>/<tenantName>/<appName>:<branch>-<short-sha>    ← alias tag
<acr>/<tenantName>/<appName>:<sha>.sig               ← Cosign signature
<acr>/<tenantName>/<appName>:<sha>.att               ← SBOM attestation
<acr>/<tenantName>/<appName>:buildcache              ← BuildKit layer cache
```

### 3.2 Azure Key Vault

Holds the platform-managed Cosign signing key. Accessed at sign time only via the ADO Key Vault task.

| Secret | Purpose | Access |
|---|---|---|
| `cosign-private-key` | Signs image digests via `cosign sign` | ADO pipeline service principal (read-only, time-scoped) |
| `cosign-private-key-password` | Key decryption password if key is encrypted | Same as above |

The Cosign **public key** is distributed separately as a platform-managed Kubernetes ConfigMap or Kyverno ClusterPolicy. The public key is used by Kyverno at admission time to verify image signatures.

**Key rotation procedure:** platform engineering rotates the key in AKV, redistributes the public key resource, and bumps the platform template version. Tenant builds pick up the new key automatically on their next run.

### 3.3 Azure Artifacts npm Feed

Platform-managed npm proxy feed. Caches packages from the public npm registry to eliminate external network dependency during Angular and React builds.

| Property | Value |
|---|---|
| Feed type | Upstream proxy to `registry.npmjs.org` |
| Feed URL | Stored in `platform-tool-versions` as `NPM_REGISTRY_URL` |
| Authentication | `System.AccessToken` (ADO pipeline token — no additional configuration required within the same ADO org) |
| Scope | Organisation-wide; accessible to all ADO pipeline agents in the org |

### 3.4 ADO Platform Templates Repository

Hosts all shared pipeline template YAML. Tenant repositories reference it via `extends:`. Platform engineering controls write access. Tenant teams have read-only access.

```
platform-templates/           ← this repository
  container-build-v2.yml
  steps/
    setup.yml
    dockerfile-lint.yml
    docker-build.yml
    sbom-sign-publish.yml
    runtime/
      angular.yml
      react.yml
      springboot.yml
      python.yml
      go.yml
```

Template versioning is managed via Git tags on this repository. Tenants reference by tag (e.g. `@platform-templates@v2.1.0`) to pin to a stable version. Security-critical updates to the base template require a new version tag and a migration notice to tenants.

### 3.5 ADO Variable Groups

| Variable Group | Scope | Contents |
|---|---|---|
| `platform-tool-versions` | Organisation-wide | Tool versions (BuildKit, Syft, Cosign, Hadolint), Azure Artifacts npm feed URL, Gradle cache endpoint |
| `tenant-<tenantName>-notifications` | Per-tenant | `TEAMS_WEBHOOK_URL` (sensitive — stored as secret variable) |

### 3.6 ADO Self-Hosted Ephemeral Agents

Agents are ephemeral — destroyed after each pipeline run. No agent-local state persists between builds.

| Requirement | Detail |
|---|---|
| Docker | BuildKit mode enabled (`DOCKER_BUILDKIT=1`) |
| BuildKit | Rootless mode where possible (`rootlesskit`) |
| Outbound access | ACR, Azure Key Vault, Azure Artifacts feed, ADO APIs |
| Language toolchains | **None required** — all build toolchains run inside Docker |
| Disk | Sufficient for local image hold between Stage 2 and Stage 4 |

---

## 4. Pipeline Architecture

### 4.1 Stage Flow

```
                   ┌──────────────────────────────────────┐
                   │           Stage 1: Setup             │
                   │                                      │
                   │  • Resolve tool versions from        │
                   │    platform-tool-versions VG         │
                   │  • Validate tenant parameters        │
                   │  • Validate runtimeType allowlist    │
                   │  • Validate Dockerfile exists        │
                   │  • Validate runtime template exists  │
                   │  • Detect version from project file  │
                   └──────────────────┬───────────────────┘
                                      │ succeeded
                   ┌──────────────────▼───────────────────┐
                   │           Stage 2: Build             │
                   │                                      │
                   │  • Runtime template validation       │
                   │  • Hadolint Dockerfile lint          │
                   │  • docker build (BuildKit)           │
                   │    ├─ Spring Boot: 2-invocation      │
                   │    │   (test export + final image)   │
                   │    └─ All others: single invocation  │
                   │  • Capture image digest              │
                   └──────────────────┬───────────────────┘
                                      │ succeeded
                   ┌──────────────────▼───────────────────┐
                   │      Stage 3: Sign & Attest          │  ← skipped if dryRun=true
                   │                                      │
                   │  • Retrieve Cosign key from AKV      │
                   │  • syft scan → CycloneDX SBOM        │
                   │  • cosign sign <image-digest>        │
                   │  • cosign attest (attach SBOM)       │
                   │  • cosign verify (block if fails)    │
                   └──────────────────┬───────────────────┘
                                      │ succeeded
                   ┌──────────────────▼───────────────────┐
                   │          Stage 4: Publish            │  ← skipped if dryRun=true
                   │                                      │
                   │  • Assert version tag not exists     │
                   │    (main branch only)                │
                   │  • Assert 'latest' not in tags       │
                   │  • docker push (all tags)            │
                   │  • Verify ACR digest matches build   │
                   │  • Publish provenance summary        │
                   │  • Set output variable (digest ref)  │
                   └──────────────────┬───────────────────┘
                                      │ always (post-pipeline)
                   ┌──────────────────▼───────────────────┐
                   │          Stage 5: Notify             │
                   │                                      │
                   │  • Post PR comment (build summary)   │
                   │  • Send Teams webhook notification   │
                   │  • Trigger security scan pipeline    │
                   │  • Publish ADO build tag (digest)    │
                   └──────────────────────────────────────┘
```

### 4.2 dryRun Mode

When `dryRun=true` is set in the tenant `azure-pipelines.yml`:

- Stages 1 and 2 execute in full (validation, lint, build)
- Stages 3 and 4 are skipped (no sign, no ACR push)
- Stage 5 fires but the PR comment and Teams notification explicitly indicate dry run status
- The security scan pipeline is NOT triggered
- The output variable is populated with the local image digest for inspection

dryRun mode is the recommended path for new tenant onboarding and Dockerfile validation.

### 4.3 Failure Behaviour

| Stage | Failure | Behaviour |
|---|---|---|
| Stage 1 | Invalid parameter | Fail with specific message; no build resources consumed |
| Stage 1 | Missing Dockerfile | Fail with file path in error message |
| Stage 1 | Invalid runtimeType | Fail listing valid values |
| Stage 1 | Version not detectable | Fail with guidance on project file format |
| Stage 2 | Hadolint ERROR | Fail; WARNING is reported only |
| Stage 2 | Spring Boot test failure | Fail at test publish; no final image built |
| Stage 2 | docker build failure | Fail with BuildKit output |
| Stage 3 | Cosign verify fails | Fail; image remains local and is NOT pushed |
| Stage 4 | Version tag collision (main) | Fail before push; no image in ACR |
| Stage 4 | Digest mismatch post-push | Fail; alert triggered |
| Stage 5 | Always runs | Reports failure context in PR comment and Teams |

---

## 5. Template Dispatch Model

The base template (`container-build-v2.yml`) dispatches to runtime step templates using ADO conditional template expressions. This is the only extensibility mechanism — platform engineering adds runtimes here, tenant teams do not.

```yaml
# container-build-v2.yml (simplified)
parameters:
  - name: runtimeType
    type: string
    values: [angular, react, springboot, python, go]

stages:
  - stage: Build
    jobs:
      - job: ValidateRuntime
        steps:
          - ${{ if eq(parameters.runtimeType, 'angular') }}:
            - template: steps/runtime/angular.yml
          - ${{ if eq(parameters.runtimeType, 'react') }}:
            - template: steps/runtime/react.yml
          - ${{ if eq(parameters.runtimeType, 'springboot') }}:
            - template: steps/runtime/springboot.yml
          - ${{ if eq(parameters.runtimeType, 'python') }}:
            - template: steps/runtime/python.yml
          - ${{ if eq(parameters.runtimeType, 'go') }}:
            - template: steps/runtime/go.yml
```

```
Tenant azure-pipelines.yml
        │
        │  extends: container-build-v2.yml@platform-templates
        │  parameters:
        │    runtimeType: springboot
        ▼
  container-build-v2.yml
        │
        │  ${{ if eq(parameters.runtimeType, 'springboot') }}
        ▼
  steps/runtime/springboot.yml
  (validation + version detection + two-invocation build)
        │
        ▼
  steps/dockerfile-lint.yml   ← Hadolint
  steps/docker-build.yml      ← BuildKit (handles 2-invocation for Spring Boot)
  steps/sbom-sign-publish.yml ← Syft → Cosign → ACR push → Notify
```

**Adding a new runtime (platform engineering process):**

1. Author `steps/runtime/<newRuntime>.yml` with validation logic and version detection
2. Add `${{ if eq(parameters.runtimeType, '<newRuntime>') }}` dispatch block in `container-build-v2.yml`
3. Add `<newRuntime>` to the `values:` allowlist in the parameter definition
4. Update `platform-tool-versions` if new toolchain pinning is required
5. Update the PRD (section 8.2, Appendix B), this document, and the tenant-facing guide
6. Tag a new version of the platform templates repository

---

## 6. Build Pattern: Multi-Stage Dockerfile (Pattern A)

All five supported runtimes use multi-stage Dockerfiles. The pipeline agent runs `docker build` only — no language toolchains exist on the agent. Tenants own their build toolchain version through the `FROM` statement in their Dockerfile.

### 6.1 Why Pattern A Only

The original design considered a two-pattern approach (Pattern A for Go/Python, Pattern B with agent-side pre-build for Angular/React/Spring Boot). Pattern B was eliminated for the following reasons:

- Agent image maintenance (Node.js LTS, JDK version management) requires platform engineering capacity that is not available
- Ephemeral agents lose all agent-local caches between runs, negating the primary caching benefit of Pattern B
- Multi-stage Dockerfiles are more portable and self-documenting — the Dockerfile is the complete specification of the build

### 6.2 BuildKit Configuration

BuildKit is required for all builds. Legacy `docker build` without BuildKit is not permitted.

```bash
# Required environment variable on the agent
export DOCKER_BUILDKIT=1

# Standard build invocation
docker build \
  --build-arg GIT_COMMIT_SHA=$(Build.SourceVersion) \
  --build-arg NPM_REGISTRY=$(NPM_REGISTRY_URL) \          # Angular/React only
  --secret id=npm_token,env=AZURE_ARTIFACTS_TOKEN \        # Angular/React only
  --cache-from type=registry,ref=$(ACR_HOST)/$(tenantName)/$(appName):buildcache \
  --cache-to   type=registry,ref=$(ACR_HOST)/$(tenantName)/$(appName):buildcache,mode=max \
  --label org.opencontainers.image.source=$(Build.Repository.Uri) \
  --label org.opencontainers.image.created=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --label org.opencontainers.image.revision=$(Build.SourceVersion) \
  --label org.opencontainers.image.title=$(appName) \
  --iidfile image-digest.txt \
  --file $(dockerfilePath) \
  $(buildContext)
```

The `--iidfile` flag writes the full image digest (`sha256:...`) to a file. This digest is the canonical reference carried through all downstream stages.

### 6.3 ACR Layer Cache

Layer caching is scoped per `<tenantName>/<appName>` to prevent cross-tenant cache access.

```
Cache ref: <acr>/<tenantName>/<appName>:buildcache

--cache-from: pulls existing cache layers from ACR before build
--cache-to:   pushes updated cache layers to ACR after build (mode=max caches all layers, not just final)
```

On a cache hit, only changed layers are rebuilt. The `npm ci`, `pip install`, and `go mod download` layers cache naturally when their respective lock files (`package-lock.json`, `requirements.txt`, `go.sum`) are unchanged.

### 6.4 OCI Labels and Build Args

Every image receives the following labels regardless of runtime:

| Label | Value | Source |
|---|---|---|
| `org.opencontainers.image.source` | Repository URI | `$(Build.Repository.Uri)` |
| `org.opencontainers.image.created` | Build timestamp (UTC) | `date -u` |
| `org.opencontainers.image.revision` | Full 40-char Git SHA | `$(Build.SourceVersion)` |
| `org.opencontainers.image.title` | Application name | `$(appName)` |

The Git SHA is also injected as `--build-arg GIT_COMMIT_SHA=<sha>`. Tenant Dockerfiles may use this to embed the commit SHA in the application binary or static assets.

---

## 7. Spring Boot: Two-Invocation BuildKit

Spring Boot builds require ADO test result publishing. Since the build runs entirely inside Docker, test results are extracted to the agent filesystem using BuildKit's `--output` mechanism before the final image is built.

### 7.1 Flow

```
springboot.yml step template
        │
        ├─ Stage 1: Detect Maven or Gradle
        │    ├─ pom.xml found → Maven mode; read <version> element
        │    ├─ gradlew found → Gradle mode; read version from build.gradle
        │    └─ neither found → fail Stage 1
        │
        ├─ Stage 1: Validate test-export stage exists in Dockerfile
        │    └─ 'FROM scratch AS test-export' not found → fail Stage 2
        │
        ├─ docker build --target test-export --output type=local,dest=./test-results .
        │    └─ BuildKit runs: builder stage (gradle test / mvnw verify)
        │                      test-export stage (scratch, copies test XML files)
        │                      outputs: ./test-results/*.xml on agent filesystem
        │
        ├─ PublishTestResults (ADO task)
        │    ├─ reads ./test-results
        │    ├─ test failure → FAIL Stage 2 (no final image built)
        │    └─ test pass → continue
        │
        └─ docker build --target final .
             └─ BuildKit: all layers from previous invocation are cache hits
                          only final stage assembly runs
                          image digest captured via --iidfile
```

### 7.2 Required Dockerfile Structure

Both Maven and Gradle Spring Boot Dockerfiles MUST include a `test-export` stage. The stage name `test-export` is a platform convention — the step template validates it by name.

**Gradle:**

```dockerfile
FROM gradle:8-jdk21 AS builder
WORKDIR /app
COPY build.gradle settings.gradle gradlew ./
COPY gradle ./gradle
RUN ./gradlew dependencies --no-daemon    # cache dependency layer separately
COPY src ./src
RUN ./gradlew test bootJar --no-daemon

FROM scratch AS test-export
COPY --from=builder /app/build/test-results ./

FROM eclipse-temurin:21-jre-alpine AS final
WORKDIR /app
COPY --from=builder /app/build/libs/*.jar app.jar
USER 1001
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Maven:**

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /app
COPY pom.xml ./
RUN mvn dependency:go-offline -B          # cache dependency layer separately
COPY src ./src
RUN mvn verify -B

FROM scratch AS test-export
COPY --from=builder /app/target/surefire-reports ./

FROM eclipse-temurin:21-jre-alpine AS final
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
USER 1001
ENTRYPOINT ["java", "-jar", "app.jar"]
```

> **Platform engineering note:** Provide these as reference Dockerfiles in the platform documentation. The `test-export` stage MUST use `FROM scratch` — any other base image adds unnecessary overhead to the output extraction step.

### 7.3 Why Two Invocations Are Near-Instant on the Second Run

BuildKit uses content-addressed layer caching. After the first invocation (`--target test-export`), every layer up to and including the `builder` stage is cached locally and in ACR. The second invocation (`--target final`) finds all `builder` stage layers as cache hits and only assembles the final stage. The second invocation typically completes in seconds.

---

## 8. npm Dependency Caching via Azure Artifacts

Angular and React builds route all npm traffic through the platform Azure Artifacts npm feed. This eliminates build time spikes caused by public npm registry network congestion during high-demand periods.

### 8.1 Architecture

```
docker build (ephemeral agent)
        │
        │  RUN --mount=type=secret,id=npm_token  npm ci
        ▼
Azure Artifacts npm feed                        ← persistent cross-build cache
        │  cache miss only
        ▼
registry.npmjs.org (public npm)
```

After the first team fetches a package version, it is served from Azure Artifacts on every subsequent build regardless of which agent or pipeline run requested it. External network dependency is limited to first-fetch per package version.

### 8.2 Injection Mechanism

The pipeline injects two values into the `docker build` invocation for Angular and React builds:

```bash
docker build \
  --build-arg NPM_REGISTRY=$(NPM_REGISTRY_URL) \
  --secret id=npm_token,env=AZURE_ARTIFACTS_TOKEN \
  ...
```

- `NPM_REGISTRY_URL` — the Azure Artifacts feed URL from `platform-tool-versions`. Not sensitive; safe as `--build-arg`.
- `AZURE_ARTIFACTS_TOKEN` — set to `$(System.AccessToken)` in the pipeline. The ADO pipeline token has implicit read access to Azure Artifacts feeds in the same organisation. Passed as a BuildKit secret — never appears in image layers or `docker history` output.

### 8.3 Tenant Dockerfile Pattern

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app

COPY package.json package-lock.json ./
ARG NPM_REGISTRY
RUN --mount=type=secret,id=npm_token \
    npm config set registry ${NPM_REGISTRY} && \
    npm config set //${NPM_REGISTRY#https://}:_authToken=$(cat /run/secrets/npm_token) && \
    npm ci

COPY . .
RUN npm run build

FROM nginx:alpine AS final
COPY --from=builder /app/dist /usr/share/nginx/html
```

**Critical structural requirement:** `COPY package.json package-lock.json ./` MUST appear before `COPY . .`. This allows the ACR layer cache to serve the `npm ci` layer as a cache hit when only source files change (not dependencies). See `docs/NPM-CACHING-PATTERN.md` for full detail.

### 8.4 Why --build-arg Cannot Be Used for the Token

`--build-arg` values are recorded in the image layer history and visible via `docker history`. Passing an auth token as a build arg would expose it to anyone with pull access to the image. Hadolint enforces this at ERROR level — any Dockerfile that uses `ARG` in a pattern consistent with secret injection fails the lint step. BuildKit `--secret` is the only compliant mechanism.

---

## 9. Image Naming, Tagging, and Versioning

### 9.1 Repository Naming

```
<acr-hostname>/<tenantName>/<appName>

Example: platform.azurecr.io/payments/payment-processor
```

Enforced by the pipeline. Tenant teams do not choose their ACR repository path.

### 9.2 Tag Convention

```
┌────────────────────────────────────────────────────────────────────────┐
│  Per build, always:                                                    │
│                                                                        │
│  <40-char-git-sha>          ← primary, immutable, used in manifests   │
│  <branch>-<short-sha>       ← alias, mutable, human navigation only   │
│                                                                        │
│  Main branch only:                                                     │
│                                                                        │
│  <version-from-project-file>  ← e.g. 1.2.3                           │
│  Pipeline FAILS if this tag already exists in ACR                     │
│  → forces version bump before merging to main                         │
│                                                                        │
│  Feature branches only:                                                │
│                                                                        │
│  <version>-<short-sha>      ← e.g. 1.2.3-abc1234                     │
│  Always unique, safe to push repeatedly                                │
│                                                                        │
│  Prohibited (pipeline assertion):                                      │
│                                                                        │
│  latest                     ← asserted before push, fail if present   │
└────────────────────────────────────────────────────────────────────────┘
```

### 9.3 Version Detection by Runtime

| Runtime | Project file | Detection method |
|---|---|---|
| Spring Boot (Maven) | `pom.xml` | XPath: `//project/version/text()` |
| Spring Boot (Gradle) | `build.gradle` | Regex: `^\s*version\s*=\s*['"](.+)['"]` |
| Spring Boot (Gradle) | `gradle.properties` | Key: `version=` |
| Angular / React | `package.json` | JSON key: `.version` |
| Python | `pyproject.toml` | TOML key: `[project].version` or `[tool.poetry].version` |
| Python | `setup.cfg` | INI key: `version` under `[metadata]` |
| Go | `VERSION` (repo root) | File contents, trimmed |
| Go | *(no VERSION file)* | Version tag skipped; SHA + alias tags only |

Version parsing MUST fail Stage 1 if the project file is present but the version field is missing or unparseable. Version tag format is validated against semver (`\d+\.\d+\.\d+`) — non-semver versions produce a Stage 1 warning but do not block.

### 9.4 Digest Reference (Gold Standard)

The pipeline output variable carries the full digest reference:

```
<acr-host>/<tenantName>/<appName>@sha256:<digest>

Example: platform.azurecr.io/payments/payment-processor@sha256:abc123...
```

This is the canonical handoff to manual Kustomize config repo updates and the future Kargo integration. Tags are never used in manifests.

---

## 10. Security and Provenance Model

### 10.1 Pipeline Security Controls

```
┌─────────────────────────────────────────────────────────────┐
│                    Security Control Stack                   │
│                                                             │
│  Hadolint (Stage 2)                                         │
│  ├─ ERROR: blocks build                                     │
│  │   ├─ Secret injection via ARG/ENV                       │
│  │   ├─ Running as root in final stage                     │
│  │   └─ Use of ADD instead of COPY                         │
│  └─ WARNING: reported only                                  │
│                                                             │
│  Cosign (Stage 3)                                           │
│  ├─ Signs image digest (not tag)                           │
│  ├─ Key retrieved from AKV at sign time only               │
│  ├─ Signature stored as <sha>.sig in ACR                   │
│  └─ Verification runs before Stage 4 — fails block push    │
│                                                             │
│  Syft SBOM (Stage 3)                                        │
│  ├─ Scans locally held image (captures base image deps)    │
│  ├─ CycloneDX JSON format                                  │
│  ├─ Stored as pipeline artifact (audit retention)          │
│  └─ Attached as OCI attestation <sha>.att in ACR           │
│                                                             │
│  ACR Push (Stage 4)                                         │
│  ├─ Per-tenant service connection (scoped to tenant/*)     │
│  ├─ No registry-wide push credentials                      │
│  ├─ Digest verified post-push (ACR ↔ build digest match)  │
│  └─ latest tag assertion before push                       │
│                                                             │
│  Secret Hygiene                                             │
│  ├─ No --build-arg for secrets                             │
│  ├─ npm token via BuildKit --secret only                   │
│  └─ Cosign key via AKV task, not persisted on agent        │
└─────────────────────────────────────────────────────────────┘
```

### 10.2 Cosign Signing Flow

```
Stage 3: Sign & Attest
        │
        ├─ AzureKeyVault@2 task
        │    └─ retrieves cosign-private-key → $COSIGN_PRIVATE_KEY
        │
        ├─ cosign sign \
        │    --key $COSIGN_PRIVATE_KEY \
        │    $(ACR_HOST)/$(tenantName)/$(appName)@$(IMAGE_DIGEST)
        │    └─ writes signature to ACR: <sha>.sig
        │
        ├─ syft $(ACR_HOST)/$(tenantName)/$(appName)@$(IMAGE_DIGEST) \
        │    -o cyclonedx-json \
        │    --file sbom.json
        │    └─ publishes sbom.json as pipeline artifact
        │
        ├─ cosign attest \
        │    --key $COSIGN_PRIVATE_KEY \
        │    --predicate sbom.json \
        │    --type cyclonedx \
        │    $(ACR_HOST)/$(tenantName)/$(appName)@$(IMAGE_DIGEST)
        │    └─ writes attestation to ACR: <sha>.att
        │
        └─ cosign verify \
             --key $COSIGN_PUBLIC_KEY \
             $(ACR_HOST)/$(tenantName)/$(appName)@$(IMAGE_DIGEST)
             ├─ pass → proceed to Stage 4
             └─ fail → BLOCK Stage 4; image NOT pushed
```

### 10.3 Digest Carry-Through

The image digest is captured at build time and carried through every subsequent operation. No step references the image by tag after Stage 2.

```
Stage 2: docker build --iidfile image-digest.txt
         IMAGE_DIGEST=$(cat image-digest.txt)   # sha256:abc123...

Stage 3: cosign sign   ... @$(IMAGE_DIGEST)
         cosign attest ... @$(IMAGE_DIGEST)
         cosign verify ... @$(IMAGE_DIGEST)

Stage 4: docker push   ... @$(IMAGE_DIGEST)
         ACR verify    → returned digest must match $(IMAGE_DIGEST)

Stage 5: output variable = $(ACR_HOST)/$(tenantName)/$(appName)@$(IMAGE_DIGEST)
```

### 10.4 ACR Provenance Traceability

Given a running container's image reference, platform engineering can reconstruct:

| Data point | How to retrieve |
|---|---|
| Source commit | Git SHA in primary image tag or `org.opencontainers.image.revision` label |
| Pipeline run | ADO build tag on the pipeline run (set to image digest in Stage 5) |
| SBOM | `cosign download attestation <image>@<digest>` from ACR |
| Cosign signature | `cosign verify --key <public-key> <image>@<digest>` |
| Build log | ADO pipeline run ID correlated via build tag |

---

## 11. Tooling Reference

### 11.1 Docker BuildKit

**Version:** Resolved from `platform-tool-versions` variable group.
**Purpose:** Container image build engine.

Key BuildKit capabilities used by this pipeline:

| Capability | Flag | Purpose |
|---|---|---|
| Registry cache | `--cache-from`, `--cache-to` | Persist layer cache in ACR between builds |
| Secret injection | `--secret id=<name>,env=<var>` | Pass npm token without exposing in layers |
| Targeted output | `--target <stage> --output type=local,dest=<path>` | Extract test results from Spring Boot builds |
| Digest capture | `--iidfile <file>` | Capture image digest for downstream steps |
| Rootless mode | `BUILDKIT_HOST` or `rootlesskit` | Avoid running builds as root on the agent |

### 11.2 Hadolint

**Version:** Resolved from `platform-tool-versions`.
**Purpose:** Dockerfile static analysis.

| Finding level | Pipeline behaviour |
|---|---|
| ERROR | Build fails immediately |
| WARNING | Reported in run summary; build continues |
| Info / Style | Ignored |

Key rules enforced at ERROR level relevant to this pipeline:

| Rule | Description |
|---|---|
| DL3002 | Last USER is not root — final stage must not run as root |
| DL3020 | Use COPY instead of ADD for local files |
| SC2028 | Shell quoting issues in RUN commands |
| DL4006 | Set SHELL option `-o pipefail` with pipe chains |

Hadolint configuration (`.hadolint.yaml` in the platform templates repository):

```yaml
failure-threshold: error
trustedRegistries: []          # base image enforcement deferred to v2
ignore:
  - DL3008                     # apt-get version pinning (managed by base images)
```

### 11.3 Syft

**Version:** Resolved from `platform-tool-versions`.
**Purpose:** SBOM generation from built container images.

```bash
syft <image>@<digest> \
  -o cyclonedx-json \
  --file sbom.json
```

Syft scans the locally held image — not source code — to capture all runtime dependencies including those introduced by the base image. The CycloneDX JSON output is:

1. Published as a named ADO pipeline artifact (`sbom-$(Build.BuildId).json`) for audit retention
2. Attached to the image as an OCI attestation via `cosign attest`

### 11.4 Cosign

**Version:** Resolved from `platform-tool-versions`.
**Purpose:** Image signing and SBOM attestation.

```bash
# Sign the image digest
cosign sign \
  --key $COSIGN_PRIVATE_KEY \
  $(ACR_HOST)/$(tenantName)/$(appName)@$(IMAGE_DIGEST)

# Attach SBOM as OCI attestation
cosign attest \
  --key $COSIGN_PRIVATE_KEY \
  --predicate sbom.json \
  --type cyclonedx \
  $(ACR_HOST)/$(tenantName)/$(appName)@$(IMAGE_DIGEST)

# Verify signature (must pass before Stage 4)
cosign verify \
  --key cosign.pub \
  $(ACR_HOST)/$(tenantName)/$(appName)@$(IMAGE_DIGEST)
```

The public key (`cosign.pub`) is stored in the platform templates repository and distributed as a platform-managed Kubernetes resource for Kyverno admission verification.

### 11.5 Platform Tool Versions Variable Group

The `platform-tool-versions` ADO variable group is the single source of truth for all pinned tool versions. Platform engineering manages this group.

| Variable | Example value | Purpose |
|---|---|---|
| `BUILDKIT_VERSION` | `0.13.2` | Docker BuildKit version |
| `SYFT_VERSION` | `1.4.1` | Syft SBOM generator version |
| `COSIGN_VERSION` | `2.2.4` | Cosign signing tool version |
| `HADOLINT_VERSION` | `2.12.0` | Dockerfile linter version |
| `NPM_REGISTRY_URL` | `https://pkgs.dev.azure.com/<org>/_packaging/<feed>/npm/registry/` | Azure Artifacts npm feed |

---

## 12. ADO Configuration Reference

### 12.1 Service Connections

One ADO service connection is required per tenant. The connection is scoped to the tenant's ACR namespace only — it MUST NOT have registry-wide push access.

| Property | Value |
|---|---|
| Type | Docker Registry |
| Registry | `<acr-hostname>` |
| Authentication | Service principal or managed identity |
| Scope | `<tenantName>/*` push only |
| Name convention | `acr-<tenantName>` |

### 12.2 Variable Groups

**`platform-tool-versions`** (organisation-wide):

```yaml
# Not secret — all values are safe to log
BUILDKIT_VERSION: 0.13.2
SYFT_VERSION: 1.4.1
COSIGN_VERSION: 2.2.4
HADOLINT_VERSION: 2.12.0
NPM_REGISTRY_URL: https://pkgs.dev.azure.com/<org>/_packaging/<feed>/npm/registry/
GRADLE_CACHE_ENDPOINT: https://<gradle-cache-host>/cache/
```

**`tenant-<tenantName>-notifications`** (per-tenant, provisioned at onboarding):

```yaml
# Secret variable — not logged, not passed as build arg
TEAMS_WEBHOOK_URL: https://...  # marked as secret in ADO
```

### 12.3 Key Vault Integration

The Cosign private key is retrieved via the ADO Key Vault task at the start of Stage 3:

```yaml
- task: AzureKeyVault@2
  inputs:
    azureSubscription: 'platform-keyvault-connection'
    KeyVaultName: 'platform-signing-kv'
    SecretsFilter: 'cosign-private-key,cosign-private-key-password'
    RunAsPreJob: false
  displayName: Retrieve Cosign signing key
```

The retrieved secret is available as a pipeline variable for the duration of Stage 3 only. It is not passed between stages or persisted on the agent.

### 12.4 Pipeline Template Reference (Tenant Side)

```yaml
# azure-pipelines.yml in tenant source repository
trigger:
  branches:
    include:
      - main

variables:
  - group: tenant-payments-notifications    # per-tenant notification config
  - name: NOTIFY_ON_SUCCESS
    value: 'true'
  - name: NOTIFICATION_EMAIL
    value: 'payments-team@company.com'

extends:
  template: templates/container-build-v2.yml@platform-templates
  parameters:
    tenantName: payments
    appName: payment-processor
    runtimeType: springboot
    dockerfilePath: ./Dockerfile
    buildContext: .
    dryRun: false
```

---

## 13. Notification Architecture

### 13.1 PR Comment

Posted by the pipeline service principal to the ADO PR via the ADO REST API at the end of Stage 5. Contains:

```
✅ Build succeeded | payments/payment-processor

Image:     platform.azurecr.io/payments/payment-processor@sha256:abc123...
Tags:      abc123...def456 (SHA) | 1.2.3 (version) | main-abc1234 (alias)
SBOM:      sbom-1234.json (pipeline artifact)
Signed:    ✅ Cosign signature verified
Runtime:   springboot | Tests: 142 passed, 0 failed

⚠️  Security scanning (Trivy/Nexus/Fortify) runs in a separate pipeline.
    Results are advisory-only. Link: <scan-pipeline-run-url>

🔄 dryRun: false — image published to ACR
```

### 13.2 Teams Webhook

Sent from Stage 5 using the webhook URL from the per-tenant notifications variable group. Includes build status, image reference, and a link to the pipeline run.

```
Architecture:
  per-tenant variable group
        │
        │  TEAMS_WEBHOOK_URL (secret variable)
        ▼
  Stage 5: Notify
        │
        │  POST $(TEAMS_WEBHOOK_URL)
        │  { "text": "Build <status> — payments/payment-processor:1.2.3" }
        ▼
  Teams channel
```

Non-sensitive notification preferences (`NOTIFY_ON_SUCCESS`, `NOTIFICATION_EMAIL`) are sourced from pipeline YAML variables in the tenant's `azure-pipelines.yml`, keeping them version-controlled and tenant-managed.

---

## 14. Security Scan Handoff

### 14.1 Trigger Mechanism

The downstream security scan pipeline is triggered via ADO pipeline completion trigger. The security scan pipeline team owns the trigger definition — no configuration is required in the build pipeline template.

The security scan pipeline declares a resource dependency on the build pipeline:

```yaml
# security-scan-pipeline (owned by security team)
resources:
  pipelines:
    - pipeline: container-build-v2
      source: container-build-v2
      trigger:
        branches:
          include: ['*']
```

### 14.2 Handoff Contract

The build pipeline makes the following outputs available to the security scan pipeline:

| Output | Type | Value |
|---|---|---|
| `IMAGE_DIGEST_REF` | Pipeline output variable | `<acr>/<tenant>/<app>@sha256:<digest>` |
| Cosign signature | ACR artifact | `<acr>/<tenant>/<app>:<sha>.sig` |
| SBOM attestation | ACR artifact | `<acr>/<tenant>/<app>:<sha>.att` |

The security scan pipeline consumes `IMAGE_DIGEST_REF` to target the exact built image. Trivy, Nexus, and Fortify findings are the exclusive responsibility of the security scan pipeline. The build pipeline does not gate on scan results.

### 14.3 Advisory-Only Status

Security scan results are advisory-only in v1. There is no deployment hold enforced by this pipeline or by the current platform. Kyverno admission enforcement (future, PRD-2026-KYVERNO-POLICY-001) will be the cluster-side gate once ImageVerification policies are in place. Until then, consultation between dev and security teams governs scan finding remediation.

---

## 15. ACR Repository Structure

For a tenant `payments` with app `payment-processor` after a successful build:

```
platform.azurecr.io/
  payments/
    payment-processor/
      :<40-char-sha>              ← primary image (immutable)
      :1.2.3                      ← version tag (main branch)
      :main-abc1234               ← alias tag (human navigation)
      :<sha>.sig                  ← Cosign signature
      :<sha>.att                  ← SBOM attestation (CycloneDX)
      :buildcache                 ← BuildKit layer cache (managed by pipeline)
```

The `buildcache` tag is managed entirely by the pipeline. It is not an image and should not be pulled directly. ACR retention policies for the cache tag should be configured to retain the most recent entry only (platform operations responsibility — out of scope for this pipeline).

---

## 16. Tenant Onboarding Checklist

Platform engineering performs the following steps when onboarding a new tenant to the build pipeline.

**Prerequisites:**

- [ ] Tenant name confirmed (lowercase alphanumeric + hyphens only)
- [ ] App name confirmed
- [ ] runtimeType confirmed (`angular`, `react`, `springboot`, `python`, `go`)
- [ ] Tenant Dockerfile reviewed for Pattern A compliance (multi-stage, non-root final stage)
- [ ] Spring Boot tenants: `test-export` stage present in Dockerfile
- [ ] Angular/React tenants: Azure Artifacts npm pattern implemented in Dockerfile

**ADO provisioning:**

- [ ] ADO service connection created: `acr-<tenantName>` scoped to `<tenantName>/*` push
- [ ] Notifications variable group created: `tenant-<tenantName>-notifications` with `TEAMS_WEBHOOK_URL`
- [ ] Tenant `azure-pipelines.yml` reviewed and `extends:` reference confirmed

**Validation:**

- [ ] First run with `dryRun: true` — confirms build succeeds without ACR push
- [ ] Review pipeline run: Stage 1 validation passes, digest captured, lint clean
- [ ] Spring Boot tenants: confirm test results appear in ADO Tests tab
- [ ] Angular/React tenants: confirm npm resolves from Azure Artifacts feed (no public registry hits in build log)

**Go-live:**

- [ ] Set `dryRun: false` in tenant `azure-pipelines.yml`
- [ ] First full run: confirm image, `.sig`, and `.att` present in ACR
- [ ] Confirm security scan pipeline triggered within SM-7 SLA (15 minutes)
- [ ] Confirm Teams notification received
- [ ] Confirm PR comment posted with correct image reference

---

## 17. Operational Considerations

### 17.1 ACR Cache Management

The `buildcache` tag per tenant is overwritten on every build (mode=max). ACR retention policies for this tag are the responsibility of platform operations and are outside this pipeline's scope. Recommended policy: no retention limit on the `buildcache` tag (always keep the latest).

Stale `buildcache` entries from decommissioned tenants should be cleaned up manually via `az acr repository delete`.

### 17.2 Cosign Key Rotation

1. Generate new Cosign key pair: `cosign generate-key-pair --kms azurekv://...`
2. Update the secret in Azure Key Vault (`cosign-private-key`)
3. Distribute the new public key as a platform-managed Kubernetes resource
4. Bump the platform templates repository version tag
5. Existing image signatures remain valid under the old public key — do not remove the old public key until all pre-rotation images are decommissioned
6. Notify tenant teams of the key rotation via the platform changelog

### 17.3 Tool Version Updates

Tool versions are updated in the `platform-tool-versions` variable group. All pipeline runs pick up the new version on their next execution — no template change is required. Breaking changes in tool behaviour require a platform templates version bump and testing against reference builds before updating the variable group.

### 17.4 Azure Artifacts Feed Management

The npm feed is a platform resource. Package retention and upstream proxy configuration are platform operations responsibilities. Monitor feed storage utilisation — Azure Artifacts applies storage quotas per organisation.

### 17.5 Adding a New Tenant Runtime

See Section 5 (Template Dispatch Model) for the full process. Any new runtime requires:

1. A new step template file
2. A dispatch entry in the base template
3. A PRD revision
4. A platform templates version tag

---

## 18. Future State

The following capabilities are planned but out of scope for v1. They are recorded here to inform architectural decisions and avoid designs that would require significant rework.

| Capability | Trigger | Notes |
|---|---|---|
| Keyless Cosign signing | Azure Workload Identity onboarding | Replaces AKV key management with OIDC-bound signatures |
| Kargo image promotion | Akuity platform adoption | Consumes pipeline output variable; replaces manual Kustomize updates |
| ARM multi-arch builds | ARM node pool adoption | Requires QEMU or native ARM agents; linux/amd64 only for v1 |
| Hadolint base image enforcement | Approved base image list availability | `trustedRegistries` config; deferred to security scan + Kyverno |
| Self-service tenant onboarding | Tenant volume growth | Automates ADO service connection and variable group provisioning |
| Maven/Gradle dependency caching | Azure Artifacts Maven/Gradle feed | Parallel to npm pattern; same BuildKit secret mechanism |
| Existing tenant migration tooling | Post-POC | Migration guide + tooling for bespoke pipeline tenants |
