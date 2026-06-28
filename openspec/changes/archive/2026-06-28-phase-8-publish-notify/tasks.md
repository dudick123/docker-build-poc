## 1. container-build-v2.yml — Publish Stage Parameters

- [x] 1.1 Update the `Publish` stage's `sbom-sign-publish.yml` template call to pass `acrHost: $(stageDependencies.Setup.Setup.outputs['resolveTools.ACR_HOST'])`
- [x] 1.2 Pass `manifestDigest: $(stageDependencies.SignAndAttest.SignAndAttest.outputs['signAttest.MANIFEST_DIGEST'])` to the Publish template call
- [x] 1.3 Add compile-time `${{ if eq(parameters.runtimeType, ...) }}` dispatch for each runtime to select the correct version output variable from Stage 2 and pass it as `runtimeVersion` to the Publish template call; cover all five runtimes: `go` → `stageDependencies.Build.Build.outputs['goRuntime.GO_VERSION']`, `python` → `pythonRuntime.PYTHON_VERSION`, `springboot` → `springbootRuntime.SPRINGBOOT_VERSION`, `angular` → `angularRuntime.ANGULAR_VERSION`, `react` → `reactRuntime.REACT_VERSION`
- [x] 1.4 Run YAML validation on `platform-templates/container-build-v2.yml` after changes

## 2. container-build-v2.yml — Notify Stage Parameters

- [x] 2.1 Update the `Notify` stage's `sbom-sign-publish.yml` template call to pass `acrHost: $(stageDependencies.Setup.Setup.outputs['resolveTools.ACR_HOST'])`
- [x] 2.2 Pass `manifestDigest: $(stageDependencies.SignAndAttest.SignAndAttest.outputs['signAttest.MANIFEST_DIGEST'])` to the Notify template call
- [x] 2.3 Pass `imageRef: $(stageDependencies.Publish.Publish.outputs['publish.IMAGE_REF'])` to the Notify template call
- [x] 2.4 Run YAML validation on `platform-templates/container-build-v2.yml` after changes

## 3. sbom-sign-publish.yml — New Parameters

- [x] 3.1 Add parameters to `platform-templates/steps/sbom-sign-publish.yml`: `manifestDigest` (string, default empty), `runtimeVersion` (string, default empty), `imageRef` (string, default empty)
- [x] 3.2 Confirm all existing parameters (`phase`, `tenantName`, `appName`, `dryRun`, `acrHost`, `imageDigest`, `syftVersion`, `cosignVersion`) are retained unchanged

## 4. sbom-sign-publish.yml — publish phase: latest-tag assertion

- [x] 4.1 In the `publish` phase block (conditioned on `${{ if eq(parameters.phase, 'publish') }}`), add a bash step named `assertTags` that computes the full-SHA, alias, and version tag strings and asserts none equals `latest`; emit `##vso[task.logissue type=error]` and `exit 1` if any tag equals `latest`; pass `TENANT_NAME`, `APP_NAME`, `ACR_HOST`, `RUNTIME_VERSION` via `env:` block

## 5. sbom-sign-publish.yml — publish phase: tag push and digest verification

- [x] 5.1 Add a bash step named `publish` (in the `publish` phase) that:
  - Tags and pushes full-SHA primary tag: `docker tag "$LOCAL_IMAGE_REF" "$ACR_IMAGE_REF:$BUILD_SOURCEVERSION" && docker push "$ACR_IMAGE_REF:$BUILD_SOURCEVERSION"`
  - Captures ACR manifest digest via `docker inspect --format='{{index .RepoDigests 0}}'` and extracts `sha256:...`
  - Compares captured digest to `$MANIFEST_DIGEST`; fails step with error if mismatched
  - Sanitizes branch name (replace `/` with `-`) and pushes alias tag: `"$ACR_IMAGE_REF:$BRANCH_NAME-$SHORT_SHA"`
  - If `RUNTIME_VERSION` is non-empty and branch is main: check tag existence via `docker manifest inspect "$ACR_IMAGE_REF:$RUNTIME_VERSION"`; fail step with error if tag exists; push if absent
  - If `RUNTIME_VERSION` is non-empty and branch is non-main: push `"$ACR_IMAGE_REF:$RUNTIME_VERSION-$SHORT_SHA"` unconditionally
  - If `RUNTIME_VERSION` is empty: skip version tag with log message
  - Emits `echo "##vso[task.setvariable variable=IMAGE_REF;isOutput=true]$ACR_IMAGE_REF@$MANIFEST_DIGEST"`
  - Pass `ACR_HOST`, `TENANT_NAME`, `APP_NAME`, `MANIFEST_DIGEST`, `RUNTIME_VERSION` via `env:` block
- [x] 5.2 Confirm the step is named `publish` and `isOutput=true` is present on the `IMAGE_REF` emit

## 6. sbom-sign-publish.yml — publish phase: provenance artifact

- [x] 6.1 After the `publish` bash step, add a bash step that writes `provenance.json` containing: `imageRef`, `manifestDigest`, `tags` (array of pushed tags), `sbomArtifact` (`sbom-<tenantName>-<appName>`), `cosignStatus` (`signed`), `pipelineRunId` (`$BUILD_BUILDID`), `acrRepository` (`$ACR_HOST/$TENANT_NAME/$APP_NAME`), `gitCommit` (`$BUILD_SOURCEVERSION`); pass all values via `env:` block
- [x] 6.2 Add a `PublishPipelineArtifact@1` task that publishes `provenance.json` as artifact `provenance-${{ parameters.tenantName }}-${{ parameters.appName }}`

## 7. sbom-sign-publish.yml — notify phase: PR comment

- [x] 7.1 Replace the notify phase stub with a bash step named `postPrComment` that:
  - Checks `$BUILD_REASON`; if not `PullRequest`, logs "Skipping PR comment: not a pull request build" and exits zero
  - Constructs a Markdown comment body including: build status, runtimeType, image digest, ACR ref, tags, SBOM artifact name, Cosign status, security scan advisory note
  - Posts via ADO REST API using `curl` with `System.AccessToken` passed via `env:`
  - Passes all values via `env:` block

## 8. sbom-sign-publish.yml — notify phase: ADO build tag

- [x] 8.1 In the `notify` phase, add a bash step named `setBuildTag`: emit `echo "##vso[build.addbuildtag]$MANIFEST_DIGEST"` if `MANIFEST_DIGEST` is non-empty; if empty (dry run), emit `echo "##vso[build.addbuildtag]dryrun-$BUILD_BUILDID"`; pass `MANIFEST_DIGEST` via `env:`

## 9. sbom-sign-publish.yml — notify phase: Teams webhook

- [x] 9.1 Add a bash step named `notifyTeams` that reads `$TEAMS_WEBHOOK_URL` from env; if empty, logs warning and exits zero; if set, constructs an Adaptive Card JSON payload using `jq` and POSTs to the webhook URL via `curl`; passes `TEAMS_WEBHOOK_URL: $(TEAMS_WEBHOOK_URL)`, `IMAGE_REF`, `RUNTIME_TYPE`, `BUILD_BUILDID` via `env:`

## 10. Security Pattern Confirmation

- [x] 10.1 Confirm all `${{ parameters.xxx }}` expressions in new publish and notify steps are only in `env:` blocks or `${{ if ... }}` conditions — none inline in bash bodies

## 11. YAML Validation

- [x] 11.1 Run `python3 -c "import yaml; yaml.safe_load(open('platform-templates/steps/sbom-sign-publish.yml'))"` — confirm no parse errors
- [x] 11.2 Run `python3 -c "import yaml; yaml.safe_load(open('platform-templates/container-build-v2.yml'))"` — confirm no parse errors
- [x] 11.3 Confirm `publish` step is named `publish` and `isOutput=true` is on `IMAGE_REF`
- [x] 11.4 Confirm publish and notify stubs are fully replaced (no remaining `STUB:` echoes)

## 12. ADO End-to-End Validation

- [ ] 12.1 Queue a `dryRun=false` pipeline with `runtimeType: go` against a simple Go repo; confirm Stage 4 pushes full-SHA, alias, and (if VERSION file present) version tags to ACR
- [ ] 12.2 Confirm digest verification passes (no mismatch error) after full-SHA push
- [ ] 12.3 Confirm `latest` tag is not present in ACR after the pipeline run
- [ ] 12.4 Confirm `IMAGE_REF` output variable resolves in Stage 5 via `stageDependencies.Publish.Publish.outputs['publish.IMAGE_REF']`
- [ ] 12.5 Confirm `provenance-<tenant>-<app>` artifact is present in pipeline run artifacts
- [ ] 12.6 Confirm PR comment appears on the triggering PR with correct digest and tag list
- [ ] 12.7 Confirm Teams notification is received when webhook is configured
- [ ] 12.8 Confirm ADO pipeline run build tag is set to the manifest digest
- [ ] 12.9 Queue a `dryRun=true` pipeline; confirm Stages 3 and 4 are skipped; confirm PR comment says "dry run, no image pushed"
- [ ] 12.10 Attempt to run pipeline when version tag already exists on main; confirm pipeline fails before push with bump-version error message
