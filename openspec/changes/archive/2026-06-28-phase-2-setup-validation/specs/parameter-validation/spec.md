## ADDED Requirements

### Requirement: All parameter validations run before any failure is raised
The `steps/setup.yml` template SHALL execute all validation checks in a single pass. Errors SHALL be collected into a list and all errors SHALL be printed before the step exits with a non-zero code. A single pipeline run SHALL surface all parameter problems simultaneously rather than one at a time.

#### Scenario: Multiple parameters are invalid at the same time
- **WHEN** a tenant pipeline is queued with both an invalid `tenantName` (e.g., `MyTenant`) and an invalid `runtimeType` (e.g., `dotnet`)
- **THEN** Stage 1 fails with both errors reported in the same step output, not sequentially across two pipeline runs

#### Scenario: All parameters are valid
- **WHEN** a tenant pipeline provides `tenantName: acme-corp`, `appName: payments-api`, `runtimeType: go`, `dockerfilePath: Dockerfile`, `buildContext: .`
- **THEN** the validation step exits cleanly with no error output and Stage 1 proceeds

### Requirement: tenantName and appName must match lowercase alphanumeric-plus-hyphens pattern
`tenantName` and `appName` SHALL each match the pattern `^[a-z0-9][a-z0-9-]*[a-z0-9]$` (lowercase letters and digits only, hyphens allowed in interior positions, minimum two characters). A value that does not match SHALL cause the validation step to record an error naming the parameter, the invalid value, and the required pattern.

#### Scenario: tenantName contains uppercase letters
- **WHEN** a tenant pipeline sets `tenantName: MyTenant`
- **THEN** the validation step records an error: `tenantName 'MyTenant' does not match required pattern ^[a-z0-9][a-z0-9-]*[a-z0-9]$ (lowercase alphanumeric and hyphens only, minimum 2 characters)`

#### Scenario: appName contains an underscore
- **WHEN** a tenant pipeline sets `appName: payments_api`
- **THEN** the validation step records an error naming `appName` and the invalid value

#### Scenario: tenantName starts with a hyphen
- **WHEN** a tenant pipeline sets `tenantName: -acme`
- **THEN** the validation step records an error; a leading hyphen violates the pattern

#### Scenario: tenantName is a valid single-segment kebab name
- **WHEN** a tenant pipeline sets `tenantName: acme-corp`
- **THEN** no error is recorded for `tenantName`

### Requirement: runtimeType must be one of the five supported values
`runtimeType` SHALL be validated against the allowlist `[angular, react, springboot, python, go]`. Any other value SHALL cause the validation step to record an error naming the invalid value and listing all allowed values.

#### Scenario: runtimeType is an unsupported value
- **WHEN** a tenant pipeline sets `runtimeType: dotnet`
- **THEN** the validation step records an error: `runtimeType 'dotnet' is not supported. Allowed values: angular, react, springboot, python, go`

#### Scenario: runtimeType is a supported value
- **WHEN** a tenant pipeline sets `runtimeType: springboot`
- **THEN** no error is recorded for `runtimeType`

#### Scenario: runtimeType uses incorrect casing
- **WHEN** a tenant pipeline sets `runtimeType: SpringBoot`
- **THEN** the validation step records an error (exact case match required; `SpringBoot` is not in the allowlist)

### Requirement: dockerfilePath must exist relative to buildContext on the agent workspace
The resolved path `$(Build.SourcesDirectory)/<buildContext>/<dockerfilePath>` SHALL be checked for existence using a filesystem test. If the file does not exist, the validation step SHALL record an error showing the fully resolved path that was checked.

#### Scenario: Dockerfile exists at the resolved path
- **WHEN** `buildContext` is `.` and `dockerfilePath` is `Dockerfile` and the file `$(Build.SourcesDirectory)/./Dockerfile` exists
- **THEN** no error is recorded for `dockerfilePath`

#### Scenario: Dockerfile does not exist at the resolved path
- **WHEN** `buildContext` is `services/api` and `dockerfilePath` is `Dockerfile.prod` and no such file exists on the agent
- **THEN** the validation step records an error: `dockerfilePath not found: $(Build.SourcesDirectory)/services/api/Dockerfile.prod`

#### Scenario: buildContext is a subdirectory and Dockerfile is in a relative sub-path
- **WHEN** `buildContext` is `services/api` and `dockerfilePath` is `docker/Dockerfile` and the file exists at `$(Build.SourcesDirectory)/services/api/docker/Dockerfile`
- **THEN** no error is recorded

### Requirement: Runtime step template file must exist in the template repository
The file `steps/runtime/<runtimeType>.yml` SHALL be confirmed to exist in the platform-templates repository checkout on the agent at `$(Agent.BuildDirectory)/s/platform-templates/steps/runtime/<runtimeType>.yml`. If it does not exist, the validation step SHALL record an error naming the missing file path and directing platform engineers to create it.

#### Scenario: Runtime template exists for the given runtimeType
- **WHEN** `runtimeType` is `go` and `$(Agent.BuildDirectory)/s/platform-templates/steps/runtime/go.yml` exists
- **THEN** no error is recorded for the runtime template check

#### Scenario: Runtime template is missing for a value that passed the allowlist check
- **WHEN** `runtimeType` is `python` and the `python.yml` template file has been accidentally deleted from the repository
- **THEN** the validation step records an error: `Runtime template not found: steps/runtime/python.yml — contact platform engineering`
