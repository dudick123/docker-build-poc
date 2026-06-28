## ADDED Requirements

### Requirement: Build tool detected from project files or Stage 1 fails
The `steps/runtime/springboot.yml` template SHALL detect the build tool by checking for `gradlew` (Gradle wrapper) or `pom.xml` in `buildContext`. If `gradlew` is present, Gradle is selected. If `pom.xml` is present and `gradlew` is absent, Maven is selected. If neither is present, the step SHALL exit non-zero with the error: `"No build tool found in <buildContext>. Expected 'gradlew' (Gradle) or 'pom.xml' (Maven) for runtimeType: springboot."` The pipeline SHALL NOT proceed to the Docker build.

#### Scenario: gradlew present — Gradle selected
- **WHEN** `gradlew` exists at `$BUILD_SOURCESDIRECTORY/$BUILD_CONTEXT/gradlew`
- **THEN** build tool is set to Gradle; the step continues

#### Scenario: pom.xml present, gradlew absent — Maven selected
- **WHEN** `pom.xml` exists but `gradlew` does not
- **THEN** build tool is set to Maven; the step continues

#### Scenario: both gradlew and pom.xml present — Gradle selected
- **WHEN** both `gradlew` and `pom.xml` exist in `buildContext`
- **THEN** Gradle is selected (takes priority); the step continues

#### Scenario: neither gradlew nor pom.xml present — hard fail
- **WHEN** neither `gradlew` nor `pom.xml` exists in `buildContext`
- **THEN** the step emits `##vso[task.logissue type=error]...` with the resolved path and exits non-zero; no Docker build runs

### Requirement: Dockerfile must contain a test-export stage or Stage 2 fails
The `springboot.yml` template SHALL check that the tenant's Dockerfile contains the string `AS test-export` before invoking `docker build`. If absent, the step SHALL exit non-zero with the error: `"Dockerfile at <path> does not contain a 'test-export' stage. Spring Boot builds require 'FROM scratch AS test-export' to extract test results. See platform reference Dockerfile at docs/reference-dockerfiles/springboot.Dockerfile."` The pipeline SHALL NOT proceed to the Docker build.

#### Scenario: Dockerfile contains test-export stage
- **WHEN** the Dockerfile contains the text `AS test-export` (e.g., `FROM scratch AS test-export`)
- **THEN** the check passes and the two-invocation build sequence begins

#### Scenario: Dockerfile missing test-export stage
- **WHEN** no line in the Dockerfile contains `AS test-export`
- **THEN** the step exits non-zero with an error message referencing the platform reference Dockerfile; no Docker build runs

### Requirement: Two-invocation BuildKit sequence with test gate
The `springboot.yml` template SHALL execute `docker build` twice for Spring Boot runtimes. The first invocation targets `test-export` and outputs test results to the agent filesystem. The `PublishTestResults` ADO task then publishes those results with `failTaskOnFailedTests: true`. Test failure at this point SHALL block the pipeline — the final image MUST NOT build when tests fail. The second invocation targets `final` and completes the image build using BuildKit cache from the first invocation.

#### Scenario: Tests pass — final image built
- **WHEN** the first `docker build --target test-export` completes with exit 0 and all JUnit XML in `./test-results/` reports no failures
- **THEN** `PublishTestResults` succeeds; the second `docker build --target final` runs; the Build stage completes successfully

#### Scenario: Tests fail — pipeline blocked
- **WHEN** the first `docker build --target test-export` completes but `./test-results/` contains JUnit XML with one or more `<failure>` elements
- **THEN** `PublishTestResults` fails the step (via `failTaskOnFailedTests: true`); the second `docker build` does NOT run; the Build stage fails

#### Scenario: First docker build fails — no test results
- **WHEN** the first `docker build --target test-export` exits non-zero (e.g., compilation error)
- **THEN** the step exits non-zero; `PublishTestResults` is skipped; the second `docker build` does NOT run

### Requirement: Version extracted from project metadata and emitted as SPRINGBOOT_VERSION
The `springboot.yml` template SHALL attempt version extraction from the project metadata file for the detected build tool:
- **Gradle:** read the first `version = ...` line from `build.gradle`; if no match, fall back to `gradle.properties`
- **Maven:** read the first `<version>` element value from `pom.xml`

The extracted version SHALL be emitted as a step output variable named `SPRINGBOOT_VERSION` on a step named `springbootRuntime`. If no version is extractable, `SPRINGBOOT_VERSION` SHALL be emitted as empty string; Phase 8 skips the version tag.

#### Scenario: Gradle build.gradle with version property
- **WHEN** `build.gradle` contains `version = '2.3.0'` or `version = "2.3.0"`
- **THEN** the step emits `SPRINGBOOT_VERSION=2.3.0`

#### Scenario: Gradle gradle.properties with version property
- **WHEN** `build.gradle` yields no version and `gradle.properties` contains `version=1.0.0`
- **THEN** the step emits `SPRINGBOOT_VERSION=1.0.0`

#### Scenario: Maven pom.xml with version element
- **WHEN** `pom.xml` contains `<version>3.1.2</version>` as its first `<version>` element
- **THEN** the step emits `SPRINGBOOT_VERSION=3.1.2`

#### Scenario: No static version extractable
- **WHEN** no version string can be extracted via grep/sed from the project metadata file
- **THEN** the step emits `SPRINGBOOT_VERSION=` (empty string); Phase 8 skips the version tag; the pipeline continues

### Requirement: springboot.yml performs no agent-side compilation or toolchain invocation
The `springboot.yml` step template SHALL NOT invoke `./gradlew`, `mvn`, `java`, or any other JVM or build tool command. No Java SDK, Gradle, or Maven installation SHALL be required on the pipeline agent. All compilation and test execution happen inside the Docker build via BuildKit.

#### Scenario: Agent has no Java toolchain installed
- **WHEN** the pipeline agent has no `java`, `./gradlew`, or `mvn` binary available
- **THEN** `springboot.yml` completes successfully (file existence checks and grep/sed only); all Java compilation occurs inside the Docker container
