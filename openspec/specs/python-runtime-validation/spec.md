## ADDED Requirements

### Requirement: Absence of recognized dependency files produces a warning, not a failure
The `steps/runtime/python.yml` template SHALL check for the presence of at least one of `requirements.txt`, `pyproject.toml`, or `poetry.lock` in `buildContext`. If none is found, the step SHALL print a warning to the build log using `##vso[task.logissue type=warning]` and SHALL exit zero. The Build stage SHALL continue.

#### Scenario: At least one dependency file is present
- **WHEN** `pyproject.toml` (or `requirements.txt` or `poetry.lock`) exists in `buildContext`
- **THEN** the dependency file check passes silently and the pipeline continues

#### Scenario: No recognized dependency files found
- **WHEN** none of `requirements.txt`, `pyproject.toml`, or `poetry.lock` exists in `buildContext`
- **THEN** the step emits a warning: `"No recognized Python dependency file found in <buildContext>. Expected one of: requirements.txt, pyproject.toml, poetry.lock"` and exits zero; the Build stage continues to the Docker build

### Requirement: Version extracted from pyproject.toml or setup.cfg and emitted as PYTHON_VERSION
The `python.yml` template SHALL attempt version extraction in this order:
1. If `pyproject.toml` exists: search for `version = "..."` under `[project]` or `[tool.poetry]` sections using pattern matching; extract the quoted value
2. Else if `setup.cfg` exists: search for `version = ...` under `[metadata]`; extract the value
3. If neither file yields a version string: emit `PYTHON_VERSION` as empty string

The extracted version SHALL be emitted as a step output variable named `PYTHON_VERSION` on a step named `pythonRuntime`. An empty `PYTHON_VERSION` instructs Phase 8 to skip the version tag.

#### Scenario: pyproject.toml with static version under [project]
- **WHEN** `pyproject.toml` contains `[project]` section with `version = "2.1.0"`
- **THEN** the step emits `PYTHON_VERSION=2.1.0`

#### Scenario: pyproject.toml with static version under [tool.poetry]
- **WHEN** `pyproject.toml` contains `[tool.poetry]` section with `version = "0.5.3"`
- **THEN** the step emits `PYTHON_VERSION=0.5.3`

#### Scenario: pyproject.toml with dynamic versioning
- **WHEN** `pyproject.toml` contains `version = {attr = "mypackage.__version__"}` (not a quoted string literal)
- **THEN** the regex does not match; the step falls through to `setup.cfg` check or emits `PYTHON_VERSION=` (empty); no error is raised

#### Scenario: setup.cfg with version under [metadata]
- **WHEN** `pyproject.toml` is absent or yields no version, and `setup.cfg` contains `[metadata]` with `version = 1.0.0`
- **THEN** the step emits `PYTHON_VERSION=1.0.0`

#### Scenario: No version extractable from either file
- **WHEN** neither `pyproject.toml` nor `setup.cfg` yields a parseable static version
- **THEN** the step emits `PYTHON_VERSION=` (empty string); Phase 8 skips the version tag; the pipeline continues without error

### Requirement: python.yml performs no agent-side compilation or toolchain invocation
The `python.yml` step template SHALL NOT invoke `python`, `pip`, `poetry`, `uv`, or any other Python toolchain command. No Python runtime or virtual environment SHALL be required on the pipeline agent.

#### Scenario: Agent has no Python interpreter installed
- **WHEN** the pipeline agent has no `python` or `python3` binary available
- **THEN** `python.yml` completes successfully (it only checks file existence and reads text files with grep/sed); all Python execution happens inside the Docker container
