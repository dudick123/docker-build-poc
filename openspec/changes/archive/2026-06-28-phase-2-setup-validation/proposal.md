## Why

The Phase 1 skeleton wired the `Setup` stage to a stub — tenant pipelines load and parse correctly but no actual validation runs, meaning bad parameters (misspelled `runtimeType`, `tenantName` with uppercase, missing Dockerfile) silently reach the Build stage and produce confusing errors far from their root cause. Phase 2 replaces that stub with the real `steps/setup.yml`, implementing tool-version resolution and all parameter validation so every failure produces a clear Stage 1 error message before any build work begins.

## What Changes

- **Replace stub** `platform-templates/steps/setup.yml` with a fully implemented ADO step template
- **Tool version resolution**: read Docker/BuildKit, Syft, Cosign, Hadolint, and Azure Artifacts npm registry URL from the `platform-tool-versions` ADO variable group; expose each as a pipeline output variable for downstream steps to consume
- **tenantName / appName validation**: assert lowercase alphanumeric + hyphens only; fail Stage 1 with explicit message identifying which parameter and what was wrong
- **runtimeType allowlist validation**: assert value is one of `angular`, `react`, `springboot`, `python`, `go`; fail Stage 1 listing the invalid value and the allowed set
- **dockerfilePath existence check**: assert the file exists at `<buildContext>/<dockerfilePath>` on the agent; fail Stage 1 with the resolved path that was checked
- **Runtime template existence check**: assert `steps/runtime/<runtimeType>.yml` exists in the template repository; fail Stage 1 if absent (guards against a broken dispatch that references a non-existent stub)

## Capabilities

### New Capabilities

- `tool-version-resolution`: Reading tool version pins from the `platform-tool-versions` ADO variable group and making them available to downstream steps as named pipeline variables. Covers which variables are required, what happens when one is absent, and the naming convention for output variables.
- `parameter-validation`: The full set of validation rules applied to tenant-supplied parameters at Stage 1 — naming convention enforcement, allowlist checking, and filesystem-level existence assertions. Covers which failures are hard errors vs. warnings, what error messages look like, and that all validations run before any failure is raised (so a tenant sees all errors in one pass, not one at a time).

### Modified Capabilities

## Impact

- **Modified file:** `platform-templates/steps/setup.yml` — stub replaced with real implementation
- **No other platform-template files change** — downstream steps (`dockerfile-lint.yml`, `docker-build.yml`, runtime stubs) consume the output variables set here; their stubs remain untouched until their respective phases
- **ADO dependency:** `platform-tool-versions` variable group must be provisioned and linked to the pipeline before Phase 2 can be validated end-to-end (infrastructure prerequisite D-4, already noted in the implementation plan for Phase 3+; Phase 2 validation can use a local variable group with test values)
- **No tenant pipeline changes required** — the `Setup` stage is internal to the base template; tenants do not reference `steps/setup.yml` directly
