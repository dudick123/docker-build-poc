## Context

The CI container build pipeline (PRD-2026-CI-BUILD-001) is delivered as a set of ADO shared YAML templates. This is a greenfield implementation — no prior `container-build-v2.yml` exists. Phase 1 establishes the entry-point template that all tenant pipelines will reference via ADO's `extends:` keyword. Every subsequent phase (setup validation, lint, build, runtime-specific logic, sign/publish) plugs into the stage structure created here.

ADO's shared template system requires the base template to be syntactically valid YAML and reference only files that exist in the template repository. This means all step template references (`steps/setup.yml`, `steps/dockerfile-lint.yml`, etc.) must exist as at least empty stubs before the skeleton will load in ADO without errors.

Stakeholders: platform engineering team (owns templates), tenant teams (consume via `extends:`).

## Goals / Non-Goals

**Goals:**
- Define the complete `parameters:` block — the stable contract between tenant pipelines and the shared template
- Define the five-stage pipeline skeleton with correct `dependsOn` chains and `condition` expressions
- Wire `dryRun=true` skip conditions on Stage 3 (Sign & Attest) and Stage 4 (Publish)
- Create stub files for all step templates so the skeleton loads in ADO without missing-file errors
- Establish the `${{ if }}` runtime dispatch block routing `runtimeType` to `steps/runtime/<runtime>.yml`

**Non-Goals:**
- Any actual implementation of setup, lint, build, sign, publish, or notify logic (Phases 2–8)
- Runtime-specific validation or behavior (Phases 4–6)
- Variable group resolution or parameter validation (Phase 2)
- Infrastructure provisioning (ACR, Key Vault, variable groups)

## Decisions

### Decision 1: Six parameters only, all platform controls internal

The `parameters:` block exposes exactly six tenant-facing parameters: `tenantName`, `appName`, `runtimeType`, `dockerfilePath`, `buildContext`, `dryRun`. All platform controls (ACR endpoint, Cosign key reference, tag convention, tool versions) are hard-coded inside the template or resolved from `platform-tool-versions` — not exposed as parameters.

**Rationale:** Limiting the parameter surface locks tenants out of platform controls, preventing accidental misconfiguration and enforcing security posture consistently. Alternatives considered: exposing `acrEndpoint` as a parameter (rejected — allows tenants to push to arbitrary registries), exposing `toolVersions` (rejected — bypasses centralized version pinning).

### Decision 2: Five named stages with explicit `dependsOn`

Stages are named `Setup`, `Build`, `SignAndAttest`, `Publish`, `Notify`. Each stage declares explicit `dependsOn` rather than relying on implicit sequential ordering, so conditions can be applied independently.

**Rationale:** ADO `condition:` expressions on stages require `dependsOn` to be explicit when the condition deviates from the default `succeeded()`. Using explicit `dependsOn` also makes the dependency graph readable without needing to count stage positions.

### Decision 3: `dryRun` implemented as stage-level condition, not step-level skip

`dryRun=true` skips Stages 3 and 4 entirely via `condition: and(succeeded(), eq('${{ parameters.dryRun }}', 'false'))`. Stage 5 (Notify) runs in dry-run mode but posts a dry-run marker comment.

**Rationale:** Skipping entire stages rather than individual steps makes the dry-run boundary clear in the ADO pipeline graph UI and avoids conditional logic scattered across step templates. The notify stage still fires so tenants get feedback even on dry runs.

### Decision 4: Runtime dispatch via compile-time `${{ if }}` expressions

The runtime dispatch uses ADO compile-time template expressions (`${{ if eq(parameters.runtimeType, 'go') }}`) rather than runtime conditions. Each branch calls the corresponding `steps/runtime/<runtime>.yml` template.

**Rationale:** Compile-time dispatch means only the matching runtime template is included in the expanded pipeline — unreferenced templates are not loaded. This keeps pipeline YAML size manageable and prevents runtime template errors from affecting unrelated builds. Alternative (runtime `condition:` on steps) rejected because it would load all runtime templates regardless of `runtimeType`.

### Decision 5: Stub files are minimal valid ADO step templates

Each stub file contains only the minimum ADO YAML required for the template to be referenced without error: a `steps:` key with a single no-op `bash` step echoing the stub name. This is replaced by the real implementation in subsequent phases.

**Rationale:** Empty files are not valid ADO step template YAML. A minimal valid stub allows Phase 1 to be validated end-to-end in ADO without waiting for later phases.

## Risks / Trade-offs

- **Stub files must be replaced before production use** — if a later phase is skipped or delayed, a stub may accidentally reach a tenant pipeline. Mitigation: stubs emit a clearly visible warning message (`echo "STUB: <name> not yet implemented"`), making accidental use obvious in build logs.
- **Parameter contract is locked after tenant adoption** — adding or renaming parameters after tenants have adopted `extends:` is a breaking change requiring coordinated migration. Mitigation: the six parameters are derived from the finalized PRD; no known parameter changes are pending.
- **ADO compile-time expression limits** — `${{ if }}` dispatch works only when `runtimeType` is a literal value passed by the tenant pipeline, not a variable resolved at runtime. This is the correct usage pattern (tenants hardcode their runtime type) but must be documented.

## Migration Plan

1. Commit all files to the ADO platform-templates repository on a feature branch
2. Create a test tenant pipeline that references the branch via `extends: template@branch`
3. Run with `dryRun=true` and each `runtimeType` value to verify the template loads and the correct stub is dispatched
4. Merge to main; all subsequent phases branch from this baseline

No rollback complexity — this phase creates only new files. Rollback is branch deletion.

## Open Questions

None — all Phase 1 decisions are resolved per the implementation plan. Phase 2 open questions (variable group name format, validation error message text) are out of scope here.
