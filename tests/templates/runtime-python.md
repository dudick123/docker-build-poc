# Test Cases: steps/runtime/python.yml

**Template:** `platform-templates/steps/runtime/python.yml`
**Pattern:** A — build inside Dockerfile; runtime step performs advisory checks only
**Stage:** Build (Stage 2) — runs before docker-build.yml
**Steps under test:** `validatePythonProject` (bash), `extractPythonVersion` (bash)

---

## TC-PY-001 — Happy path: pyproject.toml present, project name extracted

**Level:** L2
**Type:** Positive

**Precondition:** `fixtures/projects/pyproject.toml` committed at the build context root. Content:
```toml
[tool.poetry]
name = "test-app"
version = "2.1.0"
description = "Platform test application"
```

**Expected result:** `validatePythonProject` step passes. Build log contains:
```
pyproject.toml found at <buildContext>/pyproject.toml
Project name: test-app
Version: 2.1.0
```

Pipeline continues to `dockerfile-lint.yml`.

---

## TC-PY-002 — requirements.txt fallback: pyproject.toml absent but requirements.txt present

**Level:** L2
**Type:** Positive (advisory)

**Precondition:** `requirements.txt` committed in the build context. No `pyproject.toml`.

**Expected result:** `validatePythonProject` step passes (exit 0). Build log contains:
```
Warning: pyproject.toml not found — requirements.txt detected at <buildContext>/requirements.txt.
No version extraction available from requirements.txt.
```

Pipeline continues. `RUNTIME_VERSION` is empty.

---

## TC-PY-003 — No dependency file at all: advisory warning, pipeline continues

**Level:** L2
**Type:** Positive (advisory)

**Precondition:** Neither `pyproject.toml` nor `requirements.txt` nor `setup.py` exists in the build context.

**Expected result:** `validatePythonProject` step passes (exit 0). Build log contains:
```
Warning: No Python dependency file found (pyproject.toml, requirements.txt, setup.py). Proceeding — ensure the Dockerfile handles dependencies internally.
```

Pipeline is NOT blocked.

---

## TC-PY-004 — pyproject.toml version extracted: runtimeVersion output emitted

**Level:** L2
**Type:** Positive

**Precondition:** `pyproject.toml` with `version = "2.1.0"`.

**Expected result:** `extractPythonVersion` step passes. Build log contains:
```
Runtime version extracted from pyproject.toml: 2.1.0
##vso[task.setvariable variable=RUNTIME_VERSION;isOutput=true]2.1.0
```

`RUNTIME_VERSION` output variable is emitted with value `2.1.0`.

---

## TC-PY-005 — pyproject.toml uses dynamic version field: no version emitted

**Level:** L1
**Type:** Positive — edge case

**Precondition:** `pyproject.toml` with:
```toml
[project]
name = "test-app"
dynamic = ["version"]
```

**Verification (L1):** The extraction regex looks for a literal `version = "..."` pattern. A `dynamic` field does not match. `RUNTIME_VERSION` is empty (no failure).

Build log contains:
```
Warning: version field not found in pyproject.toml or uses dynamic versioning — skipping version tag.
```

---

## TC-PY-006 — setup.py present: advisory message only, no version extraction

**Level:** L2
**Type:** Positive (advisory)

**Precondition:** `setup.py` present. No `pyproject.toml`.

**Expected result:** `validatePythonProject` step passes. Build log contains:
```
Warning: setup.py detected. Version extraction is not supported for setup.py projects. Proceeding — no version tag will be applied.
```

`RUNTIME_VERSION` is empty.

---

## TC-PY-007 — runtimeVersion propagates to Stage 4 (cross-stage variable)

**Level:** L2/L3
**Type:** Positive — cross-stage variable propagation

**Precondition:** `pyproject.toml` with `version = "3.0.1"`.

**Verification:** In Stage 4, `RUNTIME_VERSION` arrives via:
```
$(stageDependencies.Build.Build.outputs['extractPythonVersion.RUNTIME_VERSION'])
```

Add a diagnostic `echo` step at the beginning of Stage 4 to confirm the value is `3.0.1`.
