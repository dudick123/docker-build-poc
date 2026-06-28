# PRD: CI Container Build Pipeline — v2 Architecture

**Document ID:** PRD-2026-CI-BUILD-001
**Document Status:** Draft
**Version:** 0.4.0
**Owner:** Platform Engineering
**Last Updated:** 2026-06-28

---

## Table of Contents

1. [Overview](#1-overview)
2. [Problem Statement](#2-problem-statement)
3. [Goals and Non-Goals](#3-goals-and-non-goals)
4. [Users and Stakeholders](#4-users-and-stakeholders)
5. [Success Metrics](#5-success-metrics)
6. [Architecture](#6-architecture)
7. [Pipeline Stages](#7-pipeline-stages)
8. [Functional Requirements](#8-functional-requirements)
9. [Non-Functional Requirements](#9-non-functional-requirements)
10. [Constraints and Assumptions](#10-constraints-and-assumptions)
11. [Dependencies](#11-dependencies)
12. [Open Questions](#12-open-questions)
13. [Revision History](#13-revision-history)
14. [Appendix A: Image Tagging Strategy Reference](#14-appendix-a-image-tagging-strategy-reference)
15. [Appendix B: Runtime Support Reference](#15-appendix-b-runtime-support-reference)
16. [Appendix C: Deferred and Future Decisions](#16-appendix-c-deferred-and-future-decisions)

---

## 1. Overview

This document defines requirements for a **v2 CI container build pipeline** for the AKS-based multi-tenant platform. The v2 architecture establishes a single, opinionated, best-practice standard for all container image builds across the platform, consumed by tenant application teams via a shared ADO pipeline template.

The pipeline triggers on PR merge to main in ADO source repositories. It builds container images from Dockerfiles, generates SBOMs using Syft, signs images and attestations using Cosign, and publishes to a shared Azure Container Registry. The pipeline establishes image provenance as a first-class output alongside the image itself.

All five supported runtimes — Angular, React, Spring Boot, Python, and Go — use self-contained multi-stage Dockerfiles (Pattern A). The pipeline agent requires Docker and BuildKit only; no language toolchains are managed on the agent image. Tenant teams own all build toolchain versions through their Dockerfile `FROM` statements.

Vulnerability and security scanning (Trivy, Nexus, Fortify) is performed by a dedicated security scan pipeline that operates as a separate loop against images already published to ACR. That pipeline is out of scope here. This pipeline hands off a signed, published image to that downstream security loop via the shared ACR. Security scan results are advisory-only until Kyverno admission enforcement is in place.

This PRD is a companion to PRD-2026-CI-RENDER-001 (manifest rendering pipeline). Together they define the CI pipelines that feed the platform GitOps delivery chain: this pipeline produces the image, the rendering pipeline consumes the image reference and produces the manifests that ArgoCD applies.

> **Scope boundary:** This PRD covers container image build, sign, and publish only. Security scanning (Trivy, Nexus, Fortify) is governed by the security scan pipeline PRD. Manifest generation, ArgoCD Application management, tenant onboarding, and runtime Kubernetes policy enforcement are governed by their respective PRDs.

---

## 2. Problem Statement

Tenant teams currently build container images using individually crafted ADO pipelines with no shared template, no enforced standards, and inconsistent security controls. Specific gaps:

**No platform build standard.** Each tenant team owns a bespoke pipeline. Build tool versions, layer caching strategies, base image sources, and output tag conventions differ across teams. There is no canonical reference for how a compliant build pipeline should be structured on this platform.

**No runtime composability model.** There is no standardised approach to supporting multiple languages and runtimes under a single pipeline template. Without a composable template model, adding a new runtime requires a new bespoke pipeline rather than a runtime-specific extension to a shared base.

**No image signing or provenance.** No images produced on the platform are currently signed. There is no attestation of build provenance, no SBOM associated with any image, and no mechanism for ArgoCD or Kyverno to enforce that only platform-built, signed images are admitted to the cluster.

**No SBOM generation.** Software Bill of Materials generation is absent. As regulatory and customer requirements around software supply chain transparency increase, the absence of SBOMs is a compliance gap.

**Tag inconsistency.** Image tags are team-defined and range from `latest` (explicitly prohibited but not enforced) to ad-hoc strings with no relationship to Git commit SHA or project version. There is no reliable way to trace a running image back to its source commit.

**ACR governance gap.** Images are pushed directly to the shared ACR without a consistent naming convention, retention policy enforcement, or a well-defined handoff point to the downstream security scan pipeline.

**npm build instability.** Angular and React builds that fetch from the public npm registry experience significant build time spikes during high-demand periods (end of sprint, many teams building simultaneously). No platform-managed dependency cache exists.

> **Note on security scanning:** Vulnerability and security scanning runs in a separate pipeline loop against images already in ACR. This pipeline's responsibility is to ensure a well-formed, signed, SBOM-attested image reaches ACR as a clean handoff to that downstream process.

---

## 3. Goals and Non-Goals

### Goals

- **G-1:** Establish a canonical v2 container build pipeline template consumed by all tenant teams via a shared ADO YAML template.
- **G-2:** Build container images from Dockerfiles on PR merge to main in ADO source repositories.
- **G-3:** Support the five primary platform runtimes (Angular, React, Spring Boot, Python, Go) through a composable base-plus-runtime-step-template architecture using self-contained multi-stage Dockerfiles for all runtimes.
- **G-4:** Generate a Software Bill of Materials (SBOM) using Syft for every image built by the pipeline.
- **G-5:** Sign every published image and its SBOM attestation using Cosign, establishing cryptographic build provenance as a platform standard.
- **G-6:** Enforce a consistent image tagging convention that includes Git commit SHA and project-file-derived version, enabling reliable traceability from running container to source code and release.
- **G-7:** Publish images to the shared ACR using a consistent repository naming convention scoped by tenant, providing a clean handoff point to the downstream security scan pipeline.
- **G-8:** Surface build provenance metadata to the ADO pull request and run summary for reviewer visibility.
- **G-9:** Provide a dryRun mode that executes all build steps without pushing to ACR or signing.
- **G-10:** Reduce npm dependency fetch instability for Angular and React builds by routing package resolution through a platform-managed Azure Artifacts npm feed.

### Non-Goals

- **NG-1:** This pipeline does not build base images. Base image curation and hardening is a platform engineering responsibility governed separately.
- **NG-2:** This pipeline does not update Kustomize overlays or config repo image references. Image reference promotion is a separate step, either manual or via a future Kargo integration.
- **NG-3:** This pipeline does not perform runtime admission enforcement. Kyverno policies governing which images are admitted to the cluster are governed by PRD-2026-KYVERNO-POLICY-001.
- **NG-4:** This pipeline does not manage ACR access control or retention policies. ACR governance is a platform engineering operation.
- **NG-5:** This pipeline does not support Cloud Native Buildpacks. CNB evaluation is deferred.
- **NG-6:** This pipeline does not perform vulnerability scanning, SCA, SAST, or DAST. Trivy, Nexus, and Fortify scanning runs in a dedicated security scan pipeline. Scan gate enforcement and remediation workflows are governed by the security scan pipeline PRD.
- **NG-7:** This pipeline does not manage language toolchain versions on the agent image. All build toolchain versions are owned by tenant teams through their Dockerfile `FROM` statements.

---

## 4. Users and Stakeholders

| Role | Interest |
|---|---|
| **Tenant Teams** | Primary consumers of the pipeline template. Build and publish application container images. Receive build provenance and signing confirmation as pipeline feedback. |
| **Platform Engineering** | Authors and maintainers of the shared pipeline template, Cosign key infrastructure, and ACR naming conventions. |
| **Platform Lead** | Accountable for v2 build pipeline adoption, supply chain security posture, and SBOM compliance requirements. |
| **Security / Compliance** | Interested in SBOM generation, Cosign signing, and image provenance as inputs to the downstream security scan pipeline. |
| **Security Scan Pipeline** | Downstream consumer of images published to ACR by this pipeline. Runs Trivy, Nexus, and Fortify scans against the published image digest. |
| **Akuity / ArgoCD + Kyverno** | Downstream consumers of signed images. Kyverno admission policies will eventually enforce that only Cosign-verified, platform-built images are admitted. |

---

## 5. Success Metrics

| ID | Metric | Target |
|---|---|---|
| SM-1 | All tenant container builds use the v2 shared pipeline template. | 100% of tenant image builds migrated from bespoke pipelines. |
| SM-2 | Every image pushed to ACR has a corresponding Cosign signature and SBOM attestation. | 100% of images published post-rollout. |
| SM-3 | Every image tag includes the Git commit SHA. | 100% of published images traceable to source commit. |
| SM-4 | The `latest` tag is never pushed to ACR by the pipeline. | Zero `latest` tag pushes from pipeline-managed builds. |
| SM-5 | Build provenance summary is surfaced in the ADO PR or run summary for every build. | 100% of builds produce a provenance summary visible to the PR author. |
| SM-6 | SBOM artifacts are published to ACR as OCI attestations alongside every image. | 100% of published images have an attached SBOM. |
| SM-7 | Every published image is consumed by the downstream security scan pipeline within the defined SLA. | Security scan pipeline triggered within 15 minutes of image push to ACR. |
| SM-8 | dryRun mode validated by at least three tenant teams before mandatory rollout. | Confirmed before rollout gate. |
| SM-9 | Mean time from PR merge to image available in ACR. | < 15 minutes for standard Dockerfile builds with warm layer cache. |

---

## 6. Architecture

### 6.1 Pattern

The pipeline follows a **build → sign → publish** pattern. No image is published without a Cosign signature and SBOM attestation. Once published to ACR, the image digest is handed off to the downstream security scan pipeline (Trivy, Nexus, Fortify) which operates as a separate loop. The pipeline is expressed as a shared ADO YAML template referenced by tenant repositories.

```
ADO Source Repo (tenant)             Azure Container Registry (shared)
+-----------------------------+       +--------------------------------+
| Source Code                 |  CI   | <tenant>/<app>:<sha>           |
| Dockerfile                  | ────► | <tenant>/<app>:<sha>.sig       |
| PR merge to main            | build |   (Cosign signature)           |
|                             | sign  | <tenant>/<app>:<sha>.att       |
|                             | push  |   (SBOM attestation)           |
+-----------------------------+       | <tenant>/<app>:<version>       |
                                      |   (version tag, main branch)   |
                                      | <tenant>/<app>:<branch>-<sha>  |
                                      |   (human-readable alias tag)   |
                                      +----------------+---------------+
                                                       |
                                          ┌────────────┴────────────┐
                                          │                         │
                                          ▼                         ▼
                                 Security Scan Pipeline     Kustomize image ref
                                 (Trivy, Nexus, Fortify)    updated manually or
                                 — advisory only in v1       via future Kargo
                                 — separate pipeline
```

### 6.2 Shared Pipeline Template Model

The v2 build pipeline is implemented as a **composable ADO YAML template set** stored in the platform templates repository. The base template owns all invariant platform steps. Runtime-specific validation is dispatched to dedicated step templates.

```
platform-templates/
  container-build-v2.yml          ← tenant entry point (base template)
  steps/
    setup.yml                     ← tool resolution, parameter validation
    dockerfile-lint.yml           ← Hadolint
    docker-build.yml              ← BuildKit build + OCI label injection
    sbom-sign-publish.yml         ← Syft, Cosign, ACR push, notify
    runtime/
      angular.yml                 ← pass-through validator + npm caching injection
      react.yml                   ← pass-through validator + npm caching injection
      springboot.yml              ← two-invocation BuildKit + test result export
      python.yml                  ← pass-through validator
      go.yml                      ← pass-through validator (go.mod check)
```

Tenant pipeline entry point:

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
    runtimeType: springboot        # angular | react | springboot | python | go
    dockerfilePath: ./Dockerfile
    buildContext: .
```

All platform controls (Cosign keys, ACR endpoint, tag convention) are defined inside the base template and not overridable by tenant parameters.

### 6.3 Image Naming and Tagging Convention

Images are published to the shared ACR using a platform-defined naming convention. Tenant teams do not control ACR repository names or tag formats.

**Repository name:** `<tenant-name>/<app-name>`

**Tags published per build:**

| Tag | Format | Condition | Purpose |
|---|---|---|---|
| Primary (immutable) | `<git-sha>` (full 40-char SHA) | Always | Canonical reference for manifests and Kyverno verification. |
| Version (main branch) | `<version-from-project-file>` | Main branch only; fails if exists | Release version tag. Fails pipeline if already exists — forces version bump. |
| Version (feature branch) | `<version-from-project-file>-<short-sha>` | Non-main branches | Preview builds, always unique, safe to overwrite. |
| Alias (human-readable) | `<branch>-<short-sha>` | Always | Human navigation only; not used in manifests. |

**Version detection by runtime:**

| Runtime | Source file | Field / element |
|---|---|---|
| Spring Boot (Maven) | `pom.xml` | `<version>` element |
| Spring Boot (Gradle) | `build.gradle` or `gradle.properties` | `version` property |
| Angular / React | `package.json` | `version` field |
| Python | `pyproject.toml` or `setup.cfg` | `version` field |
| Go | `VERSION` file in repo root | File contents (trimmed) |

If a Go repository has no `VERSION` file, the version tag is skipped. All other runtimes must have a detectable version; failure to parse the version MUST fail Stage 1.

The `latest` tag MUST NOT be pushed by any pipeline-managed build. This is enforced as a pipeline-level assertion, not a convention.

### 6.4 Image Provenance Model

Every published image has three associated OCI artifacts in ACR:

- **Image** — the built container image, tagged per 6.3.
- **Cosign signature** — a detached signature of the image digest, stored as `<image>:<sha>.sig`. Produced by `cosign sign`.
- **SBOM attestation** — a CycloneDX SBOM generated by Syft, attached as an OCI attestation via `cosign attest`. Stored as `<image>:<sha>.att`.

Cosign signing uses a platform-managed key stored in Azure Key Vault. The pipeline retrieves the private key at sign time via the ADO Key Vault task. Tenant teams have no access to the signing key.

### 6.5 Security Scan Pipeline Handoff

Vulnerability and security scanning is not performed by this pipeline. It runs in a dedicated security scan pipeline triggered via ADO pipeline completion trigger after Stage 4 (Publish) succeeds. Security scan results are advisory-only for v1 — no deployment hold is enforced by this pipeline.

The handoff contract between this pipeline and the security scan pipeline is:

| Artifact | Location | Purpose |
|---|---|---|
| Image | `<acr>/<tenant>/<app>:<sha>` | The build artifact to be scanned. |
| Image digest | ADO pipeline output variable | Unambiguous reference for the security scan pipeline. |
| Cosign signature | `<acr>/<tenant>/<app>:<sha>.sig` | Allows the security scan pipeline to verify the image originated from the platform build pipeline. |
| SBOM attestation | `<acr>/<tenant>/<app>:<sha>.att` | CycloneDX SBOM available without re-generating from the image. |

### 6.6 Cosign Key Infrastructure

Cosign signing uses **key-based signing** (not keyless/OIDC). The private key is stored in Azure Key Vault under platform engineering control. Keyless signing via Azure Workload Identity is a deferred future decision (see Appendix C).

Verification at admission time (Kyverno ImageVerification policy) uses the corresponding public key, distributed as a platform-managed ConfigMap or Kyverno ClusterPolicy resource.

### 6.7 Runtime Build Pattern

All five supported runtimes use **Pattern A — In-Dockerfile build (self-contained)**. The entire build toolchain runs inside a multi-stage Dockerfile. The pipeline agent only needs Docker (BuildKit); no language runtime is required on the agent.

| Runtime | runtimeType | Agent requirement | Final image |
|---|---|---|---|
| Angular | `angular` | Docker + BuildKit | nginx + static assets |
| React | `react` | Docker + BuildKit | nginx + static assets (or Node for SSR) |
| Spring Boot | `springboot` | Docker + BuildKit | JRE + JAR |
| Python | `python` | Docker + BuildKit | python-slim or distroless |
| Go | `go` | Docker + BuildKit | distroless/static or scratch |

**Spring Boot special case — two-invocation BuildKit for test results:**

Spring Boot builds require test results to be surfaced to ADO's Tests tab. Since the build runs inside Docker, test results are extracted using BuildKit's output mechanism rather than running Gradle or Maven on the agent:

1. The Dockerfile MUST include a `FROM scratch AS test-export` stage that copies test result files to a standard path.
2. The pipeline runs `docker build --target test-export --output type=local,dest=./test-results .` first.
3. ADO `PublishTestResults` reads from `./test-results`. Test failure blocks the pipeline.
4. The pipeline then runs `docker build --target final .`. BuildKit layer cache means this second invocation is near-instant.

**Angular and React npm caching:**

Angular and React builds route npm traffic through the platform Azure Artifacts npm feed rather than the public registry. The feed URL is provided via `platform-tool-versions`. The auth token is injected via BuildKit `--secret` — it is never stored in build args or image layers. See `docs/NPM-CACHING-PATTERN.md` for full implementation detail.

---

## 7. Pipeline Stages

| Stage | Name | Condition | Key Jobs |
|---|---|---|---|
| 1 | Setup | Always | `ResolveTools`, `ValidateParameters`, `ValidateRuntimeType`, `DetectVersion` |
| 2 | Build | Stage 1 succeeded | `ValidateRuntime`, `DockerfileLint`, `BuildImage` |
| 3 | Sign & Attest | Stage 2 succeeded; `dryRun=false` | `GenerateSBOM`, `SignImage`, `AttachSBOM` |
| 4 | Publish | Stage 3 succeeded; `dryRun=false` | `PushImage`, `PushTags`, `PublishProvenance` |
| 5 | Notify | Always (post-pipeline) | `PublishBuildSummary`, `NotifyTeams`, `NotifySecurityScanPipeline` |

The build (Stage 2) produces a local image held on the pipeline agent. For all runtimes, the runtime step template (`ValidateRuntime`) performs validation only — no pre-build steps execute on the agent. The image is not pushed to ACR until Stage 4.

For Spring Boot, the two-invocation BuildKit sequence (test extraction then final image) runs within Stage 2 as part of `BuildImage`.

---

## 8. Functional Requirements

### 8.1 Tool Resolution

**FR-1.1:** The pipeline MUST resolve all tool versions from the platform-managed ADO variable group (`platform-tool-versions`). Hard-coded tool versions in pipeline template YAML are not permitted.

**FR-1.2:** The pipeline MUST cache resolved tool binaries as a named pipeline artifact consumed by all downstream jobs.

**FR-1.3:** The following tools MUST be pinned and resolved: Docker (BuildKit), Syft, Cosign, Hadolint.

**FR-1.4:** The pipeline template MUST be stored in the platform templates repository and referenced via `extends:` by tenant repositories. Tenant repositories MUST NOT copy or inline the template.

**FR-1.5:** The Azure Artifacts npm feed URL MUST be resolved from `platform-tool-versions` and made available to the `docker build` invocation as a build arg for Angular and React builds.

### 8.2 Parameters and Tenant Interface

**FR-2.1:** The pipeline template MUST expose the following tenant-supplied parameters and no others: `tenantName`, `appName`, `runtimeType`, `dockerfilePath`, `buildContext`, `dryRun`.

**FR-2.2:** All platform controls (Cosign key reference, ACR endpoint, tag convention) MUST be platform-defined within the base template and MUST NOT be overridable by tenant parameters.

**FR-2.3:** The pipeline MUST validate that `tenantName` and `appName` conform to the platform naming convention (lowercase alphanumeric and hyphens only, no slashes) before any build step executes. Violation MUST fail Stage 1.

**FR-2.4:** The pipeline MUST validate that the Dockerfile exists at the supplied `dockerfilePath` relative to the build context. A missing Dockerfile MUST fail Stage 1.

**FR-2.5:** The `runtimeType` parameter MUST be validated against the platform-defined allowlist: `angular`, `react`, `springboot`, `python`, `go`. An unsupported value MUST fail Stage 1 with a message identifying the invalid runtime and listing valid options.

**FR-2.6:** The pipeline MUST validate that the runtime step template corresponding to `runtimeType` exists in the platform templates repository at `steps/runtime/<runtimeType>.yml`. A missing step template MUST fail Stage 1.

### 8.3 Runtime Validation and Version Detection

**FR-3.1:** All runtime step templates MUST be validated pass-throughs. No pre-build steps execute on the pipeline agent for any runtime. The Dockerfile is responsible for the full build toolchain.

**FR-3.2:** Each runtime step template MUST perform the following validation in Stage 1:

- `go.yml`: assert `go.mod` exists; fail Stage 1 if absent. Read version from `VERSION` file if present.
- `python.yml`: advisory check for `requirements.txt`, `pyproject.toml`, or `poetry.lock`; warn only, do not block. Read version from `pyproject.toml` or `setup.cfg`.
- `springboot.yml`: detect Maven (`pom.xml`) or Gradle (`gradlew`); fail Stage 1 if neither found. Read version from the detected project file.
- `angular.yml`: advisory check that `package.json` exists. Read version from `package.json` `version` field.
- `react.yml`: advisory check that `package.json` exists. Read version from `package.json` `version` field. Detect Next.js SSR and emit a warning.

**FR-3.3:** For Spring Boot, the pipeline MUST validate that the Dockerfile contains a stage named `test-export` (i.e. `FROM scratch AS test-export`). A missing `test-export` stage MUST fail Stage 2 with a message directing the tenant to the platform reference Dockerfile.

**FR-3.4:** For Spring Boot, the pipeline MUST execute a two-invocation BuildKit build sequence in Stage 2:

1. `docker build --target test-export --output type=local,dest=./test-results .` — extracts test results to the agent.
2. ADO `PublishTestResults` task reads from `./test-results`. A test failure MUST fail Stage 2 and MUST NOT proceed to the final image build.
3. `docker build --target final .` — builds the final image. BuildKit layer caching ensures this second invocation reuses all shared layers from step 1.

**FR-3.5:** For Angular and React, the pipeline MUST inject the following into the `docker build` invocation to enable Azure Artifacts npm caching within the Dockerfile:

- `--build-arg NPM_REGISTRY=$(NPM_REGISTRY_URL)` — the registry URL from `platform-tool-versions`
- `--secret id=npm_token,env=AZURE_ARTIFACTS_TOKEN` — the ADO `System.AccessToken` passed as a BuildKit secret

The npm auth token MUST NOT be passed as a `--build-arg`. Tenant Dockerfiles that implement the Azure Artifacts pattern MUST use `--mount=type=secret,id=npm_token` to consume the token.

**FR-3.6:** A runtime step template MUST NOT modify the Dockerfile, the build context, or any file outside the scope of its validation output.

### 8.4 Build

**FR-4.1:** The pipeline MUST build the container image using Docker BuildKit. Legacy `docker build` without BuildKit MUST NOT be used.

**FR-4.2:** The pipeline MUST lint the Dockerfile using Hadolint before the build step. Hadolint findings at ERROR level MUST fail the build. WARNING level findings MUST be reported in the run summary but MUST NOT block.

**FR-4.3:** The pipeline MUST pass the full Git commit SHA as a build argument (`--build-arg GIT_COMMIT_SHA=<sha>`) and as an image label (`org.opencontainers.image.revision=<sha>`). The pipeline MUST also apply the following OCI labels to every built image: `org.opencontainers.image.source`, `org.opencontainers.image.created`, `org.opencontainers.image.revision`, `org.opencontainers.image.title`.

**FR-4.4:** The pipeline MUST NOT build images as root on the pipeline agent where avoidable. BuildKit rootless mode or a rootless agent configuration MUST be used.

**FR-4.5:** The pipeline MUST use layer caching backed by ACR cache storage (`--cache-from`, `--cache-to`) scoped per tenant repository (`<tenantName>/<appName>:buildcache`) to minimise redundant layer rebuilds across pipeline runs and eliminate cross-tenant cache access.

**FR-4.6:** The built image MUST be held locally on the pipeline agent after the build step. The image MUST NOT be pushed to ACR until Stage 4.

**FR-4.7:** The pipeline MUST record the full image digest (`sha256:...`) produced by the build step and carry it through all downstream stages. All downstream references (sign, push) MUST use the digest, not a tag.

### 8.5 Security Scan Pipeline Handoff

**FR-5.1:** Upon successful image push to ACR, the pipeline MUST publish the full image digest reference (`<acr-host>/<tenant>/<app>@sha256:<digest>`) as a named ADO pipeline output variable.

**FR-5.2:** The pipeline MUST trigger the downstream security scan pipeline via ADO pipeline completion trigger upon successful completion of Stage 4 (Publish). The security scan pipeline owns the trigger definition.

**FR-5.3:** The pipeline MUST NOT gate publish on security scan results. Security scan gate enforcement is the exclusive responsibility of the security scan pipeline.

**FR-5.4:** The build summary posted to the ADO PR MUST include a note that security scanning runs in a separate pipeline and that results are advisory-only. A direct link to the triggered security scan pipeline run MUST be included where supported.

### 8.6 SBOM Generation

**FR-6.1:** The pipeline MUST generate a Software Bill of Materials (SBOM) for every successfully built image using Syft.

**FR-6.2:** The SBOM MUST be generated in CycloneDX JSON format. SPDX format generation is optional but recommended as a secondary output.

**FR-6.3:** The SBOM MUST be generated from the locally held image (not from source code alone) to capture all runtime dependencies including those introduced by the base image.

**FR-6.4:** The SBOM file MUST be published as a named pipeline artifact retained for audit purposes, independent of its attachment as an OCI attestation.

### 8.7 Image Signing and Attestation

**FR-7.1:** The pipeline MUST sign every image using Cosign before pushing to ACR. Signing MUST use the platform-managed private key retrieved from Azure Key Vault at runtime via the ADO Key Vault task.

**FR-7.2:** Cosign signing MUST operate on the image digest (`sha256:...`), not a tag.

**FR-7.3:** The pipeline MUST attach the Syft-generated SBOM as an OCI attestation using `cosign attest`. The attestation MUST be stored in the same ACR repository as the image.

**FR-7.4:** The Cosign signing key MUST be retrieved from Azure Key Vault using the ADO Key Vault task at the start of Stage 3. The key MUST NOT be stored in pipeline variables, ADO variable groups, or any location accessible to tenant teams.

**FR-7.5:** The pipeline MUST verify the Cosign signature immediately after signing, before proceeding to Stage 4. A failed verification MUST block the push.

**FR-7.6:** The Cosign public key used for signature verification MUST be published as a platform-managed resource (Kyverno ClusterPolicy or ConfigMap) and versioned separately from the signing key. Key rotation is a platform engineering operation.

### 8.8 Publish

**FR-8.1:** The pipeline MUST push the image to the shared ACR only after all of the following have completed successfully: SBOM generated, Cosign signature produced and verified.

**FR-8.2:** The image MUST be pushed to ACR using the repository name `<tenantName>/<appName>` and tagged per the convention defined in section 6.3.

**FR-8.3:** The pipeline MUST push the following tags per build:

- Primary tag (full 40-char Git SHA) — always.
- Alias tag (`<branch>-<short-sha>`) — always.
- Version tag from project file:
  - On main branch: `<version-from-project-file>`. If this tag already exists in ACR, the pipeline MUST fail before pushing with a message instructing the tenant to bump the version in their project file.
  - On non-main branches: `<version-from-project-file>-<short-sha>`. Always unique; does not fail on existing tags.
  - Go with no `VERSION` file: version tag skipped.

**FR-8.4:** The pipeline MUST assert that the `latest` tag is not being pushed. If the tag convention logic produces a `latest` tag for any reason, the pipeline MUST fail before pushing.

**FR-8.5:** After push, the pipeline MUST verify that the image digest returned by ACR matches the digest produced at build time. A digest mismatch MUST fail the pipeline and trigger an alert.

**FR-8.6:** The pipeline MUST publish an image provenance summary as a pipeline artifact and as a PR comment. The summary MUST include: image digest, ACR repository path, tags pushed, SBOM artifact location, Cosign signature status, and a reference to the triggered security scan pipeline run.

**FR-8.7:** The pipeline MUST output the full image reference (`<acr-host>/<tenant>/<app>@sha256:<digest>`) as a named ADO pipeline output variable.

### 8.9 Dry Run Mode

**FR-9.1:** When `dryRun=true`, the pipeline MUST execute Stages 1 and 2 in full (tool resolution, runtime validation, Dockerfile lint, build) and MUST skip Stages 3 and 4 (sign and publish).

**FR-9.2:** In dryRun mode, the pipeline MUST publish a build summary to the run summary. The PR comment MUST indicate clearly that this was a dry run, no image was pushed to ACR, and therefore the security scan pipeline was not triggered.

**FR-9.3:** dryRun mode MUST NOT push any artifact to ACR, including the image, signature, and SBOM attestation.

### 8.10 Notify

**FR-10.1:** The pipeline MUST post a build summary comment to the ADO PR that triggered the build. The comment MUST include: build status, `runtimeType`, image digest, SBOM status, signing status, ACR image reference, and a note that Trivy/Nexus/Fortify security scanning runs in a separate pipeline (advisory-only) with a link to that pipeline where available.

**FR-10.2:** On pipeline failure, the failure stage and reason MUST be included in the PR comment.

**FR-10.3:** The pipeline MUST publish an ADO build tag on the pipeline run corresponding to the image digest, enabling correlation between pipeline runs and ACR images from the ADO UI.

**FR-10.4:** The pipeline MUST send a Teams channel notification on build completion (success and failure) using a webhook URL stored in the per-tenant variable group (`tenant-<tenantName>-notifications`). Non-sensitive notification configuration (email distribution list, notify-on-success flag) MAY be supplied as pipeline YAML variables in the tenant's `azure-pipelines.yml`.

---

## 9. Non-Functional Requirements

**NFR-1 — Repeatability:** Building from the same Git commit SHA MUST produce a functionally equivalent image. Layer caching and BuildKit determinism settings must be configured to minimise non-determinism from timestamp injection or package resolution order.

**NFR-2 — Performance:** Total pipeline duration from trigger to image available in ACR MUST be under 15 minutes for a standard single-stage Dockerfile build with a warm layer cache. Cold cache builds MUST complete within 25 minutes.

**NFR-3 — Template Immutability:** Tenant repositories reference the shared pipeline template by version tag or commit SHA. Tenant teams MUST NOT be able to modify security controls by changing the template reference to an unreviewed version.

**NFR-4 — Least Privilege:** The pipeline agent's ACR credentials MUST be scoped to push access for `<tenantName>/*` only. No pipeline agent credential MUST have registry-wide push or admin access. The Cosign private key MUST be accessible to the pipeline agent at sign time only via Key Vault task, not persisted on the agent.

**NFR-5 — Audit Trail:** Every published image MUST be traceable to: source ADO repository, commit SHA, pipeline run ID, SBOM artifact, and Cosign signature. The traceability chain MUST be reconstructible from ACR metadata and pipeline artifact retention alone.

**NFR-6 — No Direct ACR Push:** Tenant teams MUST NOT have direct ACR push credentials outside the pipeline. All images reaching ACR MUST pass through the shared build pipeline template and its security gates.

**NFR-7 — Secret Hygiene:** Build arguments (`--build-arg`) MUST NOT be used to inject secrets into the image at build time. The npm auth token for Azure Artifacts MUST be passed via BuildKit `--secret`, not `--build-arg`. Runtime secrets are delivered via ESO + Azure Key Vault. Hadolint enforces secret injection patterns at ERROR level.

**NFR-8 — Agent Independence:** The pipeline MUST NOT require any language toolchain (Node.js, JDK, Python, Go) to be pre-installed on the pipeline agent. All language toolchains run inside Docker. The agent requires Docker (BuildKit) only.

---

## 10. Constraints and Assumptions

**C-1:** The pipeline runs on ADO self-hosted ephemeral agents with Docker (BuildKit) available and outbound access to ACR, Azure Key Vault, and the Azure Artifacts npm feed. Agent-local caches do not persist between runs.

**C-2:** The shared ACR is a single registry used by all environments. Image promotion across environments is achieved by updating the image reference in the Kustomize config repo, not by copying images between registries.

**C-3:** Tenant repositories conform to the standard ADO project and repository naming convention. Non-conforming repositories require a platform exemption before they can use the shared build template.

**C-4:** Cosign key-based signing is the implementation for v1. Keyless signing via OIDC workload identity is deferred (see Appendix C).

**C-5:** Multi-architecture image builds (linux/amd64 + linux/arm64) are out of scope for v1. All images are built for linux/amd64 only.

**C-6:** Base image selection is the responsibility of each tenant team. Enforcement of base image compliance is deferred to the downstream security scan pipeline and Kyverno admission policy.

**C-7:** This pipeline does not gate on security scan results. Security scan results are advisory-only in v1. The security scan pipeline is the future enforcement gate.

**C-8:** The `--build-arg` mechanism MUST NOT be used to inject secrets into the image at build time. The npm auth token for Azure Artifacts MUST be passed via BuildKit `--secret`. Hadolint enforces this at ERROR level.

**C-9:** The composable template architecture (base template + runtime step templates) is the only supported extensibility mechanism for adding new runtime support. Tenant teams MUST NOT fork the base template or create parallel build templates. New runtime support requires a platform engineering change.

**C-10:** Config repo image reference updates (Kustomize) are a manual step in v1. The pipeline output variable provides the full image digest reference for this update. Kargo-based automation is a future consideration (see Appendix C).

---

## 11. Dependencies

| ID | Dependency | Type | Status | Notes |
|---|---|---|---|---|
| D-1 | Shared ACR provisioned | Hard | Required | Single ACR instance. Per-tenant cache scope (`<tenantName>/<appName>:buildcache`) must be configured. |
| D-2 | Cosign key pair in Azure Key Vault | Hard | Required | Platform-managed Cosign private key must be provisioned in AKV before FR-7.1 can execute. Key rotation process must be documented. |
| D-3 | ADO platform templates repository | Hard | Required | The shared pipeline template repository must be provisioned and access-controlled before any tenant can reference it. |
| D-4 | `platform-tool-versions` variable group | Hard | Required | Must include: Docker/BuildKit, Syft, Cosign, Hadolint versions, Azure Artifacts npm feed URL. |
| D-5 | Security scan pipeline | Hard | Required | Must be operational and configured to trigger via ADO pipeline completion trigger before SM-7 can be met. |
| D-6 | Kyverno ImageVerification policy | Soft | Planned | PRD-2026-KYVERNO-POLICY-001. Without it, Cosign signing is best-effort only. |
| D-7 | ADO service connection to ACR | Hard | Required | Per-tenant scoped service connection (`<tenantName>/*` push scope) must be provisioned before tenant onboarding. |
| D-8 | ADO PR comment API access | Soft | Required | Pipeline service principal must have ADO REST API access to post PR comments (FR-10.1). |
| D-9 | Azure Artifacts npm feed | Hard | Required for Angular/React | Platform-managed npm proxy feed. URL stored in `platform-tool-versions`. Required before Angular or React builds can use dependency caching. |
| D-10 | Platform agent image | Hard | Required | Self-hosted ADO agent image with Docker (BuildKit). No language toolchains required on the agent image. |
| D-11 | Per-tenant notifications variable group | Hard | Required for FR-10.4 | Variable group named `tenant-<tenantName>-notifications` containing `TEAMS_WEBHOOK_URL`. Provisioned per tenant during onboarding. |

---

## 12. Open Questions

All open questions from v0.3.0 are resolved. Decisions are recorded below for traceability.

| ID | Question | Decision | Notes |
|---|---|---|---|
| OQ-1 | Semver trigger on Git tag push vs. version on project file? | Version-from-project-file is the pattern. No Git tag trigger needed. | Replaces Git-tag-driven semver entirely. |
| OQ-2 | Security scan pipeline trigger mechanism? | ADO pipeline completion trigger. | Security scan pipeline owns the trigger definition. |
| OQ-3 | ACR cache scope: per-tenant or shared? | Per-tenant, scoped to `<tenantName>/<appName>:buildcache`. | Eliminates cross-tenant cache poisoning risk. |
| OQ-4 | ACR retention policy? | Out of scope for this pipeline (NG-4). Deferred to platform operations. | ACR governance is a platform engineering operation. |
| OQ-5 | Base image enforcement in v1? | Defer to security scan pipeline and Kyverno admission. | No Hadolint hard block in v1. |
| OQ-6 | Keyless Cosign signing roadmap? | Key-based signing is durable v1 implementation. Keyless deferred. | Azure Workload Identity + Fulcio not yet configured. See Appendix C. |
| OQ-7 | Kustomize config repo update: automatic or manual? | Manual for v1. Kargo is directional intent only. | Pipeline output variable is the handoff point. See Appendix C. |
| OQ-8 | ADO notification channels? | Teams webhook via per-tenant variable group; non-sensitive config via pipeline YAML variables. | Existing Teams webhook patterns reused from current templates. |
| OQ-9 | Multi-arch builds for v1.1? | linux/amd64 only for v1. ARM deferred. | ARM nodes not currently scoped for the cluster. See Appendix C. |
| OQ-10 | Tenant onboarding flow? | Manual provisioning for v1. Single net-new tenant POC in current PI. Migration guide for existing tenants post-POC. | Platform engineering provisions ADO service connection and notifications variable group per tenant. |
| OQ-11 | Security scan SLA and deployment hold? | Advisory-only. No deployment hold until Kyverno enforcement is in place. | Security team aligned. Scan/deploy coordination handled in consultation between dev and security. |
| OQ-12 | Pattern A or Pattern B for Angular/React? | Pattern A (multi-stage Dockerfile) for all runtimes. Pattern B eliminated. | Agent image maintenance not feasible with current platform engineering capacity. |
| OQ-13 | Node.js version governance? | Tenants own Node version through Dockerfile `FROM` statement. No platform-managed Node on agent. | Pattern A eliminates the agent-side Node version management problem. |

---

## 13. Revision History

| Version | Date | Author | Summary |
|---|---|---|---|
| 0.1.0 | 2026-06-27 | Platform Engineering | Initial draft. Covers Dockerfile build with BuildKit, Trivy vulnerability scanning, Syft SBOM generation, Cosign signing and attestation, shared ACR publish, shared ADO template model, and dryRun mode. |
| 0.2.0 | 2026-06-27 | Platform Engineering | Removed security scanning from pipeline scope. Added section 6.5 (Security Scan Pipeline Handoff). |
| 0.3.0 | 2026-06-27 | Platform Engineering | Added runtime composability architecture. Introduced `runtimeType` parameter. Added Pattern A and Pattern B build patterns. |
| 0.4.0 | 2026-06-28 | Platform Engineering | Major architectural update. Pattern B eliminated — all runtimes now Pattern A (multi-stage Dockerfile). Spring Boot: two-invocation BuildKit for test result extraction; Maven and Gradle both supported. Versioning changed from Git-tag-driven semver to version-from-project-file; fail on overwrite on main; version+sha suffix on feature branches. npm caching via Azure Artifacts npm feed using BuildKit `--secret` (see NPM-CACHING-PATTERN.md). ACR cache scoped per tenant. Security scan trigger resolved as ADO pipeline completion trigger. All OQs resolved. Added Teams notification requirement (FR-10.4), Azure Artifacts dependency (D-9), notifications variable group (D-11). Added NFR-8 (agent independence). Updated C-1, C-4, C-7, C-8, C-9. Removed C-10 (Pattern B toolchain). Added Appendix C (Deferred and Future Decisions). |

---

## 14. Appendix A: Image Tagging Strategy Reference

### 14.1 Why `latest` Is Prohibited

The `latest` tag is mutable. Any push of a new image with `latest` silently replaces the previous reference. In a GitOps system where ArgoCD watches a specific image reference, a `latest` tag means the declared state in Git does not uniquely identify what is running. `latest` is incompatible with the platform's GitOps model and is prohibited at the pipeline level.

### 14.2 Primary Tag — Full Git SHA

The full 40-character Git commit SHA is the canonical image tag. Properties:

- **Immutable by convention.** A SHA tag should never be reused for a different image.
- **Traceable.** Any running container can be traced to its exact source commit, Dockerfile, and pipeline run.
- **Machine-friendly.** Used in Kustomize `images:` fields and Kyverno ImageVerification policies.

Usage: `<acr-host>/<tenant>/<app>:<40-char-sha>`

### 14.3 Version Tag — Project File Derived

The version tag is derived from the project's metadata file, not from Git tags. This gives development teams direct control over versioning.

**Main branch behaviour:** version tag is pushed as `<version>` (e.g. `1.2.3`). If this tag already exists in ACR, the pipeline MUST fail with a message instructing the tenant to bump the version in their project file. This enforces a one-version-per-commit discipline on main.

**Feature branch behaviour:** version tag is pushed as `<version>-<short-sha>` (e.g. `1.2.3-abc1234`). This is always unique. Multiple preview builds of the same version can coexist in ACR without collision.

**Go exception:** if no `VERSION` file is present in the repo root, the version tag is skipped entirely. The primary SHA tag and alias tag are still pushed.

### 14.4 Alias Tag — Branch + Short SHA

The alias tag (`main-abc1234`) provides a human-readable reference. Mutable by design — changes on each build. Never used in Kustomize or ArgoCD manifests. Useful for human navigation in the ACR portal.

### 14.5 Digest Reference — The Gold Standard

For the highest immutability guarantee, manifests should reference images by digest:

```yaml
images:
  - name: payment-processor
    newName: <acr-host>/payments/payment-processor
    digest: sha256:abc123...
```

Digest references cannot be spoofed by retagging. The pipeline output variable (FR-8.7) provides the full digest reference as the handoff to the manifest update step.

### 14.6 Tag Summary Table

| Tag Format | Example | Mutable | Used in Manifests | Purpose |
|---|---|---|---|---|
| Full SHA | `abc123...def456` (40 chars) | No | Yes (preferred) | Canonical, traceable reference |
| Version (main) | `1.2.3` | No (fail on overwrite) | Informational | Release version marker |
| Version (branch) | `1.2.3-abc1234` | Yes | No | Preview build reference |
| Branch + short SHA | `main-abc1234` | Yes | No | Human navigation |
| Digest | `sha256:abc123...` | No | Yes (gold standard) | Maximum immutability |
| `latest` | `latest` | Yes | **Prohibited** | Not permitted by this pipeline |

---

## 15. Appendix B: Runtime Support Reference

### 15.1 Runtime Summary Table

| Runtime | runtimeType | Build Pattern | Agent requirement | Primary Caching |
|---|---|---|---|---|
| Angular | `angular` | A — In-Dockerfile | Docker + BuildKit | Azure Artifacts npm feed + ACR layer cache |
| React | `react` | A — In-Dockerfile | Docker + BuildKit | Azure Artifacts npm feed + ACR layer cache |
| Spring Boot | `springboot` | A — In-Dockerfile | Docker + BuildKit | ACR layer cache |
| Python | `python` | A — In-Dockerfile | Docker + BuildKit | ACR layer cache |
| Go | `go` | A — In-Dockerfile | Docker + BuildKit | ACR layer cache |

### 15.2 Angular (`runtimeType: angular`)

**Pattern:** A — In-Dockerfile (self-contained)

**Step template responsibilities (`steps/runtime/angular.yml`):**

- Advisory check that `package.json` exists; warn if absent
- Read version from `package.json` `version` field; fail Stage 1 if not parseable
- Inject `NPM_REGISTRY` build arg and `npm_token` BuildKit secret into the `docker build` invocation
- No pre-build steps execute on the agent

**Dockerfile expectations:**

- Multi-stage: Node build stage using the Azure Artifacts npm feed via `--mount=type=secret,id=npm_token`
- `COPY package.json package-lock.json ./` before `COPY . .` to maximise layer cache reuse
- `RUN npm ci` with Azure Artifacts registry configured via secret mount
- `ng build --configuration production` (or equivalent)
- Final stage: `nginx:alpine` serving the compiled static assets
- The `dist/` output directory exists in the final stage

**Caching:** Azure Artifacts npm feed serves cached packages; ACR layer cache (`--cache-from`/`--cache-to`) caches the `npm ci` layer when `package-lock.json` is unchanged. See `docs/NPM-CACHING-PATTERN.md`.

### 15.3 React (`runtimeType: react`)

**Pattern:** A — In-Dockerfile (self-contained)

**Step template responsibilities (`steps/runtime/react.yml`):**

- Advisory check that `package.json` exists; warn if absent
- Read version from `package.json` `version` field; fail Stage 1 if not parseable
- Detect Next.js SSR: if `next.config.js` is present without `output: 'export'`, emit a warning that SSR requires a Node.js runtime in the final image (not nginx)
- Inject `NPM_REGISTRY` build arg and `npm_token` BuildKit secret into the `docker build` invocation
- No pre-build steps execute on the agent

**Dockerfile expectations:**

- Multi-stage: Node build stage using the Azure Artifacts npm feed via `--mount=type=secret,id=npm_token`
- `COPY package.json package-lock.json ./` before `COPY . .`
- `RUN npm ci` with Azure Artifacts registry configured; then framework build (`vite build`, `react-scripts build`, `next build`)
- Final stage: `nginx:alpine` for static exports; Node.js runtime for Next.js SSR

**Caching:** Same as Angular. See `docs/NPM-CACHING-PATTERN.md`.

### 15.4 Spring Boot (`runtimeType: springboot`)

**Pattern:** A — In-Dockerfile (self-contained) with two-invocation BuildKit for test results

**Step template responsibilities (`steps/runtime/springboot.yml`):**

- Detect build tool: Maven (`pom.xml` present) or Gradle (`gradlew` present); fail Stage 1 if neither found
- Read version:
  - Maven: `<version>` element in `pom.xml`
  - Gradle: `version` property in `build.gradle` or `gradle.properties`
- Validate `test-export` stage exists in Dockerfile; fail Stage 2 if absent
- Execute two-invocation BuildKit build:
  1. `docker build --target test-export --output type=local,dest=./test-results .`
  2. ADO `PublishTestResults` — test failure blocks Stage 2
  3. `docker build --target final .`

**Dockerfile expectations (Gradle variant):**

```dockerfile
FROM gradle:8-jdk21 AS builder
WORKDIR /app
COPY . .
RUN ./gradlew test bootJar

FROM scratch AS test-export
COPY --from=builder /app/build/test-results ./

FROM eclipse-temurin:21-jre-alpine AS final
COPY --from=builder /app/build/libs/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Dockerfile expectations (Maven variant):**

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /app
COPY . .
RUN ./mvnw verify

FROM scratch AS test-export
COPY --from=builder /app/target/surefire-reports ./

FROM eclipse-temurin:21-jre-alpine AS final
COPY --from=builder /app/target/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Caching:** Docker layer cache on ACR. The `COPY . .` + `RUN ./gradlew` or `./mvnw` layer caches when source is unchanged. Gradle and Maven dependency layers should be separated from source layers where possible for maximum cache reuse.

**Test results:** Published to ADO Tests tab via `PublishTestResults` between the two `docker build` invocations. Test failure blocks the pipeline before the final image is built.

### 15.5 Python (`runtimeType: python`)

**Pattern:** A — In-Dockerfile (self-contained)

**Step template responsibilities (`steps/runtime/python.yml`):**

- Advisory check for `requirements.txt`, `pyproject.toml`, or `poetry.lock`; warn if absent, do not block
- Read version from `pyproject.toml` (`[project] version` or `[tool.poetry] version`) or `setup.cfg` (`version =`)
- No pre-build steps execute on the agent

**Dockerfile expectations:**

- Multi-stage: `python:<version>-slim` build stage for `pip install` → slim or distroless final stage
- Non-root user in final stage (enforced by Hadolint)

**Caching:** ACR layer cache. `COPY requirements.txt` + `RUN pip install` caches naturally when dependencies are unchanged.

### 15.6 Go (`runtimeType: go`)

**Pattern:** A — In-Dockerfile (self-contained)

**Step template responsibilities (`steps/runtime/go.yml`):**

- Assert `go.mod` exists in build context; fail Stage 1 if absent
- Read version from `VERSION` file in repo root if present; skip version tag if `VERSION` file is absent
- No pre-build steps execute on the agent

**Dockerfile expectations:**

- Multi-stage: `golang:<version>-alpine` build stage → `distroless/static` or `scratch` final stage
- `CGO_ENABLED=0 GOOS=linux` set in the build stage for static binary compilation
- `COPY go.mod go.sum ./` + `RUN go mod download` separated from `COPY . .` for maximum cache reuse

**Caching:** ACR layer cache. The `go mod download` layer caches naturally when the module graph is unchanged.

### 15.7 Adding a New Runtime

1. Platform engineering authors a new step template at `steps/runtime/<newRuntime>.yml`.
2. The base template (`container-build-v2.yml`) is updated to add a dispatch `${{ if }}` branch.
3. The `runtimeType` allowlist in FR-2.5 is updated.
4. This PRD is updated with a new Appendix B section and revision history entry.
5. Tenant teams MUST NOT add new runtimes directly. All new runtime support goes through platform engineering review.

---

## 16. Appendix C: Deferred and Future Decisions

The following decisions are explicitly out of scope for v1 and are recorded here for future planning.

**Keyless Cosign signing (Azure Workload Identity + Fulcio)**
Key-based signing via Azure Key Vault is the durable v1 implementation. Keyless signing ties the signature to the ADO pipeline's OIDC identity and eliminates the static key management dependency. This requires Azure Workload Identity federation wired to ADO — not currently configured on the platform. Treat as a future migration path when Azure Workload Identity is onboarded.

**Kargo-based image promotion**
Config repo image reference updates (Kustomize) are manual in v1. Kargo is a strong directional consideration given the Akuity platform relationship but has no current POC or evaluation timeline. The pipeline output variable (FR-8.7) is the designed handoff point for Kargo when it is adopted.

**ARM multi-architecture builds**
linux/amd64 only for v1. ARM node pool adoption is not currently scoped for the cluster. Multi-arch builds (`--platform linux/amd64,linux/arm64`) require either QEMU emulation or native ARM build agents. Revisit when ARM node pool adoption is confirmed.

**Hadolint base image enforcement**
Enforcing approved base image lists via Hadolint `trustedRegistries` config is deferred to the security scan pipeline and Kyverno admission enforcement. No compliance requirement mandates build-time enforcement in v1.

**Self-service tenant onboarding automation**
Manual provisioning (ADO service connection, notifications variable group) is acceptable for the initial rollout given the single-tenant POC scope. Self-service automation can be built once the manual process is validated and tenant volume warrants it.

**Existing tenant migration tooling**
A migration guide for existing tenants (Dockerfile changes, `azure-pipelines.yml` restructure, version tag convention, Spring Boot `test-export` stage) is a post-POC deliverable, not a pre-requisite for the initial rollout.

**Maven/Gradle dependency caching via Azure Artifacts**
The Azure Artifacts npm caching pattern (see NPM-CACHING-PATTERN.md) has a direct parallel for Maven and Gradle. Azure Artifacts supports Maven and Gradle feeds. The same BuildKit `--secret` mechanism applies using `settings.xml` (Maven) or Gradle init scripts. Defer to a follow-on pattern document after the npm pattern is validated in production.
