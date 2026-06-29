# syntax=docker/dockerfile:1

# ============================================================================
# STAGE 1: build
#
# Uses the Maven wrapper (mvnw) to compile, test, and package the application
# inside the container. No JDK or Maven is needed on the pipeline agent.
#
# Using the Maven wrapper is preferred over installing Maven in the Dockerfile:
# - Pins the exact Maven version in .mvn/wrapper/maven-wrapper.properties
# - Eliminates a download step (wrapper downloads Maven on first run,
#   then caches it in the Docker layer)
# - Consistent with local developer workflow
# ============================================================================
FROM eclipse-temurin:21-jdk-alpine AS build

WORKDIR /app

# ── Maven wrapper setup ───────────────────────────────────────────────────────
# Copy wrapper files before the source to maximize Docker layer caching.
# mvnw + .mvn/ change only when the Maven version is bumped.
COPY mvnw .
COPY .mvn/ .mvn/
RUN chmod +x mvnw

# ── Dependency caching ───────────────────────────────────────────────────────
# Copy the POM file and download all dependencies as a separate layer.
# This layer is reused across builds as long as pom.xml doesn't change.
# 'dependency:go-offline' resolves and caches the entire dependency tree
# so the subsequent package step works without internet access.
COPY pom.xml ./
RUN ./mvnw dependency:go-offline -B -q

# ── Source copy ──────────────────────────────────────────────────────────────
COPY src/ src/

# GIT_COMMIT_SHA: injected by the platform pipeline.
ARG GIT_COMMIT_SHA=dev

# ── Build + test ─────────────────────────────────────────────────────────────
# 'package' compiles, runs all Surefire tests, and produces the fat JAR
# via the Spring Boot Maven plugin.
#
# -B (batch mode): disables interactive progress output; required for CI
# -DskipTests=false: explicit — prevents any profile or parent POM from
#   suppressing tests. Tests MUST run.
# -Dsha1: custom property — use in application.properties for /actuator/info
#   (e.g., info.git.sha=${sha1})
#
# Surefire writes JUnit XML to: target/surefire-reports/*.xml
RUN ./mvnw package -B \
    -DskipTests=false \
    -Dsha1="${GIT_COMMIT_SHA}" \
    -Drevision="${GIT_COMMIT_SHA}"

# ============================================================================
# STAGE 2: test-export
#
# *** THIS STAGE IS REQUIRED BY THE PLATFORM PIPELINE ***
#
# The pipeline runs:
#   docker build --target test-export --output type=local,dest=./test-results
#
# This exports the stage filesystem to ./test-results on the pipeline agent.
# PublishTestResults@2 then scans ./test-results/**/*.xml and publishes to ADO.
# If any test is marked as failed in the XML, the pipeline fails here and the
# final image build does not run.
#
# Maven Surefire default output path: target/surefire-reports/*.xml
# Failsafe (integration tests) default: target/failsafe-reports/*.xml
#
# If you collect both unit and integration test results, add a second COPY:
#   COPY --from=build /app/target/failsafe-reports/*.xml /test-results/integration/
# ============================================================================
FROM scratch AS test-export
COPY --from=build /app/target/surefire-reports/*.xml /test-results/

# ============================================================================
# STAGE 3: final
#
# Minimal JRE runtime image. The JDK (compiler, javac, jshell) is not needed
# at runtime and should never be in a production image.
#
# Using Spring Boot's layered JAR feature (enabled by default in Spring Boot 3)
# is an alternative to copying the fat JAR — it produces better Docker layer
# caching by separating dependencies from application classes. See README for
# the layered JAR pattern.
# ============================================================================
FROM eclipse-temurin:21-jre-alpine AS final

ARG GIT_COMMIT_SHA=dev

WORKDIR /app

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy the repackaged fat JAR (produced by spring-boot:repackage, which is
# bound to the 'package' phase automatically by the Spring Boot Maven plugin).
# The wildcard handles version suffixes in the artifact filename.
COPY --from=build /app/target/*.jar app.jar

ENV GIT_COMMIT_SHA=${GIT_COMMIT_SHA} \
    JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"

USER appuser

EXPOSE 8080

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar /app/app.jar"]
