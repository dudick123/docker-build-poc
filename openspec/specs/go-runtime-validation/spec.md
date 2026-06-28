## ADDED Requirements

### Requirement: go.mod must exist in buildContext or Stage 1 fails
The `steps/runtime/go.yml` template SHALL check for the existence of `go.mod` at `<buildContext>/go.mod`. If the file is not found, the step SHALL exit non-zero with an error message: `"go.mod not found at <resolved-path>. The build context root must contain go.mod for runtimeType: go."` The pipeline SHALL NOT proceed to the Docker build.

#### Scenario: go.mod present at buildContext root
- **WHEN** `go.mod` exists at `$BUILD_SOURCESDIRECTORY/$BUILD_CONTEXT/go.mod`
- **THEN** the `go.yml` step exits zero and the pipeline continues to the Docker build

#### Scenario: go.mod absent from buildContext
- **WHEN** no `go.mod` file exists at the resolved path
- **THEN** the `go.yml` step exits non-zero with an error message identifying the full path that was checked; the Build stage fails and no Docker build runs

#### Scenario: go.mod exists in a subdirectory but not at buildContext root
- **WHEN** `go.mod` is present at `services/api/go.mod` but `buildContext` is set to `.` (repo root)
- **THEN** the check fails because `./go.mod` does not exist; the error message shows the checked path `$BUILD_SOURCESDIRECTORY/./go.mod`

### Requirement: VERSION file read is optional and emits GO_VERSION output variable
If `<buildContext>/VERSION` exists, the `go.yml` template SHALL read its first non-empty line, strip leading and trailing whitespace, and emit the result as a step output variable named `GO_VERSION` on a step named `goRuntime`. If the `VERSION` file does not exist, `GO_VERSION` SHALL be emitted as an empty string. An empty `GO_VERSION` instructs Phase 8 to skip the version tag for this build.

#### Scenario: VERSION file present with a valid version string
- **WHEN** `<buildContext>/VERSION` contains `1.2.3` on the first line
- **THEN** the step emits `GO_VERSION=1.2.3` as an output variable; Phase 8 can construct a version tag from it

#### Scenario: VERSION file absent
- **WHEN** no `VERSION` file exists in `buildContext`
- **THEN** the step emits `GO_VERSION=` (empty string); no version tag is applied in Phase 8; the pipeline continues without error

#### Scenario: VERSION file contains leading/trailing whitespace
- **WHEN** `VERSION` contains `  v1.4.0  ` (with surrounding spaces)
- **THEN** the emitted `GO_VERSION` is `v1.4.0` (trimmed)

### Requirement: go.yml performs no agent-side compilation or toolchain invocation
The `go.yml` step template SHALL NOT invoke `go build`, `go test`, `go mod download`, or any other Go toolchain command. No Go runtime or SDK SHALL be required on the pipeline agent.

#### Scenario: Agent has no Go toolchain installed
- **WHEN** the pipeline agent has no `go` binary available
- **THEN** `go.yml` completes successfully (it only checks file existence and reads a text file); the Docker build stage handles all Go compilation inside the container
