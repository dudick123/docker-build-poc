## ADDED Requirements

### Requirement: Hadolint runs at the platform-pinned version before every build
The `steps/dockerfile-lint.yml` template SHALL download and execute Hadolint at the exact version specified by the `HADOLINT_VERSION` step output variable from the Setup stage (`$(stageDependencies.Setup.Setup.outputs['resolveTools.HADOLINT_VERSION'])`). The binary SHALL be downloaded from the official Hadolint GitHub release and verified executable before use.

#### Scenario: Hadolint binary downloaded and executed at pinned version
- **WHEN** the Build stage begins with `HADOLINT_VERSION` set to a valid version string (e.g., `v2.12.0`)
- **THEN** Hadolint at that exact version is downloaded, made executable, and run against the Dockerfile before the `docker build` step begins

#### Scenario: HADOLINT_VERSION variable is consumed from Setup stage output
- **WHEN** Phase 2 `resolveTools` step has emitted `HADOLINT_VERSION` as a step output variable
- **THEN** `dockerfile-lint.yml` uses that value to pin the download URL and does not hardcode a version

### Requirement: ERROR-level Hadolint findings fail the build; WARNING-level findings do not
Hadolint SHALL be invoked with `--failure-threshold error`. The lint step SHALL exit non-zero (failing the Build stage) if and only if one or more ERROR-level findings are present. WARNING-level findings SHALL be printed to the build log and SHALL NOT cause a non-zero exit code.

#### Scenario: Dockerfile has an ERROR-level finding
- **WHEN** the tenant Dockerfile triggers a Hadolint rule at ERROR severity (e.g., `DL3020 — use COPY instead of ADD`)
- **THEN** the lint step exits non-zero, the Build stage fails, and the finding is visible in the ADO build log

#### Scenario: Dockerfile has only WARNING-level findings
- **WHEN** the tenant Dockerfile triggers Hadolint rules only at WARNING severity
- **THEN** the lint step exits zero, the Build stage continues, and the warnings are printed to the build log

#### Scenario: Dockerfile has no findings
- **WHEN** the tenant Dockerfile passes all Hadolint checks
- **THEN** the lint step exits zero with a clean log line confirming no findings

### Requirement: Tenant-provided .hadolint.yaml is used when present in build context
If a file named `.hadolint.yaml` exists at the root of `buildContext`, the lint step SHALL pass it to Hadolint via `--config <buildContext>/.hadolint.yaml`. If absent, Hadolint SHALL run with its built-in defaults. Absence of the file SHALL NOT be treated as an error.

#### Scenario: .hadolint.yaml present in build context
- **WHEN** the tenant repository contains `<buildContext>/.hadolint.yaml`
- **THEN** Hadolint uses that file as its configuration, allowing tenants to ignore or override specific rules

#### Scenario: .hadolint.yaml absent from build context
- **WHEN** no `.hadolint.yaml` exists in the build context
- **THEN** Hadolint runs with built-in defaults; the lint step does not fail or warn about the missing config file
