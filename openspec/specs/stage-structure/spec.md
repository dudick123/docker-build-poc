## ADDED Requirements

### Requirement: Pipeline defines five named stages in dependency order
The template SHALL define exactly five stages: `Setup`, `Build`, `SignAndAttest`, `Publish`, `Notify`. Stages SHALL declare explicit `dependsOn` references: `Build` depends on `Setup`; `SignAndAttest` depends on `Build`; `Publish` depends on `SignAndAttest`; `Notify` depends on `Publish`.

The `sbom-sign-publish.yml` template call in the `SignAndAttest` stage SHALL pass the following parameters in addition to `tenantName`, `appName`, and `dryRun`: `acrHost` (from Setup stage output), `imageDigest` (from Build stage `buildImage` step output), `syftVersion` (from Setup stage output), `cosignVersion` (from Setup stage output).

#### Scenario: All stages visible in ADO pipeline graph after template loads
- **WHEN** a tenant pipeline using the template is queued
- **THEN** the ADO pipeline graph shows all five stages in left-to-right order with correct dependency arrows

#### Scenario: Build stage does not start if Setup fails
- **WHEN** the Setup stage fails (e.g., parameter validation error in a later phase)
- **THEN** all downstream stages (Build, SignAndAttest, Publish, Notify) are skipped automatically

#### Scenario: SignAndAttest stage receives all required parameters
- **WHEN** the SignAndAttest stage begins
- **THEN** `acrHost`, `imageDigest`, `syftVersion`, and `cosignVersion` are available as non-empty parameter values inside `sbom-sign-publish.yml`

### Requirement: Stage 3 and Stage 4 skip when dryRun is true
`SignAndAttest` and `Publish` stages SHALL include a `condition` that evaluates to false when `dryRun` is `true`, causing ADO to skip these stages without failing the pipeline.

#### Scenario: dryRun=true shows skipped stages in ADO UI
- **WHEN** the pipeline runs with `dryRun: true`
- **THEN** the ADO pipeline graph shows `SignAndAttest` and `Publish` with a skipped (grey) status icon, and the pipeline completes successfully

#### Scenario: dryRun=false runs all stages
- **WHEN** the pipeline runs with `dryRun: false`
- **THEN** no stages are skipped due to the `dryRun` condition; all stages proceed normally

### Requirement: Stage 5 (Notify) runs after Publish regardless of dryRun
The `Notify` stage SHALL execute whether or not `dryRun` is `true`. Its condition SHALL be `succeededOrFailed()` on the `Publish` stage (which will be in a skipped state on dry runs, not a failed state).

#### Scenario: Notify stage executes on dry run
- **WHEN** the pipeline completes a dry run (SignAndAttest and Publish skipped)
- **THEN** the Notify stage runs and produces dry-run output

#### Scenario: Notify stage executes on full run
- **WHEN** the pipeline completes a full run with all stages succeeded
- **THEN** the Notify stage runs and produces full-run output

### Requirement: Stub step templates load without ADO errors
Each stage SHALL reference step template files (`steps/setup.yml`, `steps/dockerfile-lint.yml`, `steps/docker-build.yml`, `steps/sbom-sign-publish.yml`, `steps/runtime/<runtimeType>.yml`) that exist in the template repository. At Phase 1, these files SHALL be valid minimal stubs (a `steps:` block with a no-op echo step) that prevent compile-time missing-file errors.

#### Scenario: Template loads in ADO with all stubs present
- **WHEN** the base template and all stub files are committed to the template repository
- **THEN** a tenant pipeline using `extends:` loads without any template-not-found or YAML parse errors

#### Scenario: Stub step emits recognizable output
- **WHEN** a stub step executes (e.g., in a dry-run test)
- **THEN** the build log contains a message clearly identifying the step as a stub (e.g., `STUB: setup.yml not yet implemented`)
