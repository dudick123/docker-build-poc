## ADDED Requirements

### Requirement: Absence of package.json produces a warning, not a failure
The `steps/runtime/react.yml` template SHALL check for `package.json` in `buildContext`. If absent, the step SHALL emit `##vso[task.logissue type=warning]` and exit zero. The Build stage SHALL continue.

#### Scenario: package.json present
- **WHEN** `package.json` exists at `$BUILD_SOURCESDIRECTORY/$BUILD_CONTEXT/package.json`
- **THEN** the check passes silently and the step continues to SSR detection

#### Scenario: package.json absent
- **WHEN** no `package.json` exists in `buildContext`
- **THEN** the step emits a warning: `"No package.json found in <buildContext>. Expected package.json for runtimeType: react."` and exits zero; the Build stage continues

### Requirement: Next.js SSR apps without static export config produce a warning
The `steps/runtime/react.yml` template SHALL detect Next.js by checking for `"next"` in `package.json` dependencies or devDependencies. If detected, the template SHALL check `next.config.js` and `next.config.mjs` in `buildContext` for the text `output.*export`. If neither config file contains that pattern, the step SHALL emit `##vso[task.logissue type=warning]` indicating the app appears to be SSR and requires a Node.js runtime in the final image, not nginx. The Build stage SHALL NOT fail.

#### Scenario: Next.js app with output: export — static SPA
- **WHEN** `package.json` contains `"next"` in dependencies AND `next.config.js` contains `output: 'export'`
- **THEN** no warning is emitted; the step continues

#### Scenario: Next.js app without output: export — likely SSR
- **WHEN** `package.json` contains `"next"` in dependencies AND neither `next.config.js` nor `next.config.mjs` contains `output.*export`
- **THEN** the step emits a warning: `"Next.js app detected without 'output: export' in next.config.js. SSR apps require a Node.js runtime in the final image, not nginx. See platform reference Dockerfile at docs/reference-dockerfiles/react-ssr.Dockerfile."` and exits zero

#### Scenario: Non-Next.js React app
- **WHEN** `package.json` does not contain `"next"` in dependencies
- **THEN** no SSR check is performed; the step continues without warning

### Requirement: Version extracted from package.json and emitted as REACT_VERSION
The `react.yml` template SHALL extract the `version` field from `package.json` using `grep`/`sed` and emit it as a step output variable named `REACT_VERSION` on a step named `reactRuntime`. If `package.json` is absent or contains no parseable `version` field, `REACT_VERSION` SHALL be emitted as empty string.

#### Scenario: package.json with version field
- **WHEN** `package.json` contains `"version": "1.5.3"`
- **THEN** the step emits `REACT_VERSION=1.5.3`

#### Scenario: package.json absent or missing version
- **WHEN** `package.json` does not exist or has no top-level `"version"` field
- **THEN** the step emits `REACT_VERSION=` (empty string); Phase 8 skips the version tag; the pipeline continues

### Requirement: react.yml performs no agent-side compilation or toolchain invocation
The `react.yml` step template SHALL NOT invoke `npm`, `node`, `react-scripts`, `next`, `vite`, or any other Node.js or React toolchain command. No Node.js runtime or npm SHALL be required on the pipeline agent.

#### Scenario: Agent has no Node.js installed
- **WHEN** the pipeline agent has no `node` or `npm` binary available
- **THEN** `react.yml` completes successfully (file existence checks and grep only); all npm and React build commands run inside the Docker container
