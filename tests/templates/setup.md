# Test Cases: steps/setup.yml

**Template:** `platform-templates/steps/setup.yml`
**Stage:** Setup (Stage 1)
**Steps under test:** `resolveTools` (bash), `validateParams` (bash)

All tests in this file are **Level 2** (ADO pipeline, dryRun: true) unless marked **L1** (static/local).

---

## TC-SETUP-001 — Happy path: all parameters valid, all variables present

**Level:** L2
**Type:** Positive

**Preconditions:**
- `platform-tool-versions` variable group contains: `DOCKER_BUILDKIT_VERSION`, `SYFT_VERSION`, `COSIGN_VERSION`, `HADOLINT_VERSION`, `NPM_REGISTRY_URL`, `ACR_HOST`
- Pipeline parameters: `tenantName=platform-test`, `appName=test-app`, `runtimeType=go`
- Dockerfile exists at `<buildContext>/Dockerfile`
- `steps/runtime/go.yml` exists in the platform-templates repository

**Expected result:** Stage 1 passes. Build log contains:
```
Tool versions resolved: Docker/BuildKit=... Syft=... Cosign=... Hadolint=... NPM_REGISTRY=... ACR=...
All parameters valid. tenantName=platform-test, appName=test-app, runtimeType=go
```

**Output variables emitted:** `ACR_HOST`, `HADOLINT_VERSION`, `SYFT_VERSION`, `COSIGN_VERSION`, `DOCKER_BUILDKIT_VERSION`, `NPM_REGISTRY_URL` — all non-empty.

---

## TC-SETUP-002 — tenantName: uppercase letter

**Level:** L1 / L2
**Type:** Negative

**Input:** `tenantName: Platform-Test`

**Expected result:** `validateParams` step fails. Build log contains:
```
##[error]tenantName 'Platform-Test' does not match required pattern ^[a-z0-9][a-z0-9-]*[a-z0-9]$ (lowercase alphanumeric and hyphens only, minimum 2 characters)
```

**Verification (L1):**
```bash
[[ "Platform-Test" =~ ^[a-z0-9][a-z0-9-]*[a-z0-9]$ ]] && echo "match" || echo "no match"
# Expected: no match
```

---

## TC-SETUP-003 — tenantName: underscore

**Level:** L1 / L2
**Type:** Negative

**Input:** `tenantName: platform_test`

**Expected result:** `validateParams` step fails. Pattern does not match underscore.

**Verification (L1):**
```bash
[[ "platform_test" =~ ^[a-z0-9][a-z0-9-]*[a-z0-9]$ ]] && echo "match" || echo "no match"
# Expected: no match
```

---

## TC-SETUP-004 — tenantName: single character (too short)

**Level:** L1 / L2
**Type:** Negative

**Input:** `tenantName: a`

**Expected result:** `validateParams` step fails. The regex requires minimum two characters because the pattern anchors on `[a-z0-9][a-z0-9-]*[a-z0-9]` — a single character cannot satisfy both the leading and trailing character class simultaneously.

**Verification (L1):**
```bash
[[ "a" =~ ^[a-z0-9][a-z0-9-]*[a-z0-9]$ ]] && echo "match" || echo "no match"
# Expected: no match
```

---

## TC-SETUP-005 — tenantName: leading hyphen

**Level:** L1 / L2
**Type:** Negative

**Input:** `tenantName: -platform-test`

**Expected result:** `validateParams` step fails. Leading hyphen does not match `^[a-z0-9]`.

---

## TC-SETUP-006 — tenantName: trailing hyphen

**Level:** L1 / L2
**Type:** Negative

**Input:** `tenantName: platform-test-`

**Expected result:** `validateParams` step fails. Trailing hyphen does not match `[a-z0-9]$`.

---

## TC-SETUP-007 — appName: same validation as tenantName

**Level:** L1 / L2
**Type:** Negative

**Input:** `tenantName: platform-test`, `appName: My_App`

**Expected result:** `validateParams` step fails with `appName 'My_App' does not match required pattern...`

---

## TC-SETUP-008 — Both tenantName and appName invalid: all errors reported together

**Level:** L2
**Type:** Negative

**Input:** `tenantName: Bad_Name`, `appName: Also_Bad`

**Expected result:** `validateParams` step fails and the build log contains **both** error messages before exit. Confirms the step collects all errors rather than short-circuiting on the first.

---

## TC-SETUP-009 — runtimeType: unsupported value

**Level:** L2
**Type:** Negative

**Input:** `runtimeType: java`

**Expected result:** `validateParams` step fails. Build log contains:
```
##[error]runtimeType 'java' is not supported. Allowed values: angular, react, springboot, python, go
```

---

## TC-SETUP-010 — runtimeType: all five valid values

**Level:** L2
**Type:** Positive

Run five separate pipelines with `runtimeType` set to each of: `go`, `python`, `springboot`, `angular`, `react`. Each must pass `validateParams`.

| runtimeType | Expected result |
|---|---|
| `go` | Pass |
| `python` | Pass |
| `springboot` | Pass |
| `angular` | Pass |
| `react` | Pass |

---

## TC-SETUP-011 — dockerfilePath: file does not exist

**Level:** L2
**Type:** Negative

**Input:** `dockerfilePath: docker/MyDockerfile` — a path that does not exist in the repository.

**Expected result:** `validateParams` step fails. Build log contains:
```
##[error]dockerfilePath not found: /agent/.../docker/MyDockerfile
```

The resolved path includes `$BUILD_SOURCESDIRECTORY/$BUILD_CONTEXT/$DOCKERFILE_PATH`.

---

## TC-SETUP-012 — dockerfilePath: file exists at non-default path

**Level:** L2
**Type:** Positive

**Precondition:** A Dockerfile exists at `docker/Dockerfile.prod` in the test repository.

**Input:** `dockerfilePath: Dockerfile.prod`, `buildContext: docker`

**Expected result:** `validateParams` passes. No error about missing Dockerfile.

---

## TC-SETUP-013 — Missing variable group entry: ACR_HOST empty

**Level:** L2
**Type:** Negative

**Precondition:** `platform-tool-versions` variable group has `ACR_HOST` removed or set to empty.

**Expected result:** `resolveTools` step fails. Build log contains:
```
##[error]Variable ACR_HOST is missing or empty in platform-tool-versions variable group
```

`validateParams` does not run (step fails before it).

---

## TC-SETUP-014 — Multiple missing variable group entries: all reported together

**Level:** L2
**Type:** Negative

**Precondition:** `ACR_HOST` and `SYFT_VERSION` removed from the variable group.

**Expected result:** `resolveTools` step fails and the build log contains **both** error messages. Confirms error collection, not short-circuit.

---

## TC-SETUP-015 — Output variables are available in Stage 2

**Level:** L2
**Type:** Positive — cross-stage variable propagation

**Verification:** After a successful Stage 1, check that `ACR_HOST` is resolvable in Stage 2:

Add a diagnostic step to the Build stage job that prints `$(stageDependencies.Setup.Setup.outputs['resolveTools.ACR_HOST'])`. Confirm it is non-empty and matches the value in the variable group.

**Note:** This validates ADO's cross-stage output variable propagation, not just the step logic.

---

## TC-SETUP-016 — Runtime template existence check: missing template file

**Level:** L2
**Type:** Negative

**Scenario:** `runtimeType: go` is supplied but `steps/runtime/go.yml` is temporarily removed from the platform-templates repository.

**Expected result:** `validateParams` step fails. Build log contains:
```
##[error]Runtime template not found: steps/runtime/go.yml — contact platform engineering
```

**Note:** This test requires temporarily modifying the platform-templates repository. Restore `go.yml` after testing.

---

## TC-SETUP-017 — Two-character minimum: exactly two characters

**Level:** L1
**Type:** Positive boundary

**Input:** `tenantName: ab`

**Verification (L1):**
```bash
[[ "ab" =~ ^[a-z0-9][a-z0-9-]*[a-z0-9]$ ]] && echo "match" || echo "no match"
# Expected: match
```

**Note:** The regex requires at least two characters because `[a-z0-9][a-z0-9-]*[a-z0-9]` demands a leading character AND a trailing character. `[a-z0-9-]*` matches zero or more middle characters, making exactly two characters the minimum that can satisfy both anchor captures.

Wait — this is incorrect. `^[a-z0-9][a-z0-9-]*[a-z0-9]$` with `ab`:
- `^[a-z0-9]` matches `a`
- `[a-z0-9-]*` matches `` (zero characters)
- `[a-z0-9]$` matches `b`
So `ab` (2 chars) matches correctly.

---

## TC-SETUP-018 — tenantName: numeric only

**Level:** L1
**Type:** Positive

**Input:** `tenantName: 42`

**Verification (L1):** The pattern `^[a-z0-9][a-z0-9-]*[a-z0-9]$` allows digits — `42` should match.

```bash
[[ "42" =~ ^[a-z0-9][a-z0-9-]*[a-z0-9]$ ]] && echo "match" || echo "no match"
# Expected: match
```
