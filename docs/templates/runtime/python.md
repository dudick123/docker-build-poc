# Template: steps/runtime/python.yml

**Path:** `platform-templates/steps/runtime/python.yml`
**Type:** Step template (runtime validator)
**Called by:** `container-build-v2.yml` — Stage 2 (Build), Job: Build, when `runtimeType: python`

---

## Description

Checks for a recognized Python dependency file in the build context and extracts a version string from project metadata for optional ACR version tagging. All checks are advisory — no hard failures are possible from this step. This step runs after the Docker build completes.

Python uses Pattern A: pip, Poetry, PDM, or any other Python toolchain runs entirely inside the Dockerfile. No Python interpreter or package manager runs on the pipeline agent.

---

## Summary

| Property | Value |
|---|---|
| Steps | 1 (bash, named `pythonRuntime`) |
| Hard failures | None |
| Advisory warnings | Missing dependency file |
| Output variables | `PYTHON_VERSION` (empty string if not extractable) |

---

## Parameters

| Name | Type | Default | Description |
|---|---|---|---|
| `buildContext` | string | `.` | Build context directory |

---

## Steps

### Step 1 — `pythonRuntime` (bash)

**Dependency file check (advisory):**

Checks for any of the following files in the build context:

| File | Tool |
|---|---|
| `requirements.txt` | pip |
| `pyproject.toml` | Poetry, PDM, Hatch, setuptools |
| `poetry.lock` | Poetry |

If none is found, emits an advisory warning via `##vso[task.logissue type=warning]`. The build is not blocked.

**Version extraction (optional):**

Attempts extraction in this order:

1. **`pyproject.toml`** — regex `^version\s*=\s*"[^"]+"` captures `version = "x.y.z"` under `[project]` or `[tool.poetry]` sections.
2. **`setup.cfg`** — regex `^version\s*=` captures `version = x.y.z` under `[metadata]`.

If a version is found, it is emitted as `PYTHON_VERSION`. If neither file exists or neither contains a static version string, `PYTHON_VERSION` is emitted as an empty string and a notice is written to the log.

**Limitation:** Dynamic version fields are not supported. The following patterns are NOT matched:
- `version = {attr: mypackage.__version__}`
- `version = {file: VERSION}`
- `dynamic = ["version"]` (PEP 517 dynamic metadata)

Use a static version string or accept that the version tag will be skipped.

---

## Output Variables

| Variable | Step name | Cross-stage reference | Notes |
|---|---|---|---|
| `PYTHON_VERSION` | `pythonRuntime` | `$(stageDependencies.Build.Build.outputs['pythonRuntime.PYTHON_VERSION'])` | Empty string if no static version found; version tag skipped in Publish stage |

---

## Version Tagging Behavior

| Condition | Tags pushed to ACR |
|---|---|
| No static version in project files | `<40-char-sha>`, `<branch>-<12-char-sha>` |
| Version found, non-main branch | `<40-char-sha>`, `<branch>-<12-char-sha>`, `<version>-<12-char-sha>` |
| Version found, `main` branch, tag is new | `<40-char-sha>`, `main-<12-char-sha>`, `<version>` |
| Version found, `main` branch, tag exists | Pipeline fails — bump the version before merging |

---

## Usage

This template is called exclusively by `container-build-v2.yml` when `runtimeType: python`. It is not intended to be called directly by tenant pipelines.

```yaml
# Inside container-build-v2.yml
- ${{ if eq(parameters.runtimeType, 'python') }}:
  - template: steps/runtime/python.yml
    parameters:
      buildContext: ${{ parameters.buildContext }}
```
