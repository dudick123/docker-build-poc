## MODIFIED Requirements

### Requirement: Pipeline defines five named stages in dependency order
The template SHALL define exactly five stages: `Setup`, `Build`, `SignAndAttest`, `Publish`, `Notify`. Stages SHALL declare explicit `dependsOn` references: `Build` depends on `Setup`; `SignAndAttest` depends on `Build`; `Publish` depends on `SignAndAttest`; `Notify` depends on `Publish`.

The `sbom-sign-publish.yml` template call in the `SignAndAttest` stage SHALL pass the following parameters in addition to `tenantName`, `appName`, and `dryRun`: `acrHost` (from Setup stage output), `imageDigest` (from Build stage `buildImage` step output), `syftVersion` (from Setup stage output), `cosignVersion` (from Setup stage output).

The `sbom-sign-publish.yml` template call in the `Publish` stage SHALL pass: `acrHost` (from Setup stage output), `manifestDigest` (from SignAndAttest stage `signAttest` step output), `runtimeVersion` (from the Build stage runtime step output variable for the active `runtimeType`, selected at compile time via `${{ if eq(parameters.runtimeType, ...) }}`).

The `sbom-sign-publish.yml` template call in the `Notify` stage SHALL pass: `acrHost` (from Setup stage output), `manifestDigest` (from SignAndAttest stage `signAttest` step output), `imageRef` (from Publish stage `publish` step output).

#### Scenario: All stages visible in ADO pipeline graph after template loads
- **WHEN** a tenant pipeline using the template is queued
- **THEN** the ADO pipeline graph shows all five stages in left-to-right order with correct dependency arrows

#### Scenario: Build stage does not start if Setup fails
- **WHEN** the Setup stage fails (e.g., parameter validation error in a later phase)
- **THEN** all downstream stages (Build, SignAndAttest, Publish, Notify) are skipped automatically

#### Scenario: SignAndAttest stage receives all required parameters
- **WHEN** the SignAndAttest stage begins
- **THEN** `acrHost`, `imageDigest`, `syftVersion`, and `cosignVersion` are available as non-empty parameter values inside `sbom-sign-publish.yml`

#### Scenario: Publish stage receives manifestDigest and runtimeVersion
- **WHEN** the Publish stage begins after Stage 3 succeeds
- **THEN** `manifestDigest` resolves to the `sha256:...` value emitted by the `signAttest` step, and `runtimeVersion` resolves to the version string emitted by the active runtime step template in Stage 2
