## Why

Phase 4 completes Go and Python pass-throughs. Spring Boot is the most complex runtime: its Dockerfile uses a dedicated `test-export` stage that lets BuildKit extract test results to the agent for ADO publication before the final image is assembled. Phase 5 implements `springboot.yml` to drive that two-invocation BuildKit sequence, publish test results, and hard-gate the final build on a passing test suite.

## What Changes

- **Replace stub** `platform-templates/steps/runtime/springboot.yml`:
  - Build tool detection: prefer Gradle wrapper (`gradlew` present); fall back to Maven (`pom.xml` present); hard fail Stage 1 if neither found
  - Version extraction: `<version>` element from `pom.xml` (Maven) or `version` property from `build.gradle` / `gradle.properties` (Gradle), emitted as `SPRINGBOOT_VERSION` step output variable
  - Dockerfile `test-export` stage assertion: check that the Dockerfile contains `AS test-export`; hard fail Stage 2 with a message directing the tenant to the platform reference Dockerfile if absent
  - Two-invocation BuildKit sequence:
    1. `docker build --target test-export --output type=local,dest=./test-results .` — extracts JUnit XML from the `test-export` stage
    2. ADO `PublishTestResults` task reads `./test-results/**/*.xml` — test failure at this step blocks the pipeline
    3. `docker build --target final .` — BuildKit cache hit on all layers from step 1; final image held locally (not pushed)
  - No pre-build steps on agent — compilation and test execution happen entirely inside Docker (Pattern A)

## Capabilities

### New Capabilities

- `springboot-runtime-validation`: The validations, version extraction, Dockerfile assertion, and two-invocation BuildKit sequence that `springboot.yml` performs before the Stage 3 sign step.

### Modified Capabilities

## Impact

- **Modified files:** `platform-templates/steps/runtime/springboot.yml`
- **No base template changes** — `container-build-v2.yml` already dispatches to `springboot.yml` and passes `buildContext`, `dockerfilePath`, and `acrHost`; no wiring changes needed
- **Produces for Phase 8:** `SPRINGBOOT_VERSION` step output variable — Phase 8 uses this for version tag construction; empty string means skip version tag
- **ADO task dependency:** `PublishTestResults` is a built-in ADO task; no new tool version pins needed in `platform-tool-versions`
- **Agent requirement:** Docker + BuildKit only (Pattern A); no Java/Gradle/Maven toolchain required on agent
