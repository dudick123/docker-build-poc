## Why

Phase 3 delivers a working build core, but the Go and Python runtime step templates are still Phase 1 stubs — any tenant pipeline using `runtimeType: go` or `runtimeType: python` reaches the Build stage and executes a no-op. Phase 4 replaces those stubs with validated pass-throughs: each template asserts that the build context contains the expected project structure, reads version metadata for downstream tag detection, and then exits cleanly so the full build runs inside the Dockerfile (Pattern A — no agent-side compilation).

## What Changes

- **Replace stub** `platform-templates/steps/runtime/go.yml`:
  - Hard assertion: `go.mod` must exist in `buildContext`; fail Stage 1 if absent with a message directing the tenant to place `go.mod` at the build context root
  - Optional version read: if `VERSION` file exists in `buildContext`, read its contents and emit as `GO_VERSION` step output variable for use in Phase 8 tag logic; if absent, emit empty string and skip version tag (per implementation plan decision)
  - No-op beyond the above — full Go build runs inside the Dockerfile

- **Replace stub** `platform-templates/steps/runtime/python.yml`:
  - Advisory check: warn if none of `requirements.txt`, `pyproject.toml`, or `poetry.lock` is found in `buildContext` (does not block the build)
  - Optional version read: if `pyproject.toml` exists, extract the `version` field from the `[project]` or `[tool.poetry]` table and emit as `PYTHON_VERSION` step output variable; else if `setup.cfg` exists, extract `version` from `[metadata]`; if neither yields a version, emit empty string
  - No-op beyond the above — full Python build runs inside the Dockerfile

## Capabilities

### New Capabilities

- `go-runtime-validation`: The assertions and metadata reads that `go.yml` performs before the BuildKit build — `go.mod` existence (hard fail), `VERSION` file read (optional), and the `GO_VERSION` output variable convention.
- `python-runtime-validation`: The advisory checks and metadata reads that `python.yml` performs before the BuildKit build — dependency file presence warning (soft), version extraction from `pyproject.toml` or `setup.cfg`, and the `PYTHON_VERSION` output variable convention.

### Modified Capabilities

## Impact

- **Modified files:** `platform-templates/steps/runtime/go.yml`, `platform-templates/steps/runtime/python.yml`
- **No base template changes** — `container-build-v2.yml` already dispatches to both files and passes `buildContext`; no wiring changes needed
- **Produces for Phase 8:** `GO_VERSION` and `PYTHON_VERSION` step output variables — Phase 8 publish logic uses these to construct version tags; empty string means skip version tag
- **No agent toolchain required** — both runtimes are Pattern A; the agent only needs Docker (already present from Phase 3)
- **Infrastructure:** no new dependencies; `platform-tool-versions` variable group requires no additions for this phase
