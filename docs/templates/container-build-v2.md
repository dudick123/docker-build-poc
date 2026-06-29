# Template: container-build-v2.yml

**Path:** `platform-templates/container-build-v2.yml`
**Type:** Base template (tenant entry point)
**Consumed via:** `extends:` in tenant `azure-pipelines.yml`

---

## Description

The base template is the single artifact that tenant pipelines reference. It defines all five pipeline stages, wires cross-stage output variables, enforces `dryRun` skip conditions, and dispatches to the correct runtime step template at compile time. Tenant pipelines supply six parameters and nothing else — all platform controls are locked inside this template.

---

## Summary

| Property | Value |
|---|---|
| Variable group consumed | `platform-tool-versions` |
| Stages | Setup → Build → Sign & Attest → Publish → Notify |
| Template expression style | Compile-time `${{ }}` for parameters; runtime `$()` for cross-stage outputs |
| Runtime dispatch mechanism | `${{ if eq(parameters.runtimeType, ...) }}` at compile time |
| Security boundary | All `${{ parameters.xxx }}` values enter bash steps via `env:` only — never inlined into script bodies |

---

## Parameters

| Name | Type | Default | Required | Description |
|---|---|---|---|---|
| `tenantName` | string | — | yes | Team/tenant slug. Pattern: `^[a-z0-9][a-z0-9-]*[a-z0-9]$` |
| `appName` | string | — | yes | Application name. Same pattern as `tenantName` |
| `runtimeType` | string | — | yes | One of: `go` `python` `springboot` `angular` `react` |
| `dockerfilePath` | string | `Dockerfile` | no | Path to Dockerfile, relative to `buildContext` |
| `buildContext` | string | `.` | no | Docker build context directory, relative to repo root |
| `dryRun` | boolean | `false` | no | When `true`, skips SignAndAttest and Publish; Notify still runs |

---

## Stages

### Setup

```yaml
- stage: Setup
  displayName: 'Setup'
  jobs:
    - job: Setup
```

Calls `steps/setup.yml`. Validates all six tenant parameters and resolves tool versions from the `platform-tool-versions` variable group. Emits six output variables consumed by downstream stages.

**Condition:** Always runs.

**Downstream stages abort if Setup fails** — all stages declare `dependsOn: Setup` (directly or transitively).

---

### Build

```yaml
- stage: Build
  displayName: 'Build'
  dependsOn: Setup
```

Calls `steps/dockerfile-lint.yml`, `steps/docker-build.yml`, and the active `steps/runtime/<runtimeType>.yml` in that order. The runtime template is selected at compile time:

```yaml
- ${{ if eq(parameters.runtimeType, 'go') }}:
  - template: steps/runtime/go.yml
- ${{ if eq(parameters.runtimeType, 'python') }}:
  - template: steps/runtime/python.yml
# ...and so on for all five runtimes
```

**Condition:** Runs when Setup succeeds (implicit `dependsOn`).

**Key output variable:** `$(stageDependencies.Build.Build.outputs['buildImage.IMAGE_DIGEST'])`

---

### Sign & Attest

```yaml
- stage: SignAndAttest
  displayName: 'Sign & Attest'
  dependsOn: Build
  condition: and(succeeded(), eq('${{ parameters.dryRun }}', 'false'))
```

Calls `steps/sbom-sign-publish.yml` with `phase: signAndAttest`. Generates the SBOM, retrieves Cosign keys from Azure Key Vault, pushes the image under a short-SHA staging tag, captures the manifest digest, signs, attests, and verifies.

**Condition:** Skipped when `dryRun: true` (evaluated at compile time) or when Build failed.

**Key output variable:** `$(stageDependencies.SignAndAttest.SignAndAttest.outputs['signAttest.MANIFEST_DIGEST'])`

---

### Publish

```yaml
- stage: Publish
  displayName: 'Publish'
  dependsOn: SignAndAttest
  condition: and(succeeded(), eq('${{ parameters.dryRun }}', 'false'))
```

Calls `steps/sbom-sign-publish.yml` with `phase: publish`. Asserts no `latest` tag, pushes the full tag set, verifies manifest digest, and writes the provenance artifact.

The job includes a compile-time `${{ if/elseif }}` `variables:` block that selects the correct runtime version output variable name based on `runtimeType`, then passes a single `$(RUNTIME_VERSION)` to the template:

```yaml
variables:
  - ${{ if eq(parameters.runtimeType, 'go') }}:
    - name: RUNTIME_VERSION
      value: $(stageDependencies.Build.Build.outputs['goRuntime.GO_VERSION'])
  - ${{ elseif eq(parameters.runtimeType, 'python') }}:
    - name: RUNTIME_VERSION
      value: $(stageDependencies.Build.Build.outputs['pythonRuntime.PYTHON_VERSION'])
  # ...
```

**Condition:** Skipped when `dryRun: true` or when SignAndAttest failed/skipped.

**Key output variable:** `$(stageDependencies.Publish.Publish.outputs['publish.IMAGE_REF'])`

---

### Notify

```yaml
- stage: Notify
  displayName: 'Notify'
  dependsOn: Publish
  condition: in(dependencies.Publish.result, 'Succeeded', 'Failed', 'SucceededWithIssues', 'Skipped')
```

Calls `steps/sbom-sign-publish.yml` with `phase: notify`. Posts a PR comment and sends an optional Teams notification.

**Condition:** Runs after Publish regardless of Publish result — including when Publish is skipped (dry run). Uses `in(dependencies.Publish.result, ...)` rather than `succeededOrFailed()` because `succeededOrFailed()` does not fire when the upstream stage is in a skipped state.

---

## Cross-Stage Output Variable Reference

All cross-stage output variables follow this pattern:

```
$(stageDependencies.<StageName>.<JobName>.outputs['<stepName>.<VARNAME>'])
```

| Variable | Reference |
|---|---|
| `ACR_HOST` | `$(stageDependencies.Setup.Setup.outputs['resolveTools.ACR_HOST'])` |
| `HADOLINT_VERSION` | `$(stageDependencies.Setup.Setup.outputs['resolveTools.HADOLINT_VERSION'])` |
| `SYFT_VERSION` | `$(stageDependencies.Setup.Setup.outputs['resolveTools.SYFT_VERSION'])` |
| `COSIGN_VERSION` | `$(stageDependencies.Setup.Setup.outputs['resolveTools.COSIGN_VERSION'])` |
| `NPM_REGISTRY_URL` | `$(stageDependencies.Setup.Setup.outputs['resolveTools.NPM_REGISTRY_URL'])` |
| `IMAGE_DIGEST` | `$(stageDependencies.Build.Build.outputs['buildImage.IMAGE_DIGEST'])` |
| `GO_VERSION` | `$(stageDependencies.Build.Build.outputs['goRuntime.GO_VERSION'])` |
| `PYTHON_VERSION` | `$(stageDependencies.Build.Build.outputs['pythonRuntime.PYTHON_VERSION'])` |
| `SPRINGBOOT_VERSION` | `$(stageDependencies.Build.Build.outputs['springbootRuntime.SPRINGBOOT_VERSION'])` |
| `ANGULAR_VERSION` | `$(stageDependencies.Build.Build.outputs['angularRuntime.ANGULAR_VERSION'])` |
| `REACT_VERSION` | `$(stageDependencies.Build.Build.outputs['reactRuntime.REACT_VERSION'])` |
| `MANIFEST_DIGEST` | `$(stageDependencies.SignAndAttest.SignAndAttest.outputs['signAttest.MANIFEST_DIGEST'])` |
| `IMAGE_REF` | `$(stageDependencies.Publish.Publish.outputs['publish.IMAGE_REF'])` |

---

## Usage

```yaml
# azure-pipelines.yml in tenant repository

trigger:
  branches:
    include:
      - main

resources:
  repositories:
    - repository: platform-templates
      type: git
      name: <YourADOProject>/platform-templates

extends:
  template: container-build-v2.yml@platform-templates
  parameters:
    tenantName: payments
    appName: payment-processor
    runtimeType: go
```

Optional parameters:

```yaml
extends:
  template: container-build-v2.yml@platform-templates
  parameters:
    tenantName: payments
    appName: payment-processor
    runtimeType: springboot
    buildContext: services/payment-processor   # non-root build context
    dockerfilePath: docker/Dockerfile           # non-default Dockerfile name/path
    dryRun: true                                # skip sign/publish for PR validation
```

---

## Design Notes

- The `extends:` pattern in ADO prevents tenants from adding arbitrary stages or jobs. Tenants can only pass the six declared parameters.
- All `${{ parameters.xxx }}` compile-time expressions are expanded before the pipeline YAML is handed to the ADO runtime. This means `runtimeType` dispatch and `dryRun` stage conditions are resolved at queue time, not at runtime.
- The `platform-tool-versions` variable group is loaded at the template level (`variables: - group: platform-tool-versions`) so its values are available to all stages without requiring the group to be referenced in each job.
