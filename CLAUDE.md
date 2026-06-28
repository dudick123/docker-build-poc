# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Status

This repository is in the **planning/POC phase**. Currently it contains only design documents. No pipeline YAML, scripts, or tooling has been implemented yet.

- `docs/PRD.md` — Full product requirements (PRD-2026-CI-BUILD-001 v0.3.0)
- `docs/PROJECT.md` — Condensed architecture summary; start here for quick context

## What This Builds

A shared Azure DevOps (ADO) YAML pipeline template for container image builds across a multi-tenant AKS platform. The pipeline: lints Dockerfiles, builds images with BuildKit, generates SBOMs via Syft, signs images via Cosign, and pushes to a shared Azure Container Registry. Security scanning (Trivy/Nexus/Fortify) is explicitly out of scope — it runs in a separate downstream pipeline.

## Target Template Structure

When implementation begins, the deliverable is an ADO shared template set:

```
platform-templates/
  container-build-v2.yml          ← tenant entry point (base template)
  steps/
    setup.yml                     ← tool resolution, parameter validation
    dockerfile-lint.yml           ← Hadolint
    docker-build.yml              ← BuildKit build + OCI label injection
    sbom-sign-publish.yml         ← Syft, Cosign, ACR push, notify
    runtime/
      angular.yml                 ← Pattern B: npm ci + ng build
      react.yml                   ← Pattern B: npm ci + framework build
      springboot.yml              ← Pattern B: ./gradlew bootJar + test publish
      python.yml                  ← Pattern A: pass-through (build in Dockerfile)
      go.yml                      ← Pattern A: pass-through (build in Dockerfile)
```

## Pipeline Stages

1. **Setup** — resolve tool versions from `platform-tool-versions` ADO variable group, validate parameters
2. **Build** — runtime pre-build (Pattern B only), Hadolint lint, `docker build` with BuildKit
3. **Sign & Attest** — Syft SBOM (CycloneDX JSON), `cosign sign`, `cosign attest` — skipped when `dryRun=true`
4. **Publish** — ACR push, digest verification, provenance summary — skipped when `dryRun=true`
5. **Notify** — PR comment, trigger downstream security scan pipeline

## Runtime Build Patterns

**Pattern A — full build inside Dockerfile** (Go, Python): pipeline agent only needs Docker; no language toolchain required on agent.

**Pattern B — artifact built on agent, copied into image** (Angular, React, Spring Boot): pre-build step runs on agent before `docker build`. Dockerfile receives the pre-built artifact via `COPY`.

## Key Design Constraints

- **Tool versions:** always resolved from `platform-tool-versions` ADO variable group — never hard-coded in YAML
- **BuildKit required:** legacy `docker build` is not permitted
- **Image held locally until Stage 4:** do not push to ACR before Stage 4 completes
- **Digest carry-through:** all downstream steps (sign, push) reference `sha256:<digest>`, never a tag
- **`latest` tag is prohibited:** enforced as a pipeline assertion before push
- **No `--build-arg` secrets:** runtime secrets delivered via ESO + Azure Key Vault only; Hadolint enforces this at ERROR level
- **Tenant parameters are limited:** only `tenantName`, `appName`, `runtimeType`, `dockerfilePath`, `buildContext`, `dryRun` — all platform controls (Cosign key, ACR endpoint, tag convention) are locked inside the base template

## Image Naming Convention

- Repository: `<acr>/<tenantName>/<appName>`
- Primary tag: full 40-char Git SHA (immutable; used in Kustomize manifests)
- Alias tag: `<branch>-<short-sha>` (human navigation only, never in manifests)
- Semver tag: `v<major>.<minor>.<patch>` — only when commit carries a semver Git tag

## Adding a New Runtime

1. Author `steps/runtime/<newRuntime>.yml`
2. Add a dispatch `${{ if }}` branch in `container-build-v2.yml`
3. Update the `runtimeType` allowlist validation in Stage 1
4. Update `platform-tool-versions` variable group if new agent toolchain versions are needed
5. Update platform agent image if Pattern B (D-10)

## Hard Dependencies (not in this repo)

| Dependency | Notes |
|---|---|
| Shared ACR | Single registry for all tenant builds |
| Cosign key pair in Azure Key Vault | Platform engineering manages; retrieved at sign time via ADO Key Vault task |
| ADO platform templates repository | Where these templates live and are referenced via `extends:` |
| `platform-tool-versions` variable group | Pins Docker/BuildKit, Syft, Cosign, Hadolint versions |
| Per-tenant ADO service connection | Scoped to `<tenantName>/*` push only |
| Platform agent image | Includes Node.js LTS, JDK, BuildKit for Pattern B runtimes |
