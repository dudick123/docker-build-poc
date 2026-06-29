# syntax=docker/dockerfile:1

# ============================================================================
# STAGE 1: build
#
# Compiles the application and runs all tests inside a JDK container.
# The Gradle wrapper (gradlew) runs inside the container — no JDK or Gradle
# is required on the pipeline agent.
#
# Using eclipse-temurin (formerly AdoptOpenJDK / Eclipse Adoptium):
# - The most widely used, fully open-source JDK distribution
# - Alpine variant keeps the image size manageable
# - Always use an LTS JDK version (11, 17, 21) in production
# ============================================================================
FROM eclipse-temurin:21-jdk-alpine AS build

WORKDIR /app

# ── Dependency caching ───────────────────────────────────────────────────────
# Copy the Gradle wrapper and build descriptor files before the source.
# Docker layer caching means 'gradlew dependencies' is only re-run when
# build.gradle or gradle.properties changes, not on every source change.
# This is the most important optimization for Gradle build times in CI.
COPY gradlew gradlew
COPY gradle/ gradle/
COPY build.gradle settings.gradle gradle.properties ./

# Make the wrapper executable and download all dependencies.
# --no-daemon: Gradle daemon processes are not appropriate for containerized
#   builds. The daemon assumes a persistent JVM but containers are ephemeral.
# --quiet: reduces log noise; change to --info for debugging.
RUN chmod +x gradlew && ./gradlew dependencies --no-daemon --quiet

# ── Source copy ──────────────────────────────────────────────────────────────
COPY src/ src/

# GIT_COMMIT_SHA: injected by the platform pipeline as the full 40-char SHA.
# Pass it to Spring Boot's build properties for /actuator/info exposure.
ARG GIT_COMMIT_SHA=dev

# ── Build + test ─────────────────────────────────────────────────────────────
# 'bootJar' compiles, runs all tests, and produces the executable fat JAR.
# If any test fails, this step fails and the Dockerfile build exits non-zero,
# which will also fail the pipeline — but the 'test-export' stage below
# provides the cleaner path for test failure reporting.
#
# -Dapp.build.gitSha: custom Gradle property — map this to a Spring Boot
#   info.git.commit value in your application.properties.
# -Dorg.gradle.jvmargs: caps the Gradle JVM heap. Platform agents have
#   limited memory — 1g is a safe default for most services.
RUN ./gradlew bootJar --no-daemon \
    -Dapp.build.gitSha="${GIT_COMMIT_SHA}" \
    -Dorg.gradle.jvmargs="-Xmx1g"

# ============================================================================
# STAGE 2: test-export
#
# *** THIS STAGE IS REQUIRED BY THE PLATFORM PIPELINE ***
#
# The pipeline runs:
#   docker build --target test-export --output type=local,dest=./test-results
#
# This exports the entire filesystem of this stage to ./test-results on the
# agent. The pipeline then runs PublishTestResults@2 over the exported
# directory, publishing JUnit XML to ADO and failing the pipeline if any
# test is marked as failed.
#
# 'FROM scratch' is the correct base for this stage — it has no content of
# its own. Everything in the exported filesystem comes from COPY statements.
#
# The COPY source path must match where Gradle writes JUnit XML:
#   Default: build/test-results/test/*.xml
#   Custom:  if you've changed the test report directory in build.gradle,
#            update the source path here accordingly.
# ============================================================================
FROM scratch AS test-export
COPY --from=build /app/build/test-results/test/*.xml /test-results/

# ============================================================================
# STAGE 3: final
#
# The production image. Uses the JRE (not JDK) for a smaller footprint and
# reduced attack surface. The JDK compiler tooling is never needed at runtime.
#
# Key JVM flags for AKS:
# -XX:+UseContainerSupport  — tells the JVM to respect container memory limits
#   rather than using the host machine's total RAM for heap sizing. Without
#   this, pods are frequently OOMKilled.
# -XX:MaxRAMPercentage=75.0 — caps heap at 75% of the container's memory limit,
#   leaving headroom for off-heap memory (thread stacks, metaspace, JIT cache).
# ============================================================================
FROM eclipse-temurin:21-jre-alpine AS final

ARG GIT_COMMIT_SHA=dev

WORKDIR /app

# Create a dedicated non-root user for the application process.
# Running as root inside a container is a security concern — if the
# application is compromised, root access within the container is immediate.
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy the fat JAR built by bootJar. The wildcard handles version suffixes
# in the filename (e.g., order-service-1.4.2.jar → app.jar).
# If you have multiple JARs in the output directory, use an explicit name:
#   COPY --from=build /app/build/libs/order-service-*.jar app.jar
COPY --from=build /app/build/libs/*.jar app.jar

ENV GIT_COMMIT_SHA=${GIT_COMMIT_SHA} \
    JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"

USER appuser

EXPOSE 8080

# Use 'sh -c' to allow JAVA_OPTS variable expansion in the exec form.
# Note: JAVA_OPTS is not expanded by the JVM directly from the environment
# in all versions; use JAVA_TOOL_OPTIONS instead if needed:
#   ENV JAVA_TOOL_OPTIONS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"
#   ENTRYPOINT ["java", "-jar", "/app/app.jar"]
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar /app/app.jar"]
