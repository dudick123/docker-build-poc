# Test Cases: steps/runtime/springboot.yml

**Template:** `platform-templates/steps/runtime/springboot.yml`
**Pattern:** B — artifact built on agent; Gradle or Maven pre-build before docker build
**Stage:** Build (Stage 2) — runs before docker-build.yml
**Steps under test:** `detectBuildTool` (bash), `runGradleBuild` (bash), `runMavenBuild` (bash), `publishTestResults` (PublishTestResults@2), `extractSpringBootVersion` (bash)

---

## TC-SB-001 — Happy path (Gradle): JAR built, tests pass, version extracted

**Level:** L2
**Type:** Positive

**Precondition:**
- `gradlew` executable committed in build context
- `build.gradle` with `version = '2.3.0'`
- Dockerfile has `AS test-export` and `AS final` stages
- `--target test-export --output type=local,dest=./test-results` step produces JUnit XML at `build/test-results/test/*.xml`

**Expected result:**
- `detectBuildTool` emits `BUILD_TOOL=gradle`
- `runGradleBuild` runs `./gradlew bootJar --no-daemon --info`
- JAR is produced (log contains `BUILD SUCCESSFUL`)
- `publishTestResults` step passes; test results visible in ADO test tab
- `RUNTIME_VERSION=2.3.0` output variable emitted
- Pipeline continues to `dockerfile-lint.yml`

---

## TC-SB-002 — Gradle version extracted from build.gradle

**Level:** L1
**Type:** Positive

**Precondition:** `build.gradle`:
```groovy
version = '2.3.0'
```

**Verification (L1):** Confirm the extraction regex `grep -E "^version\s*=" build.gradle | head -1 | sed "s/.*['\"]([^'\"]+)['\"].*/\1/"` returns `2.3.0`.

---

## TC-SB-003 — Gradle version extracted from gradle.properties

**Level:** L1
**Type:** Positive

**Precondition:** `gradle.properties`:
```properties
version=1.8.5
```
No `version` line in `build.gradle`.

**Verification (L1):** When `build.gradle` has no version line, the fallback reads `gradle.properties`. `grep "^version=" gradle.properties | cut -d= -f2` returns `1.8.5`.

---

## TC-SB-004 — gradlew not executable: step marks executable then continues

**Level:** L2
**Type:** Positive — permission fix

**Precondition:** `gradlew` committed without execute permission (common when checked in from Windows).

**Expected result:** The `runGradleBuild` step runs `chmod +x ./gradlew` before invoking it. Build succeeds.

**Verification:** Build log contains `chmod +x ./gradlew` before the `./gradlew` invocation.

---

## TC-SB-005 — Dockerfile missing test-export stage: pipeline fails

**Level:** L2
**Type:** Negative

**Precondition:** `fixtures/dockerfiles/Dockerfile.springboot-no-test-export` committed as `Dockerfile`. This Dockerfile has a `final` stage but no `AS test-export` stage.

**Expected result:** The first BuildKit invocation (`--target test-export`) fails with:
```
##[error]failed to solve: target stage "test-export" could not be found
```

`runGradleBuild` step fails. Pipeline fails at Stage 2. `publishTestResults` does not run.

---

## TC-SB-006 — Gradle tests fail: pipeline fails, test results published

**Level:** L2
**Type:** Negative

**Precondition:** A test class in the fixture project that deliberately fails (e.g., `fail("intentional test failure")`). Both Dockerfile stages present.

**Expected result:**
- `runGradleBuild` fails (Gradle exits non-zero)
- `publishTestResults` still runs (configured with `continueOnError: true` or `condition: always()`)
- Test results visible in the ADO test tab showing the failed test
- Build log contains the test failure output

---

## TC-SB-007 — Happy path (Maven): JAR built via mvnw, tests pass

**Level:** L2
**Type:** Positive

**Precondition:**
- `mvnw` (Maven wrapper) present in build context; NO `gradlew` present
- `pom.xml` with `<version>1.2.3</version>`

**Expected result:**
- `detectBuildTool` emits `BUILD_TOOL=maven`
- `runMavenBuild` runs `./mvnw package -B -DskipTests=false`
- Build log contains `BUILD SUCCESS`
- `RUNTIME_VERSION=1.2.3` output variable emitted

---

## TC-SB-008 — Maven version: first version element is parent version

**Level:** L1
**Type:** Edge case

**Precondition:** `pom.xml`:
```xml
<project>
  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.0</version>
  </parent>
  <groupId>com.example</groupId>
  <artifactId>test-app</artifactId>
  <version>1.2.3</version>
</project>
```

**Verification (L1):** The extraction must read the project's own `<version>`, not the parent's. Naive XPath-first-match returns `3.2.0` (wrong). The correct extraction filters by position: the project version is the `<version>` element that is a direct child of `<project>`, not nested under `<parent>`.

Confirm the extraction returns `1.2.3`.

---

## TC-SB-009 — Neither gradlew nor mvnw present: pipeline fails

**Level:** L2
**Type:** Negative

**Precondition:** Repository has neither `gradlew` nor `mvnw`.

**Expected result:** `detectBuildTool` step fails. Build log contains:
```
##[error]No build tool found. Spring Boot runtime requires either 'gradlew' (Gradle) or 'mvnw' (Maven) wrapper in the build context.
```

---

## TC-SB-010 — Test results published as ADO artifact

**Level:** L2
**Type:** Positive — test results visibility

**Precondition:** `gradlew bootJar` run succeeds. Tests pass. JUnit XML files present at `build/test-results/test/*.xml`.

**Expected result:**
- ADO pipeline test tab shows passing tests for this run
- `PublishTestResults@2` task reports: `Results: X tests from Y test result files`
- `failTaskOnFailedTests: true` ensures that if tests report failure in XML but Gradle exit code was somehow 0, the pipeline still fails

---

## TC-SB-011 — runtimeVersion propagates to Stage 4 (cross-stage variable)

**Level:** L2/L3
**Type:** Positive — cross-stage variable propagation

**Precondition:** `build.gradle` with `version = '4.0.0'`.

**Verification:** In Stage 4, `RUNTIME_VERSION` arrives via:
```
$(stageDependencies.Build.Build.outputs['extractSpringBootVersion.RUNTIME_VERSION'])
```

Add a diagnostic `echo` step at the beginning of Stage 4 to confirm the value is `4.0.0`.
