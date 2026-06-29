# Template: steps/runtime/springboot.yml

**Path:** `platform-templates/steps/runtime/springboot.yml`
**Type:** Step template (runtime validator + two-invocation BuildKit)
**Called by:** `container-build-v2.yml` — Stage 2 (Build), Job: Build, when `runtimeType: springboot`

---

## Description

The Spring Boot runtime template is the most operationally complex of the five. In addition to the standard validation and version extraction, it runs the Docker build twice:

1. A first build targeting `test-export` to extract JUnit XML test results to the agent
2. A second build targeting `final` to produce the application image

Both builds use BuildKit, so layer caching makes the second build near-instant. Test results are published to ADO and hard-gate the second build — if any test fails, the pipeline stops before building the final image.

Build tool detection is automatic: if `gradlew` exists at the build context root, Gradle is used. If `gradlew` is absent but `pom.xml` exists, Maven is used. If neither is found, the step fails.

---

## Summary

| Property | Value |
|---|---|
| Steps | 4 (1 bash validation + 1 bash test build + 1 ADO task + 1 bash final build) |
| Named steps | `springbootRuntime`, `extractTests`, `buildFinalImage` |
| Hard failures | No build tool (`gradlew` / `pom.xml`); `AS test-export` missing from Dockerfile; test failures |
| Advisory warnings | None |
| Output variables | `SPRINGBOOT_VERSION` |
| ACR writes | None (image held locally after `buildFinalImage`) |

---

## Parameters

| Name | Type | Default | Description |
|---|---|---|---|
| `buildContext` | string | `.` | Build context directory |
| `dockerfilePath` | string | `Dockerfile` | Path to Dockerfile, relative to `buildContext` |
| `tenantName` | string | — | Used in final image local tag |
| `appName` | string | — | Used in final image local tag |
| `acrHost` | string | — | Not used in this template (passed for future extensibility) |

---

## Steps

### Step 1 — `springbootRuntime` (bash)

**Build tool detection (hard fail if neither found):**

| File checked | Build tool |
|---|---|
| `$BUILD_CONTEXT/gradlew` | Gradle |
| `$BUILD_CONTEXT/pom.xml` | Maven (only if `gradlew` absent) |

If neither file is found:
```
##vso[task.logissue type=error]No build tool found in <path>. Expected 'gradlew' (Gradle) or 'pom.xml' (Maven) for runtimeType: springboot.
exit 1
```

**Version extraction:**

For Gradle — checks in order:
1. `build.gradle` — regex `^version\s*=\s*['"]...['"]\` extracts `version = '2.1.0'` or `version = "2.1.0"`
2. `gradle.properties` — regex `^version\s*=\s*...` extracts `version=2.1.0`

For Maven:
1. `pom.xml` — extracts first `<version>` element content using `grep -oP '(?<=<version>)[^<]+'`

If no version is found, `SPRINGBOOT_VERSION` is emitted as empty and version tagging is skipped.

**Dockerfile `test-export` stage assertion (hard fail if absent):**

Greps the Dockerfile for the string `AS test-export`. If not found:
```
##vso[task.logissue type=error]Dockerfile at <path> does not contain a 'test-export' stage. Spring Boot builds require 'FROM scratch AS test-export' to extract test results. See platform reference Dockerfile at docs/reference-dockerfiles/springboot.Dockerfile.
exit 1
```

---

### Step 2 — `extractTests` (bash)

Runs BuildKit targeting the `test-export` stage and exports the stage filesystem to `./test-results` on the agent:

```bash
DOCKER_BUILDKIT=1 docker build \
  --target test-export \
  --output "type=local,dest=./test-results" \
  -f "$DOCKERFILE_PATH" \
  "$BUILD_CONTEXT"
```

Everything the Dockerfile copies into the `test-export` stage (`FROM scratch AS test-export / COPY --from=build /app/build/test-results/test/*.xml /test-results/`) is written to `./test-results` on the agent. The stage filesystem becomes the export — `FROM scratch` ensures no base image layers are included.

---

### Step 3 — `PublishTestResults@2`

Publishes JUnit XML from `./test-results/**/*.xml`:

```yaml
- task: PublishTestResults@2
  inputs:
    testResultsFormat: JUnit
    testResultsFiles: '**/*.xml'
    searchFolder: '$(System.DefaultWorkingDirectory)/test-results'
    failTaskOnFailedTests: true
```

`failTaskOnFailedTests: true` causes the Build stage to fail if any test in the XML reports is marked as failed. The pipeline stops here if tests fail — the final image build does not run.

Test results are visible in the ADO pipeline run's **Tests** tab and in PR summary views.

---

### Step 4 — `buildFinalImage` (bash)

Builds the final application image targeting the `final` stage:

```bash
DOCKER_BUILDKIT=1 docker build \
  --target final \
  -f "$DOCKERFILE_PATH" \
  -t "<tenantName>/<appName>:pipeline-<BUILD_BUILDID>" \
  "$BUILD_CONTEXT"
```

Because the `build` stage layers were already computed and cached during Step 2, this invocation reuses those layers and only executes layers unique to the `final` stage. Build time is typically a few seconds.

The image is held locally on the agent under the same tag format as `docker-build.yml` (`<tenantName>/<appName>:pipeline-<BUILD_BUILDID>`). It is not pushed to ACR here.

> **Note:** For Spring Boot, this step replaces the role of `docker-build.yml`'s `buildImage` step. The `docker-build.yml` template still runs first (it executes the single-target build for all runtimes), but for Spring Boot the final image for downstream stages is the one produced by `buildFinalImage` here. Both produce images with the same local tag pattern; the Spring Boot final image is the one that persists since it runs last.

---

## Output Variables

| Variable | Step name | Cross-stage reference | Notes |
|---|---|---|---|
| `SPRINGBOOT_VERSION` | `springbootRuntime` | `$(stageDependencies.Build.Build.outputs['springbootRuntime.SPRINGBOOT_VERSION'])` | Empty if no version extractable; version tag skipped in Publish |

---

## Dockerfile Requirements

The tenant Dockerfile **must** define these two named stages. No other names are accepted — the pipeline targets them by name.

```dockerfile
# Stage that exports test results
FROM scratch AS test-export
COPY --from=build /app/build/test-results/test/*.xml /test-results/   # Gradle
# OR
COPY --from=build /app/target/surefire-reports/*.xml /test-results/   # Maven

# Final application image
FROM eclipse-temurin:21-jre-alpine AS final
...
```

The `test-export` base must be `FROM scratch` (or any image that does not add unwanted content). The pipeline exports the entire stage filesystem; `scratch` ensures only the COPY'd XML files are exported.

---

## Version Tagging Behavior

| Condition | Tags pushed to ACR |
|---|---|
| No version in project files | `<40-char-sha>`, `<branch>-<12-char-sha>` |
| Version found, non-main branch | `<40-char-sha>`, `<branch>-<12-char-sha>`, `<version>-<12-char-sha>` |
| Version found, `main` branch, tag is new | `<40-char-sha>`, `main-<12-char-sha>`, `<version>` |
| Version found, `main` branch, tag exists | Pipeline fails — bump version before merging |

---

## Usage

This template is called exclusively by `container-build-v2.yml` when `runtimeType: springboot`.

```yaml
# Inside container-build-v2.yml
- ${{ if eq(parameters.runtimeType, 'springboot') }}:
  - template: steps/runtime/springboot.yml
    parameters:
      buildContext: ${{ parameters.buildContext }}
      dockerfilePath: ${{ parameters.dockerfilePath }}
      tenantName: ${{ parameters.tenantName }}
      appName: ${{ parameters.appName }}
      acrHost: $(stageDependencies.Setup.Setup.outputs['resolveTools.ACR_HOST'])
```
