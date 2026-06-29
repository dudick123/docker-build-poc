# Quick Start: Spring Boot (Gradle)

This guide covers what the pipeline requires from a Spring Boot application that uses Gradle. Read [QUICK-START.md](QUICK-START.md) first for prerequisites and the base `azure-pipelines.yml` structure.

For Maven projects, see [QUICK-START-MAVEN.md](QUICK-START-MAVEN.md).

---

## Pipeline File

```yaml
trigger:
  branches:
    include:
      - main

resources:
  repositories:
    - repository: platform-templates
      type: git
      name: <YourADOProject>/platform-templates

extends:
  template: container-build-v2.yml@platform-templates
  parameters:
    tenantName: my-team
    appName: my-service
    runtimeType: springboot
```

---

## Build Pattern

Spring Boot uses a self-contained multi-stage Dockerfile. The pipeline runs `docker build` twice against the same Dockerfile using BuildKit targets:

1. **`--target test-export`** — runs tests inside the container and exports JUnit XML results to the agent. The pipeline publishes these results and fails if any test fails.
2. **`--target final`** — builds the final application image. Because BuildKit reuses cached layers, this second build is near-instant.

No JDK, Gradle, or Maven is installed on the pipeline agent.

---

## Repository Requirements

### Gradle wrapper (required)

The pipeline detects the build tool by looking for `gradlew` at the build context root. If `gradlew` is present, Gradle is used. If `gradlew` is absent but `pom.xml` is present, Maven is used. If neither is found, Stage 2 fails with:

```
No build tool found. Expected 'gradlew' (Gradle) or 'pom.xml' (Maven) for runtimeType: springboot.
```

**Important:** The pipeline looks for `gradlew` (the Gradle wrapper script), not `gradle`. The wrapper script must be present and committed to the repository. Do not add `gradlew` to `.gitignore`.

### test-export stage in Dockerfile (required)

The Dockerfile **must** contain a stage named `test-export`. The pipeline's first build invocation targets this stage to extract JUnit XML test results. If the stage is missing, Stage 2 fails with:

```
Dockerfile does not contain a 'test-export' stage. Spring Boot builds require 'FROM scratch AS test-export'...
```

See the reference Dockerfile below for the required structure.

### Version extraction

The pipeline extracts the version for optional ACR version tags. For Gradle, it checks in this order:

1. `build.gradle` — line matching `^version\s*=\s*'...'` or `^version\s*=\s*"..."`
2. `gradle.properties` — line matching `^version\s*=\s*...`

Example `build.gradle`:
```groovy
version = '2.1.0'
```

Example `gradle.properties`:
```properties
version=2.1.0
```

If no version is found, the version tag is skipped.

---

## Reference Dockerfile (Gradle)

```dockerfile
# syntax=docker/dockerfile:1

# ── build stage ──────────────────────────────────────────────────────────────
FROM eclipse-temurin:21-jdk-alpine AS build

WORKDIR /app

# Copy Gradle wrapper and dependency declarations first for better layer caching
COPY gradlew gradlew
COPY gradle/ gradle/
COPY build.gradle settings.gradle ./

# Download dependencies (cached layer unless build.gradle changes)
RUN chmod +x gradlew && ./gradlew dependencies --no-daemon --quiet

COPY src/ src/

ARG GIT_COMMIT_SHA=dev

# Run tests and build the application JAR
RUN ./gradlew bootJar --no-daemon \
    -Dapp.build.gitSha=${GIT_COMMIT_SHA} \
    -Dorg.gradle.jvmargs="-Xmx1g"

# ── test-export stage ─────────────────────────────────────────────────────────
# This stage is required by the pipeline. The build invocation
#   docker build --target test-export --output type=local,dest=./test-results
# copies everything in this stage to the agent's ./test-results directory.
# JUnit XML files written to /test-results inside this stage are published
# to ADO Test Results and fail the pipeline if any test fails.
FROM scratch AS test-export
COPY --from=build /app/build/test-results/test/*.xml /test-results/

# ── final stage ──────────────────────────────────────────────────────────────
FROM eclipse-temurin:21-jre-alpine AS final

ARG GIT_COMMIT_SHA=dev

WORKDIR /app

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY --from=build /app/build/libs/*.jar app.jar

ENV GIT_COMMIT_SHA=${GIT_COMMIT_SHA} \
    JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"

USER appuser

EXPOSE 8080

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar /app/app.jar"]
```

### Key points

- **Stage names matter** — the pipeline targets `test-export` and `final` by name. Do not rename these stages.
- **`FROM scratch AS test-export`** — `scratch` is the correct base for a stage whose only purpose is to export files. It adds zero bytes to the image and has no attack surface.
- **JUnit XML path** — the `COPY --from=build` in `test-export` must point to wherever Gradle writes JUnit XML. The default Gradle path is `build/test-results/test/*.xml`. If you use a custom reports directory, adjust the path.
- **`--no-daemon`** — Gradle daemon processes are not appropriate in a container build context. Always pass `--no-daemon` for CI builds.
- **JRE in final stage** — use `eclipse-temurin:21-jre-alpine` (not `-jdk`) in the final stage. The JDK includes compiler tooling that is not needed at runtime and increases attack surface.
- **`-XX:+UseContainerSupport`** — enables the JVM's container-aware heap sizing. Without this, the JVM may size its heap based on the host's total memory rather than the container's memory limit, leading to OOMKilled pods in AKS.
- **`GIT_COMMIT_SHA` build arg** — injected by the pipeline as the full 40-character Git SHA. Pass it to Spring Boot's `BuildProperties` or embed it in an `application.properties` value for `/actuator/info` exposure.

---

## Multi-Module Gradle Projects

If your repository is a multi-module Gradle project, `gradlew` will still be at the project root. Adjust the `bootJar` task to reference the correct submodule:

```dockerfile
RUN ./gradlew :api-service:bootJar --no-daemon
```

And copy the JAR from the correct submodule output:

```dockerfile
COPY --from=build /app/api-service/build/libs/*.jar app.jar
```

---

## What the Pipeline Does

The Spring Boot runtime step (`steps/runtime/springboot.yml`) runs and performs:

1. Detects `gradlew` at the build context root — sets build tool to Gradle (hard fail if neither `gradlew` nor `pom.xml` found)
2. Extracts version from `build.gradle` or `gradle.properties` — emits as `SPRINGBOOT_VERSION`
3. Asserts the Dockerfile contains `AS test-export` — hard fail if absent
4. Runs `docker build --target test-export --output type=local,dest=./test-results` — exports JUnit XML to the agent
5. Publishes test results via `PublishTestResults@2` — fails the pipeline if any test fails
6. Runs `docker build --target final` — builds the final image (uses BuildKit cache; near-instant)

---

## Version Tagging Summary

| Scenario | Tags pushed to ACR |
|---|---|
| No version in `build.gradle` or `gradle.properties` | `<40-char-sha>`, `<branch>-<12-char-sha>` |
| Version found, non-main branch | `<40-char-sha>`, `<branch>-<12-char-sha>`, `<version>-<12-char-sha>` |
| Version found, `main` branch, tag does not exist | `<40-char-sha>`, `main-<12-char-sha>`, `<version>` |
| Version found, `main` branch, tag already exists | Pipeline fails — bump the version before merging |

---

## Common Issues

**`No build tool found` — Stage 2 fails**
`gradlew` is not at the build context root. Common causes: `gradlew` is in `.gitignore`, the build context is set to the wrong directory, or the project uses a non-standard wrapper location. Verify `gradlew` is committed to the repository and `buildContext` points to the directory that contains it.

**`Dockerfile does not contain a 'test-export' stage` — Stage 2 fails**
Add the `AS test-export` stage to your Dockerfile as shown in the reference above. The string `AS test-export` must appear in the Dockerfile literally — the pipeline does a `grep` for this string.

**Tests fail and block the build**
The `PublishTestResults@2` task is configured with `failTaskOnFailedTests: true`. Fix the failing tests. To diagnose, download the `test-results` directory from the pipeline artifacts or check the Test Results tab in the ADO pipeline UI.

**No JUnit XML files found in `test-export`**
The `COPY --from=build` in the `test-export` stage is pointing to the wrong path. Check where Gradle actually writes JUnit XML (default: `build/test-results/test/`) and update the `COPY` source accordingly.

**Out of memory during Gradle build**
Set the Gradle JVM heap explicitly:
```dockerfile
RUN ./gradlew bootJar --no-daemon -Dorg.gradle.jvmargs="-Xmx1g"
```
Platform agents have a default memory limit. Keep the Gradle JVM heap at or below 1 GB for typical builds.

**Slow builds — dependencies re-downloaded each run**
Layer caching is most effective when `build.gradle` and `gradle.properties` are copied before the source code and `./gradlew dependencies` runs as a separate `RUN` step. If you copy the entire source tree first, any source change invalidates the dependency cache layer.
