# Test Cases: steps/dockerfile-lint.yml

**Template:** `platform-templates/steps/dockerfile-lint.yml`
**Stage:** Build (Stage 2) — first step
**Steps under test:** Download Hadolint (bash), Run Hadolint (bash)

---

## TC-LINT-001 — Happy path: Dockerfile with no violations

**Level:** L2
**Type:** Positive

**Precondition:** `fixtures/dockerfiles/Dockerfile.go-valid` committed as `Dockerfile` in the test repository.

**Input Dockerfile:** See `fixtures/dockerfiles/Dockerfile.go-valid` — uses `COPY`, JSON `ENTRYPOINT`, pinned base images, non-root user. All rules pass.

**Expected result:** Both steps pass. Build log contains:
```
Hadolint downloaded: hadolint-Linux-x86_64 v2.x.x
Dockerfile lint passed.
```

---

## TC-LINT-002 — ERROR-level finding: ADD instead of COPY (DL3020)

**Level:** L2
**Type:** Negative

**Precondition:** `fixtures/dockerfiles/Dockerfile.lint-error` committed as `Dockerfile` in the test repository. This file contains `ADD . /app` instead of `COPY . /app`.

**Expected result:** "Run Hadolint" step fails. Build log contains:
```
DL3020 error: Use COPY instead of ADD for local files
```

Exit code is non-zero. The Docker build step does not run — Stage 2 fails at the lint step.

**Verification:** Confirm that the `buildImage` step name does NOT appear in the build log (Docker build was not attempted).

---

## TC-LINT-003 — WARNING-level finding only: does not block build

**Level:** L2
**Type:** Positive (advisory behavior)

**Precondition:** `fixtures/dockerfiles/Dockerfile.lint-warning-only` committed as `Dockerfile`. This file uses `apt-get install` without version pinning (DL3008 — Warning level by default).

**Expected result:** "Run Hadolint" step passes (exit code 0). Build log contains the DL3008 warning text but Stage 2 continues to the Docker build step.

**Verification:** Both the Hadolint warning AND the `Building image:` message appear in the log.

---

## TC-LINT-004 — Tenant .hadolint.yaml suppresses an ERROR-level rule

**Level:** L2
**Type:** Positive (tenant config override)

**Precondition:**
- `Dockerfile` contains `ADD . /app` (would normally trigger DL3020 ERROR)
- `.hadolint.yaml` at the build context root contains:
  ```yaml
  ignore:
    - DL3020
  ```

**Expected result:** "Run Hadolint" step passes. DL3020 is not reported. Build continues to Docker build step.

**Note:** This tests the `if [ -f "$BUILD_CONTEXT/.hadolint.yaml" ]; then CONFIG_FLAG="--config ..."` branch in `dockerfile-lint.yml`.

---

## TC-LINT-005 — .hadolint.yaml present but does not suppress the failing rule

**Level:** L2
**Type:** Negative

**Precondition:**
- `Dockerfile` contains `ADD . /app` (DL3020 ERROR)
- `.hadolint.yaml` suppresses a different rule (`DL3008`) but not `DL3020`

**Expected result:** "Run Hadolint" step fails. DL3020 is still reported as an error.

---

## TC-LINT-006 — Hadolint version is resolved from variable group (not hardcoded)

**Level:** L2
**Type:** Positive — tool version pin verification

**Precondition:** `HADOLINT_VERSION` in `platform-tool-versions` is set to `2.12.0`.

**Expected result:** "Download Hadolint" step downloads from:
```
https://github.com/hadolint/hadolint/releases/download/v2.12.0/hadolint-Linux-x86_64
```

Build log contains: `Hadolint downloaded: hadolint-Linux-x86_64 v2.12.0`

**Verification:** Grep the build log for the version string and confirm it matches the variable group value.

---

## TC-LINT-007 — buildContext non-default: Dockerfile found at subdirectory

**Level:** L2
**Type:** Positive

**Precondition:** Dockerfile at `services/api/Dockerfile` in the test repository.

**Input:** `buildContext: services/api`, `dockerfilePath: Dockerfile`

**Expected result:** Hadolint lints `services/api/Dockerfile`. Step passes.

**Note:** The step resolves the target as `$BUILD_CONTEXT/$DOCKERFILE_PATH`.
