# Pipeline Template Testing Plan

Testing strategy, environment requirements, test levels, execution guide, and traceability index for all step templates in `platform-templates/`.

---

## Scope

This plan covers every step template in the pipeline:

| Template | Test file | Test ID prefix |
|---|---|---|
| `steps/setup.yml` | [templates/setup.md](templates/setup.md) | `TC-SETUP` |
| `steps/dockerfile-lint.yml` | [templates/dockerfile-lint.md](templates/dockerfile-lint.md) | `TC-LINT` |
| `steps/docker-build.yml` | [templates/docker-build.md](templates/docker-build.md) | `TC-BUILD` |
| `steps/sbom-sign-publish.yml` — signAndAttest phase | [templates/sbom-sign-publish.md](templates/sbom-sign-publish.md) | `TC-SIGN` |
| `steps/sbom-sign-publish.yml` — publish phase | [templates/sbom-sign-publish.md](templates/sbom-sign-publish.md) | `TC-PUB` |
| `steps/sbom-sign-publish.yml` — notify phase | [templates/sbom-sign-publish.md](templates/sbom-sign-publish.md) | `TC-NOTIFY` |
| `steps/runtime/go.yml` | [templates/runtime-go.md](templates/runtime-go.md) | `TC-GO` |
| `steps/runtime/python.yml` | [templates/runtime-python.md](templates/runtime-python.md) | `TC-PY` |
| `steps/runtime/springboot.yml` | [templates/runtime-springboot.md](templates/runtime-springboot.md) | `TC-SB` |
| `steps/runtime/angular.yml` | [templates/runtime-angular.md](templates/runtime-angular.md) | `TC-ANG` |
| `steps/runtime/react.yml` | [templates/runtime-react.md](templates/runtime-react.md) | `TC-REACT` |

---

## Test Levels

Tests are organized into three levels based on infrastructure requirements:

### Level 1 — Static / Local

Tests that can be verified without an ADO pipeline run. Validate script logic, regex patterns, and file structure expectations by reading the template source or running the bash scripts manually.

**Infrastructure required:** None (bash shell)

**Examples:** Parameter regex validation, version extraction regex, file-existence check logic

### Level 2 — ADO Pipeline (dryRun: true)

Tests that run an ADO pipeline with `dryRun: true`. Sign & Attest (Stage 3) and Publish (Stage 4) are skipped, so no ACR pushes occur. Covers Setup validation, Hadolint, Docker build, all five runtime validators, and Notify.

**Infrastructure required:**
- ADO organization with `platform-templates` loaded as a resource
- `platform-tool-versions` variable group with valid entries
- Agent with Docker + BuildKit
- A minimal test repository (fixture Dockerfiles committed)

**Test pipelines:** See [pipelines/](pipelines/)

### Level 3 — ADO Pipeline (full run)

Tests that require a live end-to-end run: ACR pushes, Cosign signing, SBOM attestation, provenance artifacts, PR comment. These require all hard dependencies provisioned.

**Infrastructure required:** All Level 2 requirements, plus:
- Shared ACR with push permission for the test tenant service connection
- Cosign key pair in Azure Key Vault (`cosign-private-key`, `cosign-public-key`)
- `COSIGN_AKV_SERVICE_CONNECTION` and `COSIGN_KEY_VAULT_NAME` pipeline variables
- A PR open against the test repository for Notify tests

---

## Test Environment Setup

### Minimum test tenant

All tests use a dedicated platform test tenant. Request provisioning from platform engineering:

```
tenantName:  platform-test
appName:     (varies per test — see individual test cases)
```

Required platform provisioning:
- ADO service connection: `platform-test-acr-push` scoped to `platform-test/*` in shared ACR
- Variable group: `platform-tool-versions` accessible to the test ADO project
- Pipeline variables: `COSIGN_AKV_SERVICE_CONNECTION`, `COSIGN_KEY_VAULT_NAME`

### Test repository structure

A dedicated `platform-template-tests` ADO repository containing the fixture files from [fixtures/](fixtures/). Each test case's `azure-pipelines.yml` file in [pipelines/](pipelines/) references fixtures by path.

The repository should be organized as:

```
platform-template-tests/
  azure-pipelines.yml         ← whichever test pipeline is active
  fixtures/
    dockerfiles/
      Dockerfile.go-valid
      Dockerfile.python-valid
      Dockerfile.springboot-valid
      Dockerfile.lint-error
      Dockerfile.springboot-no-test-export
    projects/
      go.mod
      VERSION
      pyproject.toml
      package.json.angular
      package.json.react
      next.config.js
      build.gradle
      gradle.properties
      pom.xml
```

---

## Execution Guide

### Level 2 tests (dryRun)

1. Copy the desired test pipeline from `tests/pipelines/` to the `platform-template-tests` repository as `azure-pipelines.yml`
2. Commit the fixture files from `tests/fixtures/` to the repository
3. Queue the pipeline manually in ADO
4. Observe the build log for expected pass/fail messages
5. Compare to the expected results column in the relevant test case file

### Level 3 tests (full run)

1. Follow Level 2 setup
2. Open a draft PR in `platform-template-tests` against the `main` branch
3. Change `dryRun: false` in the test pipeline or remove the dryRun parameter
4. Queue the pipeline on the PR branch
5. Verify ACR tags, Cosign objects, ADO artifacts, and PR comment

### Negative tests (expected failures)

For tests where a pipeline failure is the expected outcome:
1. Queue the pipeline
2. Confirm the pipeline fails at the expected stage and step
3. Check the build log for the exact error message specified in the test case
4. Confirm no downstream stages ran

---

## Test Fixtures

| File | Used by |
|---|---|
| `fixtures/dockerfiles/Dockerfile.go-valid` | TC-LINT-001, TC-BUILD-001, TC-GO-* |
| `fixtures/dockerfiles/Dockerfile.python-valid` | TC-PY-* |
| `fixtures/dockerfiles/Dockerfile.springboot-valid` | TC-SB-* |
| `fixtures/dockerfiles/Dockerfile.lint-error` | TC-LINT-002, TC-LINT-003 |
| `fixtures/dockerfiles/Dockerfile.springboot-no-test-export` | TC-SB-005 |
| `fixtures/projects/go.mod` | TC-GO-001, TC-GO-002 |
| `fixtures/projects/VERSION` | TC-GO-004 through TC-GO-008 |
| `fixtures/projects/pyproject.toml` | TC-PY-003, TC-PY-004 |
| `fixtures/projects/package.json.angular` | TC-ANG-001, TC-ANG-002 |
| `fixtures/projects/package.json.react` | TC-REACT-001 |
| `fixtures/projects/next.config.js` | TC-REACT-005 |
| `fixtures/projects/build.gradle` | TC-SB-002, TC-SB-007 |
| `fixtures/projects/pom.xml` | TC-SB-008 |

---

## Test Pipeline Files

| File | Tests |
|---|---|
| `pipelines/test-go-dryrun.yml` | TC-GO-*, TC-LINT-001, TC-BUILD-001 |
| `pipelines/test-python-dryrun.yml` | TC-PY-*, TC-LINT-001, TC-BUILD-001 |
| `pipelines/test-springboot-gradle-dryrun.yml` | TC-SB-001 through TC-SB-007 |
| `pipelines/test-springboot-maven-dryrun.yml` | TC-SB-001 (Maven path), TC-SB-008 |
| `pipelines/test-angular-dryrun.yml` | TC-ANG-*, TC-BUILD-001 |
| `pipelines/test-react-dryrun.yml` | TC-REACT-* |
| `pipelines/test-setup-negative.yml` | TC-SETUP-002 through TC-SETUP-012 |
| `pipelines/test-lint-negative.yml` | TC-LINT-002, TC-LINT-003 |

---

## Traceability Matrix

Maps PRD functional requirements to test case IDs.

| PRD Requirement | Test Cases |
|---|---|
| FR-1: Parameter validation | TC-SETUP-002 through TC-SETUP-012 |
| FR-2: Tool version resolution | TC-SETUP-013 through TC-SETUP-018 |
| FR-3: Dockerfile linting (Hadolint) | TC-LINT-001 through TC-LINT-006 |
| FR-4: BuildKit image build | TC-BUILD-001 through TC-BUILD-007 |
| FR-5: SBOM generation (Syft) | TC-SIGN-001 through TC-SIGN-003 |
| FR-6: Image signing (Cosign) | TC-SIGN-004 through TC-SIGN-007 |
| FR-7: ACR publish + digest verify | TC-PUB-001 through TC-PUB-008 |
| FR-8: latest tag prohibition | TC-PUB-002 through TC-PUB-004 |
| FR-9: Notify (PR comment, build tag, Teams) | TC-NOTIFY-001 through TC-NOTIFY-009 |
| FR-10: Go runtime | TC-GO-001 through TC-GO-008 |
| FR-11: Python runtime | TC-PY-001 through TC-PY-007 |
| FR-12: Spring Boot runtime | TC-SB-001 through TC-SB-011 |
| FR-13: Angular runtime | TC-ANG-001 through TC-ANG-005 |
| FR-14: React / Next.js runtime | TC-REACT-001 through TC-REACT-007 |
