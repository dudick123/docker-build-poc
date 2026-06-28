## ADDED Requirements

### Requirement: Setup step reads five tool versions from platform-tool-versions variable group
The `steps/setup.yml` template SHALL read the following five variables from the `platform-tool-versions` ADO variable group: `DOCKER_BUILDKIT_VERSION`, `SYFT_VERSION`, `COSIGN_VERSION`, `HADOLINT_VERSION`, and `NPM_REGISTRY_URL`. Each variable SHALL be emitted as a step output variable using the ADO logging command `##vso[task.setvariable variable=<NAME>;isOutput=true]` from a step named `resolveTools`.

#### Scenario: All five variables present and non-empty
- **WHEN** the `platform-tool-versions` variable group contains non-empty values for all five variables and is linked to the pipeline
- **THEN** the `Setup` stage succeeds and each variable is accessible to downstream steps as `$(Setup.resolveTools.DOCKER_BUILDKIT_VERSION)`, `$(Setup.resolveTools.SYFT_VERSION)`, `$(Setup.resolveTools.COSIGN_VERSION)`, `$(Setup.resolveTools.HADOLINT_VERSION)`, and `$(Setup.resolveTools.NPM_REGISTRY_URL)`

#### Scenario: A required variable is absent or empty in the variable group
- **WHEN** one or more of the five variables is missing or empty in `platform-tool-versions`
- **THEN** the setup step fails with an error message that names the missing variable and the variable group it must be set in, and Stage 1 fails before any build work begins

### Requirement: Variable group is not managed or overridden by tenant pipelines
The `platform-tool-versions` variable group SHALL be linked to the pipeline exclusively by the base template. Tenant pipelines SHALL NOT be required or permitted to link or override this variable group directly.

#### Scenario: Tenant pipeline does not declare the variable group
- **WHEN** a tenant pipeline uses `extends:` referencing the base template without declaring `platform-tool-versions` in its own `variables:` block
- **THEN** tool versions are still resolved correctly because the base template owns the variable group linkage

#### Scenario: Tool version is updated in the variable group by the platform team
- **WHEN** the platform team updates `HADOLINT_VERSION` in `platform-tool-versions`
- **THEN** all tenant pipelines that queue after the update automatically use the new version without any change to their `azure-pipelines.yml`

### Requirement: Output variable names follow the established convention
The five output variables SHALL use the exact names `DOCKER_BUILDKIT_VERSION`, `SYFT_VERSION`, `COSIGN_VERSION`, `HADOLINT_VERSION`, and `NPM_REGISTRY_URL`. Downstream step templates (Phases 3–8) SHALL reference them using the `$(Setup.resolveTools.<NAME>)` syntax.

#### Scenario: Downstream step references a resolved tool version
- **WHEN** the `Setup` stage completes successfully and the `Build` stage begins
- **THEN** a step in `steps/dockerfile-lint.yml` can reference `$(Setup.resolveTools.HADOLINT_VERSION)` to pin the Hadolint binary version without re-reading the variable group
