# Example: Spring Boot (Gradle)

A complete, annotated example of the platform container build pipeline for a Spring Boot application built with Gradle. All files can be copied into an application repository root and customized.

---

## Files in This Example

| File | Purpose |
|---|---|
| `azure-pipelines.yml` | ADO pipeline definition |
| `Dockerfile` | Three-stage Dockerfile: `build` → `test-export` → `final` |
| `README.md` | This document |

---

## How It Works

The Spring Boot pipeline runs the Docker build twice using BuildKit targets:

```
Stage 2 (Build)
  ├── dockerfile-lint.yml  → Hadolint checks Dockerfile
  ├── docker-build.yml     → BuildKit build (full image, default target)
  └── runtime/springboot.yml:
        1. Detects gradlew → build tool is Gradle
        2. Extracts version from build.gradle / gradle.properties
        3. Asserts Dockerfile contains 'AS test-export'
        4. docker build --target test-export --output type=local,dest=./test-results
        5. PublishTestResults@2  ← fails pipeline if any test fails
        6. docker build --target final  ← near-instant (BuildKit cache reuse)
```

BuildKit caches every intermediate layer with `mode=max`. The second `docker build` invocation reuses the `build` stage layers — only the layers unique to `final` execute. For a typical Spring Boot service, the first build takes 3–5 minutes; subsequent builds with unchanged dependencies take 30–60 seconds.

---

## Required Repository Structure

```
order-service/
  gradlew             ← required: pipeline detects Gradle by this file
  gradle/
    wrapper/
      gradle-wrapper.jar
      gradle-wrapper.properties
  build.gradle        ← version extracted from here
  gradle.properties   ← fallback version source
  settings.gradle
  src/
    main/java/...
    test/java/...
  Dockerfile          ← required: must contain 'AS test-export' and 'AS final'
  azure-pipelines.yml
```

### gradlew must be committed

The Gradle wrapper script (`gradlew`) is frequently added to `.gitignore` by accident. Remove the ignore rule and commit `gradlew`, `gradle/wrapper/gradle-wrapper.jar`, and `gradle/wrapper/gradle-wrapper.properties`. The pipeline hard-fails if `gradlew` is absent.

### Dockerfile stage names are fixed

The pipeline targets stages by name. The Dockerfile **must** contain these exact strings:

- `AS test-export` — targeted for JUnit XML extraction
- `AS final` — targeted for the production image build

These can appear anywhere in the file; the pipeline does a string search.

---

## Version Tagging

The pipeline reads the version from Gradle project files in this order:

1. `build.gradle` — `version = '2.3.1'` or `version = "2.3.1"`
2. `gradle.properties` — `version=2.3.1`

Example `build.gradle`:

```groovy
plugins {
    id 'org.springframework.boot' version '3.2.0'
    id 'io.spring.dependency-management' version '1.1.4'
    id 'java'
}

group = 'com.example'
version = '2.3.1'      // ← extracted by the pipeline

java {
    sourceCompatibility = '21'
}
```

| Branch | Tags pushed to ACR |
|---|---|
| Feature branch, version found | `<sha>`, `<branch>-<short-sha>`, `2.3.1-<short-sha>` |
| `main`, tag `2.3.1` is new | `<sha>`, `main-<short-sha>`, `2.3.1` |
| `main`, tag `2.3.1` exists | **Pipeline fails** — bump the version before merging |
| No version in project files | `<sha>`, `<branch>-<short-sha>` |

---

## Customizing the Dockerfile

### Multi-module Gradle projects

If this is a module in a multi-module project, target the specific module in the build command and adjust JAR and test result paths:

```dockerfile
# In the build stage
RUN ./gradlew :order-service:bootJar --no-daemon

# In test-export stage
FROM scratch AS test-export
COPY --from=build /app/order-service/build/test-results/test/*.xml /test-results/

# In final stage
COPY --from=build /app/order-service/build/libs/*.jar app.jar
```

### Integration tests with a separate task

If you have separate `test` and `integrationTest` tasks and want both reported:

```dockerfile
RUN ./gradlew test integrationTest --no-daemon

FROM scratch AS test-export
COPY --from=build /app/build/test-results/test/*.xml /test-results/unit/
COPY --from=build /app/build/test-results/integrationTest/*.xml /test-results/integration/
```

The pipeline scans `./test-results/**/*.xml` recursively — subdirectories are supported.

### Changing the JDK version

Update `eclipse-temurin:21-jdk-alpine` (build stage) and `eclipse-temurin:21-jre-alpine` (final stage) to a supported LTS version. Always pin to the same major version in both stages:

```dockerfile
FROM eclipse-temurin:17-jdk-alpine AS build
# ...
FROM eclipse-temurin:17-jre-alpine AS final
```

### Spring Boot Actuator — exposing the Git SHA

In `application.properties`:

```properties
management.endpoints.web.exposure.include=health,info
management.info.env.enabled=true
info.app.version=${GIT_COMMIT_SHA:dev}
```

The `GIT_COMMIT_SHA` environment variable is set in the final image from the build arg.

### Customizing JVM memory

Override `JAVA_OPTS` in the pod spec or Kubernetes `ConfigMap` rather than in the Dockerfile. The Dockerfile sets a container-aware default; adjust per pod based on the workload:

```yaml
# Kubernetes Deployment (excerpt)
env:
  - name: JAVA_TOOL_OPTIONS
    value: "-XX:+UseContainerSupport -XX:MaxRAMPercentage=70.0 -Xss256k"
```

---

## Hadolint Considerations

Hadolint issues specific to Spring Boot Dockerfiles:

| Rule | Issue | Fix |
|---|---|---|
| `DL3018` | `apk add` without `--no-cache` | Add `--no-cache` or use `apk add --update && rm -rf /var/cache/apk/*` |
| `DL3002` | `USER` changed to root after a non-root `USER` directive | Do not switch back to root after setting the application user |

---

## What the Pipeline Produces

For a successful non-dry-run run on `main` with `version = '2.3.1'`:

### ACR tags (at `<platform-acr>/commerce/order-service`)

| Tag | Use |
|---|---|
| `<40-char-sha>` | Kustomize manifests (immutable) |
| `main-<12-char-sha>` | Human navigation |
| `2.3.1` | Release version (main branch only) |

### ADO Test Results tab

All JUnit XML files exported from the `test-export` stage are published and visible in the pipeline run's **Tests** tab. Failed tests block the pipeline before the final image is built.

### Pipeline artifacts

| Artifact | Contents |
|---|---|
| `sbom-commerce-order-service` | CycloneDX JSON SBOM |
| `provenance-commerce-order-service` | JSON provenance record |

---

## Troubleshooting

**`No build tool found` — Stage 2 fails**
`gradlew` is absent from the repository root (or `buildContext` if set). Verify `gradlew` is committed and not in `.gitignore`.

**`Dockerfile does not contain a 'test-export' stage`**
The string `AS test-export` is missing from the Dockerfile. Add `FROM scratch AS test-export` exactly as shown. The pipeline does a literal string search.

**Tests fail and block the final image build**
This is the intended behavior. Download the `test-results/` directory from the pipeline artifacts or check the **Tests** tab in the ADO pipeline UI. Fix the failing tests before re-running.

**No JUnit XML files exported**
The `COPY --from=build` path in the `test-export` stage is wrong. Verify where Gradle actually writes test XML. Default: `build/test-results/test/*.xml`. If you have customized `testResultsDirName` in your Gradle build script, update the COPY source path accordingly.

**`OutOfMemoryError` during `./gradlew bootJar`**
Increase the Gradle JVM heap: `-Dorg.gradle.jvmargs="-Xmx2g"`. If the agent itself runs out of memory, contact platform engineering to request a higher-memory agent pool.

**Two JARs in `build/libs/`**
The `COPY --from=build /app/build/libs/*.jar app.jar` wildcard fails if multiple JARs match. This can happen when `bootJar` and `jar` both run. Disable the plain `jar` task in `build.gradle`:

```groovy
jar {
    enabled = false
}
```

Or specify the exact filename:

```dockerfile
COPY --from=build /app/build/libs/order-service-*.jar app.jar
```
