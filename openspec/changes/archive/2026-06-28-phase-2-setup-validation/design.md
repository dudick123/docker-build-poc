## Context

`steps/setup.yml` is currently a stub from Phase 1. It is the first step template called in the `Setup` stage, before any build work begins. Its two responsibilities are: (1) surface tool versions from the `platform-tool-versions` variable group as named pipeline variables so downstream step templates can reference them without repeating variable-group lookups, and (2) validate all six tenant-supplied parameters and fail Stage 1 with actionable error messages before any build work is attempted.

The `platform-tool-versions` variable group is a platform-managed ADO variable group that pins exact versions for Docker/BuildKit, Syft, Cosign, Hadolint, and the Azure Artifacts npm registry URL. It is linked to the pipeline at the template level — tenants do not manage it.

Stakeholders: platform engineering (owns the template), tenant teams (see the error messages).

## Goals / Non-Goals

**Goals:**
- Read five variables from `platform-tool-versions` and surface them as step output variables consumable by downstream templates
- Fail Stage 1 immediately and clearly when any parameter is invalid
- Collect all validation errors in a single pass so tenants see everything wrong at once, not one error per run
- Implement all validation logic as bash steps — no marketplace task dependencies

**Non-Goals:**
- Validating the content of the Dockerfile (Phase 3's Hadolint step owns that)
- Verifying tool binaries are installed on the agent (Phase 3 will assert this at first use)
- Reading version tags or project metadata files (runtime-specific phases own that)
- Validating `dryRun` semantics (already enforced by stage-level conditions in the base template)

## Decisions

### Decision 1: Validation collects all errors before failing

All validation checks run regardless of whether earlier checks pass. Errors are collected into an array; if the array is non-empty after all checks, the step prints every error and then calls `exit 1`.

**Rationale:** Fail-fast (stop at first error) forces tenants to fix issues one at a time across multiple pipeline runs. Collect-all is a strictly better DX for a setup step where all checks are independent. The downside (slightly more code) is trivial.

### Decision 2: Tool versions surfaced via ADO step output variables

The `Setup` step reads each tool version variable from the variable group and re-emits it using `##vso[task.setvariable variable=<NAME>;isOutput=true]`. Downstream step templates reference them as `$(Setup.resolveTools.<VARNAME>)` (using the job-level output variable syntax).

The five output variable names are: `DOCKER_BUILDKIT_VERSION`, `SYFT_VERSION`, `COSIGN_VERSION`, `HADOLINT_VERSION`, `NPM_REGISTRY_URL`.

**Rationale:** Step output variables are the standard ADO mechanism for passing values between steps within a job. Using a dedicated `resolveTools` step name (rather than a stage variable) scopes the output cleanly and avoids collision with other variables. Alternative (setting stage-level variables at runtime) was rejected because ADO does not support runtime-set stage variables that propagate to other stages in all pipeline types.

### Decision 3: Naming convention enforced by regex, not character-by-character check

`tenantName` and `appName` are validated against the pattern `^[a-z0-9][a-z0-9-]*[a-z0-9]$` using bash's `=~` operator. Minimum length is 2 characters (one leading + one trailing non-hyphen char).

**Rationale:** A single regex check is readable and unambiguous. The pattern matches the ADO resource naming convention already used for the per-tenant service connection (`<tenantName>/*`) and ACR repository path (`<tenantName>/<appName>`). Alternative (allow single-character names) was rejected because single-char names are too collision-prone in the ACR namespace.

### Decision 4: `dockerfilePath` validated as a filesystem path relative to `buildContext`

The check is: `test -f "$(buildContext)/$(dockerfilePath)"`. The `buildContext` parameter is treated as a path relative to `$(Build.SourcesDirectory)` (the agent workspace root). If the file does not exist, the error message includes the fully resolved absolute path so tenants know exactly what path was checked.

**Rationale:** Resolving the full path makes the error actionable — tenants don't need to mentally reconstruct what the agent sees. Alternative (check only `dockerfilePath` without `buildContext`) was rejected because a Dockerfile at the default path `Dockerfile` in a non-default build context would false-negative.

### Decision 5: Runtime template existence check is a defensive assertion against invariant violation

Since the `runtimeType` allowlist validation (Decision 3) already restricts values to the five runtimes that have corresponding stub files (Phase 1), the runtime template check `test -f "$(Agent.BuildDirectory)/s/<template-resource-alias>/steps/runtime/<runtimeType>.yml"` is belt-and-suspenders. It fires only if the allowlist and stubs fall out of sync — for example, a new runtime added to the allowlist before its template file is created.

**Rationale:** The check is cheap (one file test) and produces an unambiguous error message directing platform engineers to the missing file. Omitting it would allow a broken state to surface as a confusing ADO template-expansion error rather than a clear Stage 1 failure. The template resource alias for the platform-templates repository is assumed to be `platform-templates` (aligned with Phase 1 repo registration convention).

### Decision 6: Missing variable group variable is a hard failure

If any of the five expected variables is absent from `platform-tool-versions` (empty string after variable group lookup), the setup step fails with a message naming the missing variable and the variable group it should be in.

**Rationale:** A missing tool version variable would cause silent failures downstream (Hadolint not found, Cosign wrong version, etc.). Failing loudly at setup is strictly better than failing silently at use.

## Risks / Trade-offs

- **Variable group not linked → pipeline queue failure before Stage 1 runs** — If `platform-tool-versions` is not linked to the pipeline, ADO will refuse to queue the run at all (variable group resolution happens at queue time). The Stage 1 validation code never runs. Mitigation: document the variable group linkage requirement in the base template YAML comment and in platform onboarding docs.
- **Template resource alias assumption** — Decision 5 assumes the platform-templates repository is registered as a resource with alias `platform-templates`. If a tenant registers it under a different alias, the runtime template check will false-positive. Mitigation: the base template will document the required resource alias; the runtime check error message will name the expected alias.
- **Regex minimum-length edge case** — The pattern `^[a-z0-9][a-z0-9-]*[a-z0-9]$` requires at least 2 characters. A single-character `tenantName` (e.g., `a`) would fail. This is intentional but should be documented in the parameter description.

## Open Questions

None. All Phase 2 decisions are resolved. The template resource alias (`platform-templates`) is a convention established by the platform team and is treated as fixed for this phase.
