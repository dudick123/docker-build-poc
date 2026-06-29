# Template: steps/setup.yml

**Path:** `platform-templates/steps/setup.yml`
**Type:** Step template
**Called by:** `container-build-v2.yml` — Stage 1 (Setup), Job: Setup

---

## Description

Performs all pre-build gate checks in a single Stage 1 job. There are two distinct concerns handled sequentially: resolving pinned tool versions from the `platform-tool-versions` ADO variable group, and validating the six tenant-supplied parameters. Any failure here aborts the entire pipeline before any build work starts, giving tenant teams fast, clear feedback without wasting agent minutes.

---

## Summary

| Property | Value |
|---|---|
| Steps | 2 (bash) |
| Named steps | `resolveTools`, `validateParams` |
| Output variables emitted | 6 (`ACR_HOST`, `HADOLINT_VERSION`, `SYFT_VERSION`, `COSIGN_VERSION`, `DOCKER_BUILDKIT_VERSION`, `NPM_REGISTRY_URL`) |
| Hard failures | Any empty tool variable; name pattern violation; unsupported runtimeType; Dockerfile not found; runtime template not found |
| Security note | All `${{ parameters.xxx }}` values enter bash via `env:` only |

---

## Parameters

| Name | Type | Default | Description |
|---|---|---|---|
| `tenantName` | string | — | Validated against `^[a-z0-9][a-z0-9-]*[a-z0-9]$` |
| `appName` | string | — | Validated against `^[a-z0-9][a-z0-9-]*[a-z0-9]$` |
| `runtimeType` | string | — | Validated against allowlist: `angular`, `react`, `springboot`, `python`, `go` |
| `dockerfilePath` | string | `Dockerfile` | Resolved as `<buildContext>/<dockerfilePath>` on the agent; must exist |
| `buildContext` | string | `.` | Used to resolve the Dockerfile path |

---

## Steps

### Step 1 — `resolveTools` (bash)

**Purpose:** Assert that all required entries in the `platform-tool-versions` variable group are populated and emit them as step output variables for downstream stages.

**What it checks:**

| Variable group key | Emitted as output variable |
|---|---|
| `DOCKER_BUILDKIT_VERSION` | `resolveTools.DOCKER_BUILDKIT_VERSION` |
| `SYFT_VERSION` | `resolveTools.SYFT_VERSION` |
| `COSIGN_VERSION` | `resolveTools.COSIGN_VERSION` |
| `HADOLINT_VERSION` | `resolveTools.HADOLINT_VERSION` |
| `NPM_REGISTRY_URL` | `resolveTools.NPM_REGISTRY_URL` |
| `ACR_HOST` | `resolveTools.ACR_HOST` |

If any variable is missing or empty, all violations are collected and emitted as ADO error log issues before exiting with code 1. The step does not short-circuit on the first error — it reports all missing variables at once.

**Output variables cross-stage reference syntax:**

```
$(stageDependencies.Setup.Setup.outputs['resolveTools.ACR_HOST'])
$(stageDependencies.Setup.Setup.outputs['resolveTools.HADOLINT_VERSION'])
$(stageDependencies.Setup.Setup.outputs['resolveTools.SYFT_VERSION'])
$(stageDependencies.Setup.Setup.outputs['resolveTools.COSIGN_VERSION'])
$(stageDependencies.Setup.Setup.outputs['resolveTools.NPM_REGISTRY_URL'])
$(stageDependencies.Setup.Setup.outputs['resolveTools.DOCKER_BUILDKIT_VERSION'])
```

---

### Step 2 — `validateParams` (bash)

**Purpose:** Validate the five tenant-supplied parameters before any build work starts. All validations are collected before exiting — multiple errors are reported together rather than one at a time.

**Validation rules:**

| Parameter | Rule | Failure message |
|---|---|---|
| `tenantName` | Must match `^[a-z0-9][a-z0-9-]*[a-z0-9]$` | `tenantName 'X' does not match required pattern...` |
| `appName` | Must match `^[a-z0-9][a-z0-9-]*[a-z0-9]$` | `appName 'X' does not match required pattern...` |
| `runtimeType` | Must be one of `angular`, `react`, `springboot`, `python`, `go` | `runtimeType 'X' is not supported. Allowed values: ...` |
| `dockerfilePath` | `$BUILD_SOURCESDIRECTORY/$BUILD_CONTEXT/$DOCKERFILE_PATH` must exist on agent | `dockerfilePath not found: <resolved-path>` |
| Runtime template | `$AGENT_BUILDDIRECTORY/s/platform-templates/steps/runtime/$RUNTIME_TYPE.yml` must exist | `Runtime template not found: steps/runtime/X.yml — contact platform engineering` |

The `tenantName` and `appName` pattern requires:
- Lowercase alphanumeric characters and hyphens only
- Minimum two characters total
- Cannot start or end with a hyphen

---

## Output Variables

All output variables are emitted from the `resolveTools` step with `isOutput=true` and are available to subsequent stages via the cross-stage reference syntax shown above.

| Variable | Description |
|---|---|
| `ACR_HOST` | FQDN of the shared Azure Container Registry (e.g., `myplatform.azurecr.io`) |
| `HADOLINT_VERSION` | Pinned Hadolint version string (e.g., `2.12.0`) |
| `SYFT_VERSION` | Pinned Syft version string (e.g., `1.4.1`) |
| `COSIGN_VERSION` | Pinned Cosign version string (e.g., `2.2.4`) |
| `DOCKER_BUILDKIT_VERSION` | Pinned Docker/BuildKit version string |
| `NPM_REGISTRY_URL` | Platform Azure Artifacts npm feed URL |

---

## Dependencies

- `platform-tool-versions` ADO variable group must be linked to the pipeline and must contain all six keys listed above.
- The Dockerfile must be committed at the resolved path before the pipeline runs.
- The runtime step template (`steps/runtime/<runtimeType>.yml`) must exist in the platform-templates repository.

---

## Usage

This template is called exclusively by `container-build-v2.yml`. It is not intended to be called directly by tenant pipelines.

```yaml
# Inside container-build-v2.yml
- stage: Setup
  jobs:
    - job: Setup
      steps:
        - template: steps/setup.yml
          parameters:
            tenantName: ${{ parameters.tenantName }}
            appName: ${{ parameters.appName }}
            runtimeType: ${{ parameters.runtimeType }}
            dockerfilePath: ${{ parameters.dockerfilePath }}
            buildContext: ${{ parameters.buildContext }}
```
