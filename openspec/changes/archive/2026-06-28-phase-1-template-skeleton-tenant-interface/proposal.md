## Why

The shared ADO container build pipeline needs a well-defined entry point that all tenant pipelines can reference via `extends:`. Phase 1 establishes `container-build-v2.yml` — the base template skeleton with the complete parameter contract, stage structure, and runtime dispatch scaffolding — so every subsequent phase has a stable foundation to build on and the template can be loaded and validated in ADO from day one.

## What Changes

- **New file** `platform-templates/container-build-v2.yml` — the tenant-facing base template with:
  - Full parameter block: `tenantName`, `appName`, `runtimeType`, `dockerfilePath`, `buildContext`, `dryRun`
  - Five-stage pipeline structure (Setup → Build → Sign & Attest → Publish → Notify) with correct dependency conditions
  - Runtime dispatch scaffolding: `${{ if }}` blocks for all five runtimes (`angular`, `react`, `springboot`, `python`, `go`) as stubs — each calls the appropriate `steps/runtime/<runtime>.yml` template (stub files created as empty templates, filled in by Phases 4–6)
  - `dryRun` skip conditions wired on Stages 3 (Sign & Attest) and 4 (Publish)
  - Stub `steps/` template references (linting, build, sbom-sign-publish) so the skeleton compiles without missing-template errors

## Capabilities

### New Capabilities

- `tenant-interface`: The parameter contract exposed to tenant pipelines — the six parameters (`tenantName`, `appName`, `runtimeType`, `dockerfilePath`, `buildContext`, `dryRun`), their types, defaults, and allowed values, as documented in the base template's `parameters:` block.
- `stage-structure`: The five-stage pipeline skeleton with stage names, `dependsOn` chains, and `dryRun` condition logic that all later phases plug into.
- `runtime-dispatch`: The `${{ if }}` dispatch block that routes each `runtimeType` value to its corresponding `steps/runtime/<runtime>.yml` template; stubs only at this phase.

### Modified Capabilities

## Impact

- **New file:** `platform-templates/container-build-v2.yml`
- **New stub files:** `platform-templates/steps/setup.yml`, `platform-templates/steps/dockerfile-lint.yml`, `platform-templates/steps/docker-build.yml`, `platform-templates/steps/sbom-sign-publish.yml`, `platform-templates/steps/runtime/angular.yml`, `platform-templates/steps/runtime/react.yml`, `platform-templates/steps/runtime/springboot.yml`, `platform-templates/steps/runtime/python.yml`, `platform-templates/steps/runtime/go.yml`
- **No existing code changed** — this is a greenfield deliverable
- **ADO dependency:** template must be hosted in the ADO platform-templates repository and referenceable via `extends:` from tenant pipelines
- **No hard infrastructure dependencies at this phase** — no ACR, Key Vault, or variable groups required; skeleton validates purely on template syntax
