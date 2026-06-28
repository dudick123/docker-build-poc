## ADDED Requirements

### Requirement: runtimeType dispatches to exactly one runtime step template
The base template SHALL use ADO compile-time `${{ if }}` expressions to dispatch the `runtimeType` parameter to the matching `steps/runtime/<runtimeType>.yml` template. Exactly one branch SHALL be included in the expanded pipeline for any given `runtimeType` value.

#### Scenario: Go runtime dispatches to go.yml
- **WHEN** a tenant pipeline sets `runtimeType: go`
- **THEN** the expanded pipeline includes `steps/runtime/go.yml` and no other runtime step template

#### Scenario: Python runtime dispatches to python.yml
- **WHEN** a tenant pipeline sets `runtimeType: python`
- **THEN** the expanded pipeline includes `steps/runtime/python.yml` and no other runtime step template

#### Scenario: SpringBoot runtime dispatches to springboot.yml
- **WHEN** a tenant pipeline sets `runtimeType: springboot`
- **THEN** the expanded pipeline includes `steps/runtime/springboot.yml` and no other runtime step template

#### Scenario: Angular runtime dispatches to angular.yml
- **WHEN** a tenant pipeline sets `runtimeType: angular`
- **THEN** the expanded pipeline includes `steps/runtime/angular.yml` and no other runtime step template

#### Scenario: React runtime dispatches to react.yml
- **WHEN** a tenant pipeline sets `runtimeType: react`
- **THEN** the expanded pipeline includes `steps/runtime/react.yml` and no other runtime step template

### Requirement: All five runtime stub files exist and are valid ADO templates
The files `steps/runtime/go.yml`, `steps/runtime/python.yml`, `steps/runtime/springboot.yml`, `steps/runtime/angular.yml`, and `steps/runtime/react.yml` SHALL exist in the template repository at Phase 1. Each SHALL be a valid ADO step template stub. They are replaced by real implementations in Phases 4–6.

#### Scenario: Each runtime stub file loads without ADO parse errors
- **WHEN** a tenant pipeline is queued with any valid `runtimeType` value
- **THEN** ADO expands the template without a missing-file or YAML parse error for the runtime step template

#### Scenario: Runtime stub emits identifiable log output
- **WHEN** a runtime stub step executes
- **THEN** the build log contains a message identifying which runtime stub ran (e.g., `STUB: go.yml runtime not yet implemented`)

### Requirement: Invalid runtimeType value is not dispatched to any stub
If `runtimeType` does not match any of the five valid values, no runtime template SHALL be included in the expanded pipeline. Validation of the `runtimeType` allowlist is owned by `steps/setup.yml` (Phase 2) and is out of scope for the dispatch block itself; the dispatch block SHALL NOT silently fall through to a default template.

#### Scenario: Unknown runtimeType produces no runtime dispatch branch
- **WHEN** a tenant pipeline sets `runtimeType: dotnet` (not in the allowlist)
- **THEN** the compile-time dispatch block includes no runtime step template branch; parameter validation in Phase 2 will produce the user-facing error
