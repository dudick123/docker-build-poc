## ADDED Requirements

### Requirement: Absence of package.json produces a warning, not a failure
The `steps/runtime/angular.yml` template SHALL check for `package.json` in `buildContext`. If absent, the step SHALL emit `##vso[task.logissue type=warning]` and exit zero. The Build stage SHALL continue.

#### Scenario: package.json present
- **WHEN** `package.json` exists at `$BUILD_SOURCESDIRECTORY/$BUILD_CONTEXT/package.json`
- **THEN** the check passes silently and the step continues to version extraction

#### Scenario: package.json absent
- **WHEN** no `package.json` exists in `buildContext`
- **THEN** the step emits a warning: `"No package.json found in <buildContext>. Expected package.json for runtimeType: angular."` and exits zero; the Build stage continues

### Requirement: Absence of ng in package.json scripts produces a warning, not a failure
The `steps/runtime/angular.yml` template SHALL check that the string `"ng` appears in the `scripts` section of `package.json`. If absent, the step SHALL emit `##vso[task.logissue type=warning]` and continue. The Build stage SHALL NOT fail.

#### Scenario: ng referenced in scripts
- **WHEN** `package.json` contains `"build": "ng build"` or similar
- **THEN** the check passes silently

#### Scenario: ng not found in scripts
- **WHEN** `package.json` exists but contains no `"ng` in any script value
- **THEN** the step emits a warning: `"'ng' not found in package.json scripts. Confirm this is an Angular project for runtimeType: angular."` and continues

### Requirement: Version extracted from package.json and emitted as ANGULAR_VERSION
The `angular.yml` template SHALL extract the `version` field from `package.json` using `grep`/`sed` and emit it as a step output variable named `ANGULAR_VERSION` on a step named `angularRuntime`. If `package.json` is absent or contains no parseable `version` field, `ANGULAR_VERSION` SHALL be emitted as empty string.

#### Scenario: package.json with version field
- **WHEN** `package.json` contains `"version": "2.0.1"`
- **THEN** the step emits `ANGULAR_VERSION=2.0.1`

#### Scenario: package.json absent or missing version
- **WHEN** `package.json` does not exist or has no top-level `"version"` field
- **THEN** the step emits `ANGULAR_VERSION=` (empty string); Phase 8 skips the version tag; the pipeline continues

### Requirement: angular.yml performs no agent-side compilation or toolchain invocation
The `angular.yml` step template SHALL NOT invoke `npm`, `node`, `ng`, or any other Node.js or Angular CLI command. No Node.js runtime, npm, or Angular CLI SHALL be required on the pipeline agent.

#### Scenario: Agent has no Node.js installed
- **WHEN** the pipeline agent has no `node`, `npm`, or `ng` binary available
- **THEN** `angular.yml` completes successfully (file existence checks and grep only); all npm and Angular build commands run inside the Docker container
