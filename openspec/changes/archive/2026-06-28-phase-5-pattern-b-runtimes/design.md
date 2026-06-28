## Context

`steps/runtime/springboot.yml` is a Phase 1 stub executing in the Build stage's single job, after `dockerfile-lint.yml` and `docker-build.yml` would otherwise run. However, Spring Boot changes the normal Build stage flow: the two-invocation BuildKit sequence in `springboot.yml` supersedes the single `docker build` in `docker-build.yml`. The base template dispatches to `springboot.yml` when `runtimeType: springboot`; it receives `buildContext`, `dockerfilePath`, and `acrHost` as parameters.

The architectural decision for Phase 5 is **Pattern A** — all Java/Gradle/Maven compilation happens inside Docker. The agent only runs `docker build`. The `test-export` stage in the tenant's Dockerfile is a BuildKit target that produces JUnit XML, allowing ADO's `PublishTestResults` task to surface results in the Tests tab without requiring a Java SDK on the agent.

Stakeholders: platform engineering (owns templates), Spring Boot tenant teams (receive error messages, test results, version tag behavior).

## Goals / Non-Goals

**Goals:**
- Detect build tool (Gradle wrapper or Maven) from project files; fail Stage 1 if neither found
- Assert that the tenant's Dockerfile contains a `test-export` stage; fail with a clear message if absent
- Execute two-invocation BuildKit: extract test results, publish to ADO, gate final build on test passage
- Extract `SPRINGBOOT_VERSION` from project metadata for downstream version tagging
- No Java/Gradle/Maven toolchain required on agent — Pattern A

**Non-Goals:**
- Running `./gradlew`, `mvn`, or any Java toolchain on the agent
- Validating the content of the `test-export` stage beyond existence
- Configuring Maven/Gradle dependency caching beyond what BuildKit layer cache provides
- Supporting Kotlin DSL (`build.gradle.kts`) version extraction (only Groovy DSL and `gradle.properties` in scope)

## Decisions

### Decision 1: Gradle wrapper takes priority over Maven for build tool detection

Detection order: (1) `gradlew` present in `buildContext` → Gradle, (2) `pom.xml` present in `buildContext` → Maven, (3) neither → hard fail Stage 1. Polyglot repos with both `gradlew` and `pom.xml` are treated as Gradle projects.

**Rationale:** `gradlew` presence is a more authoritative signal for Gradle projects than `pom.xml` is for Maven — a Gradle project building a Spring Boot app commonly includes `pom.xml` for BOM management. Failing when neither is found makes misconfiguration immediately visible rather than letting a wrong `docker build` invocation produce a cryptic failure.

### Decision 2: test-export stage assertion via grep, not full Dockerfile parse

The Dockerfile is checked with `grep -q 'AS test-export'` before the first `docker build` invocation. Hard fail if absent with message: `"Dockerfile at <path> does not contain a 'test-export' stage. Spring Boot builds require 'FROM scratch AS test-export' to extract test results. See platform reference Dockerfile at docs/reference-dockerfiles/springboot.Dockerfile."`.

**Rationale:** A full Dockerfile parser is not available as a standalone binary on the agent. `grep` is sufficient to detect the stage name string. False negatives (e.g., stage in a comment) are acceptable — Hadolint has already linted the Dockerfile in the prior step so structural correctness is already gated.

### Decision 3: Two-invocation BuildKit sequence gated by ADO PublishTestResults

Invocation 1: `DOCKER_BUILDKIT=1 docker build --target test-export --output type=local,dest=./test-results -f <dockerfilePath> <buildContext>`. Invocation 2 runs only after the `PublishTestResults` ADO task succeeds. ADO's `PublishTestResults` task has a `failTaskOnFailedTests: true` option which causes the task to fail the step when test failures are detected — this is the gate.

**Rationale:** Gating on the ADO task (not a script exit code) produces proper ADO test result integration: results appear in the Tests tab and the failure is attributed to the correct step. The alternative (parsing XML in bash) would duplicate ADO's built-in capability and produce inferior UX.

### Decision 4: Version extraction uses grep/sed with no Java/Gradle/Maven invocation

- **Maven:** `grep -oP '(?<=<version>)[^<]+' pom.xml | head -1` — extracts the first `<version>` element value (typically the project version when it is the first occurrence in standard Maven POM layout)
- **Gradle (build.gradle):** `grep -E "^version\s*=" build.gradle | head -1 | sed "s/.*=\s*['\"]//;s/['\"].*//"`
- **Gradle (gradle.properties):** fallback if `build.gradle` yields no match: `grep -E "^version\s*=" gradle.properties | head -1 | sed "s/.*=\s*//;s/[[:space:]]*$//"`

If no version is extractable, `SPRINGBOOT_VERSION` is emitted as empty string; Phase 8 skips the version tag.

**Rationale:** Pattern A prohibits invoking Maven or Gradle on the agent. Regex extraction is sufficient for standard project layouts. Edge cases (e.g., `version` assigned from a variable in Groovy DSL) yield empty string and skip version tag — the correct fallback.

### Decision 5: All parameter values pass through the env: block

Consistent with the Phase 2 security pattern. No `${{ parameters.xxx }}` expressions appear inside bash script bodies. `BUILD_CONTEXT`, `DOCKERFILE_PATH`, and `ACR_HOST` are mapped to environment variables in the `env:` block.

**Rationale:** Prevents template injection if a tenant supplies shell metacharacters in parameter values.

## Risks / Trade-offs

- **Maven POM first `<version>` assumption** — If a multi-module `pom.xml` declares a parent version before the project version, `head -1` may extract the wrong version. Mitigation: document that the root POM must declare `<version>` as its first version element; tenants with complex multi-module layouts should use `gradle.properties` instead.
- **Gradle Kotlin DSL not supported** — `build.gradle.kts` uses `version = "..."` syntax with double quotes but the file extension is `.kts`; the Groovy DSL grep targets `.gradle` only. Mitigation: Kotlin DSL projects emit empty `SPRINGBOOT_VERSION` and skip version tag; no pipeline failure. Document as a known limitation.
- **BuildKit `--output` requires BuildKit 0.8+** — The `type=local` output mode is unavailable on very old Docker versions. Mitigation: `platform-tool-versions` already pins `DOCKER_BUILDKIT_VERSION`; this version must be ≥ 0.8. No action needed in template.
- **Test results path coupling** — The template assumes test results land at `./test-results/`. If the tenant's `test-export` stage copies to a different path, `PublishTestResults` will report no results. Mitigation: document that `test-export` must `COPY --from=test /app/build/test-results /` (platform reference Dockerfile enforces this).

## Open Questions

None. All decisions align with Phase 5 scope in the implementation plan.
