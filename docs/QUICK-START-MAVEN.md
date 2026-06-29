# Quick Start: Spring Boot (Maven)

This guide covers what the pipeline requires from a Spring Boot application that uses Maven. Read [QUICK-START.md](QUICK-START.md) first for prerequisites and the base `azure-pipelines.yml` structure.

For Gradle projects, see [QUICK-START-GRADLE.md](QUICK-START-GRADLE.md).

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

Both Gradle and Maven projects use `runtimeType: springboot`. The pipeline auto-detects the build tool by checking for `gradlew` first, then `pom.xml`.

---

## Build Pattern

Spring Boot uses a self-contained multi-stage Dockerfile. The pipeline runs `docker build` twice against the same Dockerfile using BuildKit targets:

1. **`--target test-export`** — runs tests inside the container and exports JUnit XML results to the agent. The pipeline publishes these results and fails if any test fails.
2. **`--target final`** — builds the final application image. BuildKit layer caching makes this second invocation near-instant.

No JDK or Maven is installed on the pipeline agent.

---

## Repository Requirements

### pom.xml (required)

The pipeline detects Maven when `pom.xml` exists at the build context root and `gradlew` does not. If neither is present, Stage 2 fails with:

```
No build tool found. Expected 'gradlew' (Gradle) or 'pom.xml' (Maven) for runtimeType: springboot.
```

### test-export stage in Dockerfile (required)

The Dockerfile **must** contain a stage named `test-export`. This is required regardless of whether you use Gradle or Maven. If the stage is missing, Stage 2 fails with:

```
Dockerfile does not contain a 'test-export' stage. Spring Boot builds require 'FROM scratch AS test-export'...
```

See the reference Dockerfile below.

### Version extraction

The pipeline extracts the version from `pom.xml` for optional ACR version tags. It reads the first `<version>` element in the file, which is conventionally the project version declared at the top of `<project>`:

```xml
<project>
  <groupId>com.example</groupId>
  <artifactId>my-service</artifactId>
  <version>2.1.0</version>
  ...
</project>
```

**Important:** The pipeline uses a simple regex (`grep -oP '(?<=<version>)[^<]+'`) that matches the first `<version>` element. If your `pom.xml` inherits from a parent POM and declares `<parent><version>` before your project `<version>`, the parent version will be extracted instead. To avoid this, declare your project version before any `<parent>` block, or ensure your project version is the first `<version>` element in the file.

If no version is found or the file is absent, the version tag is skipped.

---

## Reference Dockerfile (Maven)

```dockerfile
# syntax=docker/dockerfile:1

# ── build stage ──────────────────────────────────────────────────────────────
FROM eclipse-temurin:21-jdk-alpine AS build

WORKDIR /app

# Install Maven (or use the Maven wrapper — see note below)
ARG MAVEN_VERSION=3.9.6
RUN apk add --no-cache curl && \
    curl -fsSL "https://archive.apache.org/dist/maven/maven-3/${MAVEN_VERSION}/binaries/apache-maven-${MAVEN_VERSION}-bin.tar.gz" \
    | tar -xz -C /opt && \
    ln -s /opt/apache-maven-${MAVEN_VERSION}/bin/mvn /usr/local/bin/mvn

# Copy POM first for dependency caching
COPY pom.xml ./

# Download dependencies (cached layer unless pom.xml changes)
RUN mvn dependency:go-offline -B -q

COPY src/ src/

ARG GIT_COMMIT_SHA=dev

# Run tests and build the fat JAR
RUN mvn package -B -DskipTests=false \
    -Dsha1=${GIT_COMMIT_SHA} \
    -Drevision=${GIT_COMMIT_SHA}

# ── test-export stage ─────────────────────────────────────────────────────────
# Required by the pipeline. The build invocation:
#   docker build --target test-export --output type=local,dest=./test-results
# exports the contents of this stage to the agent. JUnit XML files found under
# /test-results are published to ADO Test Results and fail the pipeline on failure.
FROM scratch AS test-export
COPY --from=build /app/target/surefire-reports/*.xml /test-results/

# ── final stage ──────────────────────────────────────────────────────────────
FROM eclipse-temurin:21-jre-alpine AS final

ARG GIT_COMMIT_SHA=dev

WORKDIR /app

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY --from=build /app/target/*.jar app.jar

ENV GIT_COMMIT_SHA=${GIT_COMMIT_SHA} \
    JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"

USER appuser

EXPOSE 8080

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar /app/app.jar"]
```

### Using the Maven wrapper (mvnw)

If your project includes the Maven wrapper (`mvnw`), you can use it instead of installing Maven manually:

```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS build

WORKDIR /app

# Copy wrapper files first
COPY mvnw .
COPY .mvn/ .mvn/
COPY pom.xml .

RUN chmod +x mvnw && ./mvnw dependency:go-offline -B -q

COPY src/ src/

ARG GIT_COMMIT_SHA=dev

RUN ./mvnw package -B -DskipTests=false -Dsha1=${GIT_COMMIT_SHA}
```

The Maven wrapper is the preferred approach — it pins the Maven version in the repository (via `.mvn/wrapper/maven-wrapper.properties`) without requiring a separate download step.

### Key points

- **Stage names matter** — the pipeline targets `test-export` and `final` by name. Do not rename these stages.
- **`FROM scratch AS test-export`** — `scratch` is the correct base for a stage whose only purpose is to export files. It adds zero bytes to the image.
- **JUnit XML path** — Maven Surefire writes JUnit XML to `target/surefire-reports/*.xml` by default. If you use a custom reports directory, update the `COPY` path in the `test-export` stage.
- **`-DskipTests=false`** — explicit, since `-DskipTests` is sometimes set in profiles or parent POMs. This ensures tests always run in the pipeline build.
- **JRE in final stage** — use `eclipse-temurin:21-jre-alpine` (not `-jdk`) in the final stage. The JDK is not needed at runtime and increases the attack surface.
- **`-XX:+UseContainerSupport`** — enables container-aware JVM heap sizing. Without it, the JVM may allocate heap based on the host's total RAM, leading to OOMKilled pods.
- **`GIT_COMMIT_SHA` build arg** — injected by the pipeline as the full 40-character Git SHA. Pass it to `BuildProperties` or embed it in `application.properties` for `/actuator/info` exposure.

---

## Multi-Module Maven Projects

For a multi-module Maven project, `pom.xml` at the root is still the entry point. Build the specific module JAR and copy it from the correct target directory:

```dockerfile
# Build only the service module (reactor build)
RUN mvn package -pl api-service -am -B -DskipTests=false

# test-export: collect test results from all modules
FROM scratch AS test-export
COPY --from=build /app/api-service/target/surefire-reports/*.xml /test-results/api-service/
COPY --from=build /app/shared/target/surefire-reports/*.xml /test-results/shared/

# final: copy the service JAR
FROM eclipse-temurin:21-jre-alpine AS final
COPY --from=build /app/api-service/target/*.jar app.jar
```

The `test-export` stage can include JUnit XML from multiple modules — the pipeline recursively scans `./test-results/**/*.xml`.

---

## What the Pipeline Does

The Spring Boot runtime step (`steps/runtime/springboot.yml`) detects Maven (when `pom.xml` is present and `gradlew` is not) and performs:

1. Sets build tool to Maven
2. Extracts version from the first `<version>` element in `pom.xml` — emits as `SPRINGBOOT_VERSION`
3. Asserts the Dockerfile contains `AS test-export` — hard fail if absent
4. Runs `docker build --target test-export --output type=local,dest=./test-results` — exports JUnit XML
5. Publishes test results via `PublishTestResults@2` — fails the pipeline if any test fails
6. Runs `docker build --target final` — builds the final image

---

## Version Tagging Summary

| Scenario | Tags pushed to ACR |
|---|---|
| No `<version>` in `pom.xml` | `<40-char-sha>`, `<branch>-<12-char-sha>` |
| Version found, non-main branch | `<40-char-sha>`, `<branch>-<12-char-sha>`, `<version>-<12-char-sha>` |
| Version found, `main` branch, tag does not exist | `<40-char-sha>`, `main-<12-char-sha>`, `<version>` |
| Version found, `main` branch, tag already exists | Pipeline fails — bump the version before merging |

---

## Common Issues

**`No build tool found` — Stage 2 fails**
`pom.xml` is not at the build context root. If your Maven module root is a subdirectory, set `buildContext`:

```yaml
parameters:
  buildContext: services/api   # pom.xml must be at services/api/pom.xml
```

**`Dockerfile does not contain a 'test-export' stage` — Stage 2 fails**
Add `FROM scratch AS test-export` to your Dockerfile. The string `AS test-export` must appear literally — the pipeline does a string search for it.

**Tests fail and block the build**
The `PublishTestResults@2` task is configured with `failTaskOnFailedTests: true`. Fix the failing tests. The JUnit XML files are visible in the ADO pipeline's Test Results tab, and the `test-results/` directory is available as a downloadable artifact.

**No JUnit XML files in `test-export`**
Verify that the `COPY --from=build` path in `test-export` matches where Surefire actually writes XML. The default is `target/surefire-reports/`. If you have customized the Surefire output directory, update the `COPY` source path.

**Wrong version extracted (parent POM version)**
If your `pom.xml` declares a `<parent>` before the project `<version>`, the regex may extract the parent version. Move the project `<version>` declaration before any `<parent>` block, or consider setting a static `revision` property:

```xml
<properties>
  <revision>2.1.0</revision>
</properties>
<version>${revision}</version>
```

Note: the pipeline regex does not expand Maven property references — use a literal version string for reliable extraction.

**Maven download fails during `dependency:go-offline`**
If your Maven settings require a corporate proxy or private repository, configure `settings.xml` and mount it in the build stage:

```dockerfile
COPY settings.xml /root/.m2/settings.xml
RUN mvn dependency:go-offline -B -q
```

Contact platform engineering if the private Maven repository requires credentials — these should be injected via BuildKit secrets, not hardcoded in `settings.xml`.

**Out of memory during Maven build**
Set Maven's JVM heap explicitly:
```dockerfile
ENV MAVEN_OPTS="-Xmx1g"
RUN mvn package -B -DskipTests=false
```
Keep the heap at or below 1 GB for typical builds on platform agents.
