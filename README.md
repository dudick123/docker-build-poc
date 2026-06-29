# Platform Container Build Pipeline v2

Shared Azure DevOps YAML pipeline template for container image builds across the multi-tenant AKS platform. Tenant teams consume this template via `extends:` — one six-line `azure-pipelines.yml` is all that is required in an application repository.

**What the pipeline does:**

1. Validates parameters and resolves pinned tool versions from the `platform-tool-versions` variable group
2. Lints the Dockerfile with Hadolint (ERROR findings fail the build; WARNING findings are logged)
3. Builds the container image with BuildKit (rootless, with ACR layer cache and OCI labels)
4. Generates a CycloneDX SBOM using Syft
5. Signs the image manifest and attaches the SBOM as an OCI attestation using Cosign (keys from Azure Key Vault)
6. Pushes the full tag set to the shared ACR and verifies manifest digest integrity
7. Posts a build summary to the triggering PR and optionally notifies a Teams channel

Security scanning (Trivy, Nexus, Fortify) runs in a separate downstream pipeline against images already in ACR and is out of scope here.

---

## Supported Runtimes

| `runtimeType` | Build pattern | Notes |
|---|---|---|
| `go` | Multi-stage Dockerfile | `go.mod` required at build context root |
| `python` | Multi-stage Dockerfile | `pyproject.toml`, `requirements.txt`, or `poetry.lock` recommended |
| `springboot` | Multi-stage Dockerfile | `gradlew` or `pom.xml` required; Dockerfile must include `AS test-export` stage |
| `angular` | Multi-stage Dockerfile | npm credentials auto-injected via BuildKit secret |
| `react` | Multi-stage Dockerfile | npm credentials auto-injected via BuildKit secret; Next.js SSR apps require `output: export` |

All runtimes build entirely inside the Dockerfile. The pipeline agent requires only Docker and BuildKit — no language toolchains are installed on the agent.

---

## Minimal Usage

Add `azure-pipelines.yml` to your application repository root:

```yaml
trigger:
  branches:
    include:
      - main

extends:
  template: templates/container-build-v2.yml@platform-templates
  parameters:
    tenantName: payments          # your team slug — lowercase, hyphens allowed
    appName: payment-processor    # your app name — lowercase, hyphens allowed
    runtimeType: go               # go | python | springboot | angular | react
```

That is the entire tenant pipeline definition. All platform controls (ACR endpoint, Cosign key, tag convention, tool versions) are locked inside the base template.

---

## Pipeline Parameters

| Parameter | Type | Default | Required | Description |
|---|---|---|---|---|
| `tenantName` | string | — | yes | Team/tenant slug. Pattern: `^[a-z0-9][a-z0-9-]*[a-z0-9]$` |
| `appName` | string | — | yes | Application name. Same pattern as `tenantName` |
| `runtimeType` | string | — | yes | One of: `go`, `python`, `springboot`, `angular`, `react` |
| `dockerfilePath` | string | `Dockerfile` | no | Path to Dockerfile, relative to `buildContext` |
| `buildContext` | string | `.` | no | Docker build context, relative to repository root |
| `dryRun` | boolean | `false` | no | When `true`, skips sign, publish, and all ACR writes |

---

## Pipeline Stages

```
Setup → Build → Sign & Attest → Publish → Notify
```

| Stage | Skipped when | Description |
|---|---|---|
| Setup | never | Tool-version resolution + parameter validation |
| Build | Setup fails | Hadolint lint + BuildKit image build + runtime validation |
| Sign & Attest | `dryRun: true` or Build fails | Syft SBOM, Cosign sign, Cosign attest |
| Publish | `dryRun: true` or Sign fails | ACR tag push, digest verify, provenance artifact |
| Notify | never | PR comment + Teams notification (dry-run aware) |

---

## Image Tags Produced

For every successful (non-dry-run) pipeline run, three tags are pushed to ACR at `<acr>/<tenantName>/<appName>`:

| Tag | Format | Purpose |
|---|---|---|
| Primary | `<full-40-char-git-sha>` | Immutable; used in Kustomize manifests |
| Alias | `<branch>-<12-char-sha>` | Human navigation; not for manifests |
| Version | `<version>` (main) or `<version>-<12-char-sha>` (other branches) | Omitted if project has no version file |

The `latest` tag is prohibited and enforced as a pipeline assertion before any push.

---

## Pipeline Artifacts

Every successful run produces two ADO pipeline artifacts:

| Artifact | Name | Contents |
|---|---|---|
| SBOM | `sbom-<tenantName>-<appName>` | CycloneDX JSON (all image packages including base) |
| Provenance | `provenance-<tenantName>-<appName>` | JSON: digest, tags, artifact locations, git commit |

The signed manifest digest is also set as an ADO build tag on the pipeline run for correlation.

---

## Dry Run

Set `dryRun: true` to test the lint and build stages without touching ACR:

```yaml
extends:
  template: templates/container-build-v2.yml@platform-templates
  parameters:
    tenantName: payments
    appName: payment-processor
    runtimeType: go
    dryRun: true
```

Sign & Attest and Publish stages are skipped. The Notify stage still runs and posts a "Dry Run" summary to the PR.

---

## Platform Dependencies

These must be in place before any tenant can use the template:

| Dependency | Owner | Notes |
|---|---|---|
| `platform-tool-versions` variable group | Platform Engineering | Pins Docker/BuildKit, Syft, Cosign, Hadolint versions + ACR host + npm registry URL |
| Shared ACR | Platform Engineering | Single registry; all tenant images land here |
| Cosign key pair in Azure Key Vault | Platform Engineering | `cosign-private-key` + `cosign-public-key` secrets; retrieved per-run |
| `COSIGN_AKV_SERVICE_CONNECTION` | Platform Engineering | ADO service connection to the Key Vault subscription |
| `COSIGN_KEY_VAULT_NAME` | Platform Engineering | Name of the AKV instance holding the Cosign keys |
| Per-tenant ADO service connection | Platform Engineering | Scoped to `<tenantName>/*` push in ACR |
| `tenant-<name>-notifications` variable group | Tenant (optional) | Set `TEAMS_WEBHOOK_URL` for Teams notifications |

---

## Quick-Start Guides

| Runtime | Guide |
|---|---|
| All runtimes (start here) | [docs/QUICK-START.md](docs/QUICK-START.md) |
| Go | [docs/QUICK-START-GO.md](docs/QUICK-START-GO.md) |
| Python | [docs/QUICK-START-PYTHON.md](docs/QUICK-START-PYTHON.md) |
| Angular | [docs/QUICK-START-ANGULAR.md](docs/QUICK-START-ANGULAR.md) |
| React / Next.js | [docs/QUICK-START-REACT.md](docs/QUICK-START-REACT.md) |
| Spring Boot (Gradle) | [docs/QUICK-START-GRADLE.md](docs/QUICK-START-GRADLE.md) |
| Spring Boot (Maven) | [docs/QUICK-START-MAVEN.md](docs/QUICK-START-MAVEN.md) |

---

## Template Structure

```
platform-templates/
  container-build-v2.yml          ← tenant entry point (base template)
  steps/
    setup.yml                     ← tool resolution, parameter validation
    dockerfile-lint.yml           ← Hadolint
    docker-build.yml              ← BuildKit build + OCI label injection
    sbom-sign-publish.yml         ← Syft, Cosign, ACR push, notify
    runtime/
      go.yml                      ← go.mod assertion + version extraction
      python.yml                  ← dependency file check + version extraction
      springboot.yml              ← build tool detection + two-invocation BuildKit
      angular.yml                 ← package.json check + version extraction
      react.yml                   ← package.json check + Next.js SSR detection
  test/
    tenant-go-dryrun.yml          ← reference dry-run pipeline (Go)
```

---

## Adding a New Runtime

1. Author `steps/runtime/<newRuntime>.yml`
2. Add a `${{ if eq(parameters.runtimeType, '<newRuntime>') }}` dispatch block in `container-build-v2.yml` (Build stage and Publish stage `variables:` block)
3. Add `<newRuntime>` to the `runtimeType` allowlist check in `steps/setup.yml`
4. Add any new tool version pins to the `platform-tool-versions` variable group
5. Open a platform engineering ticket to update the platform agent image if the new runtime requires agent-side tooling

---

## For Platform Engineering

Tool versions are never hard-coded in YAML. All version pins live in the `platform-tool-versions` ADO variable group. To update a tool version, update the variable group — no template YAML changes are required.

`${{ parameters.xxx }}` template expressions are never inlined into bash script bodies. All parameter values are passed exclusively through the `env:` block of each bash step to prevent template injection.
