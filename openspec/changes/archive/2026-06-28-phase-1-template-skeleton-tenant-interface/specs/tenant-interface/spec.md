## ADDED Requirements

### Requirement: Template exposes exactly six tenant parameters
The base template `container-build-v2.yml` SHALL declare a `parameters:` block containing exactly these six parameters: `tenantName` (string, required), `appName` (string, required), `runtimeType` (string, required), `dockerfilePath` (string, default `Dockerfile`), `buildContext` (string, default `.`), `dryRun` (boolean, default `false`). No other parameters SHALL be exposed to tenant pipelines.

#### Scenario: Tenant pipeline references template with all required parameters
- **WHEN** a tenant pipeline uses `extends:` referencing `container-build-v2.yml` and provides `tenantName`, `appName`, and `runtimeType`
- **THEN** the pipeline loads successfully in ADO with no missing-parameter errors

#### Scenario: Tenant pipeline omits optional parameters
- **WHEN** a tenant pipeline provides only `tenantName`, `appName`, and `runtimeType`, omitting `dockerfilePath`, `buildContext`, and `dryRun`
- **THEN** the template applies defaults: `dockerfilePath=Dockerfile`, `buildContext=.`, `dryRun=false`

#### Scenario: Tenant pipeline provides an undeclared parameter
- **WHEN** a tenant pipeline passes a parameter not in the six-parameter contract (e.g., `acrEndpoint`)
- **THEN** ADO rejects the pipeline at compile time with an unknown-parameter error

### Requirement: Platform controls are not exposed as parameters
All platform controls — ACR endpoint, Cosign key vault reference, tag naming convention, tool version pins — SHALL be internal to the template and NOT exposed as tenant-facing parameters.

#### Scenario: Tenant attempts to override ACR endpoint
- **WHEN** a tenant pipeline attempts to pass an `acrEndpoint` or `registry` parameter
- **THEN** ADO rejects the pipeline at compile time (undeclared parameter)

#### Scenario: Platform tool version is updated centrally
- **WHEN** the platform team updates a tool version in `platform-tool-versions` variable group
- **THEN** all tenant pipelines pick up the new version automatically without any change to their `azure-pipelines.yml`

### Requirement: `dryRun` parameter controls sign and publish stages
When `dryRun` is `true`, Stage 3 (Sign & Attest) and Stage 4 (Publish) SHALL be skipped. Stage 5 (Notify) SHALL still execute and SHALL include a dry-run indicator in its output.

#### Scenario: Pipeline runs with dryRun=true
- **WHEN** a tenant pipeline triggers with `dryRun: true`
- **THEN** Stages 3 and 4 are skipped and shown as skipped in the ADO pipeline graph; Stage 5 executes

#### Scenario: Pipeline runs with dryRun=false (default)
- **WHEN** a tenant pipeline triggers with `dryRun: false` or omits the parameter
- **THEN** all five stages are eligible to run (subject to their own success conditions)
