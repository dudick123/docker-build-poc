## Context

`steps/runtime/go.yml` and `steps/runtime/python.yml` are Phase 1 stubs executing in the Build stage's single job, after `dockerfile-lint.yml` and `docker-build.yml` have already run. Both receive `buildContext` as a parameter from the base template. Their role in Pattern A is narrow: validate that the project structure makes sense for the runtime, extract a version string if one is available, then exit. All compilation happens inside the multi-stage Dockerfile — nothing runs on the agent.

Both templates emit a named step output variable (`GO_VERSION` / `PYTHON_VERSION`) that Phase 8 will consume for version tag construction. An empty string means "skip version tag for this build".

Stakeholders: platform engineering (owns templates), Go and Python tenant teams (receive error messages and version tag behavior).

## Goals / Non-Goals

**Goals:**
- `go.yml`: assert `go.mod` existence (hard fail if absent), read `VERSION` file if present, emit `GO_VERSION`
- `python.yml`: warn if no recognized dependency file is found (soft), extract version from `pyproject.toml` or `setup.cfg` if present, emit `PYTHON_VERSION`
- Both: pass-through — exit zero and let the Dockerfile build proceed
- Both: no agent-side compilation, no language toolchain required on agent

**Non-Goals:**
- Validating the content of `go.mod` or `pyproject.toml` beyond version extraction
- Running `go build`, `go test`, `pip install`, or any other language tool on the agent
- Enforcing specific Go module or Python packaging conventions beyond what is stated
- Configuring the BuildKit build arguments for the runtime (owned by `docker-build.yml`)

## Decisions

### Decision 1: go.mod absence is a hard Stage 1 failure, not a warning

If `go.mod` is not found at `<buildContext>/go.mod`, the `go.yml` step fails immediately with a clear error message: `"go.mod not found at <resolved-path>. The build context root must contain go.mod for runtimeType: go."` The pipeline does not proceed to the Docker build.

**Rationale:** A Go image built without `go.mod` in the context will always fail during `docker build` anyway (the Go compiler requires a module definition). Failing at Stage 1 before Hadolint and BuildKit run saves time and produces a clearer error. Alternative (warn only) was rejected — there is no valid Pattern A Go build without `go.mod`.

### Decision 2: VERSION file is optional; absence silently skips the version tag

If `<buildContext>/VERSION` exists, its first non-empty line is trimmed and emitted as `GO_VERSION`. If absent, `GO_VERSION` is set to empty string. Phase 8 interprets an empty `GO_VERSION` as "skip version tag" rather than an error.

**Rationale:** Per the implementation plan architectural decision: "Go versioning — `VERSION` file in repo root; if absent, version tag is skipped." Making absence an error would force all Go tenants to maintain a `VERSION` file even when they don't need version tagging, which is unnecessarily prescriptive.

### Decision 3: Python dependency file check is advisory only

If none of `requirements.txt`, `pyproject.toml`, or `poetry.lock` is found in `buildContext`, the `python.yml` step prints a warning to the build log but exits zero. The pipeline continues.

**Rationale:** A Python Dockerfile may legitimately have no top-level dependency file (e.g., dependencies are declared inside a sub-directory, or only in the Dockerfile itself). Unlike `go.mod`, the absence of these files does not guarantee the Docker build will fail. Warning gives tenants visibility without blocking valid builds.

### Decision 4: Python version extracted via grep/sed, no Python interpreter required on agent

Version extraction from `pyproject.toml` uses `grep` and `sed` to find `version = "..."` under `[project]` or `[tool.poetry]`. Version extraction from `setup.cfg` uses `grep` to find `version = ...` under `[metadata]`. No `python` or `toml` parsing library is invoked on the agent.

**Rationale:** Pattern A requires no language toolchain on the agent. A regex-based extraction is sufficient for the well-structured version fields in these files. Edge cases (e.g., dynamic versioning with `version = {attr = "..."}`) yield an empty string and skip the version tag — which is the correct fallback behavior rather than an error.

### Decision 5: Both templates pass parameter values through env: block, not inline template expressions

Consistent with the Phase 2 security pattern, all `${{ parameters.xxx }}` values are mapped to environment variables in the `env:` block. Bash scripts reference only `$ENV_VAR_NAME`.

**Rationale:** Prevents template injection if a tenant supplies shell metacharacters in `buildContext` or `tenantName`. Establishes a consistent pattern across all step templates.

## Risks / Trade-offs

- **`VERSION` file first line only** — If a tenant's `VERSION` file has a blank first line or leading whitespace, trimming may produce an unexpected value. Mitigation: document that `VERSION` must contain only the version string on the first line; the step trims whitespace but does not validate the format.
- **`pyproject.toml` dynamic versioning** — Projects using `version = {attr = "mypackage.__version__"}` will not have a statically extractable version. The step emits empty string and version tag is skipped silently. Mitigation: acceptable per the design; tenants who need version tagging should use a static version string.
- **grep-based extraction fragility** — A `pyproject.toml` with `version` appearing in a comment or string value before the actual field could produce a wrong extraction. Mitigation: the extraction pattern anchors to `^version\s*=` which is unlikely to appear in comment context; the risk is low for standard packaging files.

## Open Questions

None. All decisions align with the implementation plan architectural decisions for Phase 4.
