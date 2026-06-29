# Example: Spring Boot (Maven)

A complete, annotated example of the platform container build pipeline for a Spring Boot application built with Maven. All files can be copied into an application repository root and customized.

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
  ├── dockerfile-lint.yml  → Hadolint
  ├── docker-build.yml     → BuildKit build (full image, default target)
  └── runtime/springboot.yml:
        1. Detects pom.xml (no gradlew) → build tool is Maven
        2. Extracts version from first <version> element in pom.xml
        3. Asserts Dockerfile contains 'AS test-export'
        4. docker build --target test-export --output type=local,dest=./test-results
        5. PublishTestResults@2  ← fails pipeline if any test fails
        6. docker build --target final  ← near-instant (BuildKit cache reuse)
```

---

## Build Tool Detection

The pipeline checks for build tools in this order:

1. `gradlew` present → **Gradle** (see springboot-gradle example)
2. `pom.xml` present, no `gradlew` → **Maven**
3. Neither → **Stage 2 hard failure**

If your repository has both `gradlew` and `pom.xml` (e.g., a multi-module project with both), the pipeline will select Gradle. To force Maven, remove `gradlew` or move it out of the `buildContext`.

---

## Required Repository Structure

```
inventory-service/
  mvnw                ← Maven wrapper (required if using mvnw approach)
  .mvn/
    wrapper/
      maven-wrapper.jar
      maven-wrapper.properties
  pom.xml             ← version extracted from first <version> element
  src/
    main/java/...
    test/java/...
  Dockerfile          ← required: must contain 'AS test-export' and 'AS final'
  azure-pipelines.yml
```

---

## Version Tagging

The pipeline reads the first `<version>` element from `pom.xml`.

```xml
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>inventory-service</artifactId>
  <version>1.8.0</version>    <!-- ← extracted by the pipeline -->
  ...
</project>
```

### Parent POM version ordering

The pipeline uses a simple regex that matches the **first** `<version>` element in the file. If your POM declares a `<parent>` block before the project `<version>`, the parent's version will be extracted instead:

```xml
<!-- This layout causes the WRONG version to be extracted -->
<project>
  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.0</version>    <!-- ← extracted instead of your version -->
  </parent>
  <version>1.8.0</version>      <!-- ← not reached by the regex -->
</project>
```

**Fix:** Move your project `<version>` before the `<parent>` block, or use the Maven `${revision}` property pattern:

```xml
<project>
  <properties>
    <revision>1.8.0</revision>   <!-- ← regex does NOT expand properties -->
  </properties>
  <version>${revision}</version>
```

Note: the pipeline regex does not expand `${}` references. If you use `${revision}`, the version tag will be skipped because the literal string `${revision}` is not a valid semver. Use a static version string in the `<version>` element for reliable extraction.

| Branch | Tags pushed to ACR |
|---|---|
| Feature branch, version found | `<sha>`, `<branch>-<short-sha>`, `1.8.0-<short-sha>` |
| `main`, tag `1.8.0` is new | `<sha>`, `main-<short-sha>`, `1.8.0` |
| `main`, tag `1.8.0` exists | **Pipeline fails** — bump the version before merging |
| No version extracted | `<sha>`, `<branch>-<short-sha>` |

---

## Customizing the Dockerfile

### Without the Maven wrapper

If you prefer to install Maven in the Dockerfile rather than use `mvnw`:

```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS build

ARG MAVEN_VERSION=3.9.6
RUN apk add --no-cache curl && \
    curl -fsSL "https://archive.apache.org/dist/maven/maven-3/${MAVEN_VERSION}/binaries/apache-maven-${MAVEN_VERSION}-bin.tar.gz" \
    | tar -xz -C /opt && \
    ln -s /opt/apache-maven-${MAVEN_VERSION}/bin/mvn /usr/local/bin/mvn

COPY pom.xml ./
RUN mvn dependency:go-offline -B -q

COPY src/ src/
ARG GIT_COMMIT_SHA=dev
RUN mvn package -B -DskipTests=false -Dsha1="${GIT_COMMIT_SHA}"
```

### Spring Boot layered JAR (optimized Docker caching)

Spring Boot 3 supports extracting the fat JAR into layers for better Docker layer caching. Unchanged dependency layers are reused across builds even when application code changes:

```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS build
# ... build stage unchanged ...

FROM eclipse-temurin:21-jdk-alpine AS extract
WORKDIR /extract
COPY --from=build /app/target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:21-jre-alpine AS final
ARG GIT_COMMIT_SHA=dev
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY --from=extract /extract/dependencies/ ./
COPY --from=extract /extract/spring-boot-loader/ ./
COPY --from=extract /extract/snapshot-dependencies/ ./
COPY --from=extract /extract/application/ ./

ENV GIT_COMMIT_SHA=${GIT_COMMIT_SHA} \
    JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"
USER appuser
EXPOSE 8080
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS org.springframework.boot.loader.launch.JarLauncher"]
```

Add the `test-export` stage between `build` and `extract`:

```dockerfile
FROM scratch AS test-export
COPY --from=build /app/target/surefire-reports/*.xml /test-results/
```

### Including integration test results

If you run both Surefire (unit) and Failsafe (integration) tests:

```dockerfile
FROM scratch AS test-export
COPY --from=build /app/target/surefire-reports/*.xml /test-results/unit/
COPY --from=build /app/target/failsafe-reports/*.xml /test-results/integration/
```

The pipeline scans `./test-results/**/*.xml` recursively.

### Multi-module Maven project

Target the specific service module and copy from its subdirectory:

```dockerfile
RUN ./mvnw package -pl inventory-service -am -B -DskipTests=false

FROM scratch AS test-export
COPY --from=build /app/inventory-service/target/surefire-reports/*.xml /test-results/

FROM eclipse-temurin:21-jre-alpine AS final
COPY --from=build /app/inventory-service/target/*.jar app.jar
```

---

## What the Pipeline Produces

For a successful non-dry-run run on `main` with `<version>1.8.0</version>` in `pom.xml`:

### ACR tags (at `<platform-acr>/commerce/inventory-service`)

| Tag | Use |
|---|---|
| `<40-char-sha>` | Kustomize manifests (immutable) |
| `main-<12-char-sha>` | Human navigation |
| `1.8.0` | Release version (main branch only) |

### ADO Test Results tab

Surefire XML files exported from `test-export` are published and visible in the **Tests** tab. Failed tests block the pipeline.

### Pipeline artifacts

| Artifact | Contents |
|---|---|
| `sbom-commerce-inventory-service` | CycloneDX JSON SBOM |
| `provenance-commerce-inventory-service` | JSON provenance record |

---

## Troubleshooting

**`No build tool found` — Stage 2 fails**
Neither `gradlew` nor `pom.xml` was found at the build context root. If `buildContext` is set to a subdirectory, `pom.xml` must be at that directory's root.

**`Dockerfile does not contain a 'test-export' stage`**
The string `AS test-export` is missing from the Dockerfile. Add `FROM scratch AS test-export` exactly as shown. The pipeline does a literal string search.

**Wrong version extracted (parent POM version)**
Reorder the `pom.xml` so the project `<version>` appears before the `<parent>` block, or switch to a static version string instead of `${revision}`. See [Version Tagging](#version-tagging) above.

**`dependency:go-offline` fails**
The Maven wrapper is downloading Maven for the first time and needs internet access, or the `settings.xml` points to a private repository requiring credentials. If your organization uses a private Maven repository, add a `settings.xml` COPY step and configure the repository URL. Contact platform engineering for credential injection via BuildKit secrets.

**`target/*.jar` wildcard matches multiple JARs**
The Spring Boot Maven plugin produces one repackaged JAR and one original JAR (suffix `-original.jar`). If both are in `target/`, specify the exact filename:

```dockerfile
COPY --from=build /app/target/inventory-service-*.jar app.jar
```

Or exclude the original JAR:

```dockerfile
COPY --from=build /app/target/inventory-service-[0-9]*.jar app.jar
```

**Tests fail intermittently (flaky tests)**
Fix the flaky tests — `failTaskOnFailedTests: true` is enforced and cannot be bypassed at the pipeline level. ADO Test Results shows flaky test history over time.
