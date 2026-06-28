## 1. springboot.yml — Build Tool Detection

- [x] 1.1 Replace the stub body of `platform-templates/steps/runtime/springboot.yml`; retain the existing `buildContext` and `dockerfilePath` parameter declarations; add `acrHost` parameter
- [x] 1.2 Add a bash step named `springbootRuntime` that tests for `$BUILD_SOURCESDIRECTORY/$BUILD_CONTEXT/gradlew` (Gradle) and `$BUILD_SOURCESDIRECTORY/$BUILD_CONTEXT/pom.xml` (Maven); set a `BUILD_TOOL` local variable to `gradle` or `maven`
- [x] 1.3 If neither `gradlew` nor `pom.xml` is found, emit `##vso[task.logissue type=error]No build tool found in <resolved-path>. Expected 'gradlew' (Gradle) or 'pom.xml' (Maven) for runtimeType: springboot.` and `exit 1`

## 2. springboot.yml — Version Extraction

- [x] 2.1 When `BUILD_TOOL=gradle`: extract version from `build.gradle` using `grep -E "^version\s*=" | head -1 | sed "s/.*=\s*['\"]//;s/['\"].*//"`; if no match, fall back to `gradle.properties` using `grep -E "^version\s*=" | head -1 | sed "s/.*=\s*//;s/[[:space:]]*$//"`
- [x] 2.2 When `BUILD_TOOL=maven`: extract version from `pom.xml` using `grep -oP '(?<=<version>)[^<]+' | head -1`
- [x] 2.3 Store extracted value in `SPRING_VER` (empty string if no match); emit `echo "##vso[task.setvariable variable=SPRINGBOOT_VERSION;isOutput=true]$SPRING_VER"`

## 3. springboot.yml — Dockerfile test-export Stage Assertion

- [x] 3.1 After version extraction, check that the resolved Dockerfile (`$BUILD_SOURCESDIRECTORY/$BUILD_CONTEXT/$DOCKERFILE_PATH`) contains the string `AS test-export` using `grep -q 'AS test-export'`
- [x] 3.2 If absent, emit `##vso[task.logissue type=error]Dockerfile at <path> does not contain a 'test-export' stage. Spring Boot builds require 'FROM scratch AS test-export' to extract test results. See platform reference Dockerfile at docs/reference-dockerfiles/springboot.Dockerfile.` and `exit 1`

## 4. springboot.yml — Two-Invocation BuildKit Sequence

- [x] 4.1 Add a bash step (after `springbootRuntime`) that runs `DOCKER_BUILDKIT=1 docker build --target test-export --output type=local,dest=./test-results -f "$BUILD_SOURCESDIRECTORY/$BUILD_CONTEXT/$DOCKERFILE_PATH" "$BUILD_SOURCESDIRECTORY/$BUILD_CONTEXT"`; name this step `extractTests`; pass `BUILD_CONTEXT`, `DOCKERFILE_PATH` via `env:` block
- [x] 4.2 Add an ADO `PublishTestResults` task step after `extractTests`: set `testResultsFormat: JUnit`, `testResultsFiles: '**/*.xml'`, `searchFolder: '$(System.DefaultWorkingDirectory)/test-results'`, `failTaskOnFailedTests: true`, `displayName: 'Publish Spring Boot Test Results'`
- [x] 4.3 Add a bash step (after `PublishTestResults`) that runs `DOCKER_BUILDKIT=1 docker build --target final -f "$BUILD_SOURCESDIRECTORY/$BUILD_CONTEXT/$DOCKERFILE_PATH" -t "$LOCAL_IMAGE_REF" "$BUILD_SOURCESDIRECTORY/$BUILD_CONTEXT"`; name this step `buildFinalImage`; construct `LOCAL_IMAGE_REF` as `$TENANT_NAME/$APP_NAME:pipeline-$BUILD_BUILDID`; pass `BUILD_CONTEXT`, `DOCKERFILE_PATH`, `TENANT_NAME`, `APP_NAME` via `env:` block

## 5. springboot.yml — Security Pattern Confirmation

- [x] 5.1 Confirm no `${{ parameters.xxx }}` expressions appear in any bash script body in `springboot.yml`; all parameter values are in the `env:` block only

## 6. springboot.yml — YAML Validation

- [x] 6.1 Run `python3 -c "import yaml; yaml.safe_load(open('platform-templates/steps/runtime/springboot.yml'))"` — confirm no parse errors
- [x] 6.2 Confirm the first step is named `springbootRuntime` and `isOutput=true` is present on the `SPRINGBOOT_VERSION` emit
- [x] 6.3 Confirm the `PublishTestResults` task step is present with `failTaskOnFailedTests: true`

## 7. ADO End-to-End Validation

- [ ] 7.1 Queue a `dryRun=true` pipeline with `runtimeType: springboot` against a repo with `gradlew`, `build.gradle` containing `version = '1.0.0'`, and a Dockerfile with `AS test-export`; confirm `SPRINGBOOT_VERSION=1.0.0` in step log and test results appear in ADO Tests tab
- [ ] 7.2 Queue a `dryRun=true` pipeline with `runtimeType: springboot` against a repo with no `gradlew` and no `pom.xml`; confirm Stage 1 fails with the path-specific error message
- [ ] 7.3 Queue a `dryRun=true` pipeline with `runtimeType: springboot` against a Dockerfile missing `AS test-export`; confirm Stage 2 fails with the reference Dockerfile message
- [ ] 7.4 Queue a `dryRun=true` pipeline with `runtimeType: springboot` against a repo whose tests fail; confirm pipeline is blocked at `PublishTestResults` and the final image is not built
- [ ] 7.5 Queue a `dryRun=true` pipeline with `runtimeType: springboot` using a Maven project (`pom.xml`, no `gradlew`); confirm `SPRINGBOOT_VERSION` is extracted from `pom.xml`
