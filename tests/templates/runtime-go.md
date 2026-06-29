# Test Cases: steps/runtime/go.yml

**Template:** `platform-templates/steps/runtime/go.yml`
**Pattern:** A — build inside Dockerfile; runtime step performs advisory checks only
**Stage:** Build (Stage 2) — runs before docker-build.yml
**Steps under test:** `validateGoModule` (bash), `extractGoVersion` (bash)

---

## TC-GO-001 — Happy path: go.mod present, module name extracted

**Level:** L2
**Type:** Positive

**Precondition:** `fixtures/projects/go.mod` committed at the repository root (or `buildContext`). Content:
```
module github.com/platform-test/test-app

go 1.22
```

**Expected result:** `validateGoModule` step passes. Build log contains:
```
go.mod found at <buildContext>/go.mod
Module: github.com/platform-test/test-app
Go version: 1.22
```

No step failure. Pipeline continues to `dockerfile-lint.yml`.

---

## TC-GO-002 — go.mod absent: advisory warning, pipeline continues

**Level:** L2
**Type:** Positive (advisory)

**Precondition:** `go.mod` does NOT exist in the repository.

**Expected result:** `validateGoModule` step passes (exit 0). Build log contains:
```
Warning: go.mod not found at <buildContext>/go.mod — skipping module validation.
```

Pipeline continues to `dockerfile-lint.yml`. Pipeline does NOT fail.

---

## TC-GO-003 — VERSION file present: runtimeVersion output emitted

**Level:** L2
**Type:** Positive

**Precondition:** `fixtures/projects/VERSION` file with content `1.4.2` committed at the build context root.

**Expected result:** `extractGoVersion` step passes. Build log contains:
```
Runtime version extracted from VERSION file: 1.4.2
##vso[task.setvariable variable=RUNTIME_VERSION;isOutput=true]1.4.2
```

`RUNTIME_VERSION` output variable is emitted with value `1.4.2`. Stage 4 uses this for the version tag.

---

## TC-GO-004 — VERSION file missing: empty runtimeVersion, no failure

**Level:** L2
**Type:** Positive

**Precondition:** No `VERSION` file in the repository.

**Expected result:** `extractGoVersion` step passes (exit 0). `RUNTIME_VERSION` output variable is empty or not emitted. Build log contains:
```
No VERSION file found at <buildContext>/VERSION — skipping version tag.
```

Stage 4 pushes only the full-SHA and alias tags (no version tag).

---

## TC-GO-005 — VERSION file contains 'latest': pipeline fails at assertTags

**Level:** L3
**Type:** Negative

**Precondition:** `VERSION` file contains the string `latest`.

**Expected result:** `extractGoVersion` passes (it reads the value without validation). The `assertTags` step in Stage 4 fails. See TC-PUB-002 for the exact error message.

**Note:** Version validation happens in Stage 4 (assertTags), not Stage 2. This TC confirms the failure location.

---

## TC-GO-006 — VERSION file with leading/trailing whitespace: whitespace stripped

**Level:** L1
**Type:** Positive — edge case

**Precondition:** `VERSION` file contains `  1.4.2  \n` (leading/trailing whitespace and newline).

**Verification (L1):** The extraction command uses `$(cat VERSION | tr -d '[:space:]')` or `$(cat VERSION | xargs)`. Confirm the output variable is `1.4.2`, not `  1.4.2  `.

---

## TC-GO-007 — go.mod with replace directives: module name still extracted

**Level:** L1
**Type:** Positive — edge case

**Precondition:** `go.mod` containing:
```
module github.com/platform-test/test-app

go 1.22

require (
  github.com/some/dep v1.0.0
)

replace github.com/some/dep => ../local-dep
```

**Verification (L1):** The extraction command reads only the first `module` line. `replace` directives do not affect the module name extraction.

---

## TC-GO-008 — runtimeVersion propagates to Stage 4 (cross-stage variable)

**Level:** L2/L3
**Type:** Positive — cross-stage variable propagation

**Precondition:** `VERSION` file containing `1.5.0`.

**Verification:** In Stage 4, the `RUNTIME_VERSION` value arrives via:
```
$(stageDependencies.Build.Build.outputs['extractGoVersion.RUNTIME_VERSION'])
```

Add a diagnostic `echo` step at the beginning of Stage 4 to confirm the value is `1.5.0` and non-empty.
