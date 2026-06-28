## 1. Repository Structure

- [x] 1.1 Create `platform-templates/` directory at the repo root
- [x] 1.2 Create `platform-templates/steps/` subdirectory
- [x] 1.3 Create `platform-templates/steps/runtime/` subdirectory

## 2. Base Template — Parameter Block

- [x] 2.1 Create `platform-templates/container-build-v2.yml` with the `parameters:` block declaring all six parameters: `tenantName` (string, required), `appName` (string, required), `runtimeType` (string, required), `dockerfilePath` (string, default `Dockerfile`), `buildContext` (string, default `.`), `dryRun` (boolean, default `false`)
- [x] 2.2 Verify no additional parameters are declared beyond the six in the spec

## 3. Base Template — Stage Structure

- [x] 3.1 Add `stages:` block with five named stages: `Setup`, `Build`, `SignAndAttest`, `Publish`, `Notify`
- [x] 3.2 Add explicit `dependsOn` to each stage: `Build` → `Setup`; `SignAndAttest` → `Build`; `Publish` → `SignAndAttest`; `Notify` → `Publish`
- [x] 3.3 Add `condition: and(succeeded(), eq('${{ parameters.dryRun }}', 'false'))` to `SignAndAttest` stage
- [x] 3.4 Add `condition: and(succeeded(), eq('${{ parameters.dryRun }}', 'false'))` to `Publish` stage
- [x] 3.5 Add `condition: succeededOrFailed()` to `Notify` stage so it fires on both dry runs and full runs

## 4. Step Template Stubs — Core Steps

- [x] 4.1 Create `platform-templates/steps/setup.yml` as a minimal valid ADO step template stub with a single `bash` step echoing `STUB: setup.yml not yet implemented`
- [x] 4.2 Create `platform-templates/steps/dockerfile-lint.yml` as a minimal valid ADO step template stub with a single `bash` step echoing `STUB: dockerfile-lint.yml not yet implemented`
- [x] 4.3 Create `platform-templates/steps/docker-build.yml` as a minimal valid ADO step template stub with a single `bash` step echoing `STUB: docker-build.yml not yet implemented`
- [x] 4.4 Create `platform-templates/steps/sbom-sign-publish.yml` as a minimal valid ADO step template stub with a single `bash` step echoing `STUB: sbom-sign-publish.yml not yet implemented`

## 5. Step Template Stubs — Runtime Steps

- [x] 5.1 Create `platform-templates/steps/runtime/go.yml` as a minimal valid ADO step template stub with a single `bash` step echoing `STUB: go.yml runtime not yet implemented`
- [x] 5.2 Create `platform-templates/steps/runtime/python.yml` as a minimal valid ADO step template stub with a single `bash` step echoing `STUB: python.yml runtime not yet implemented`
- [x] 5.3 Create `platform-templates/steps/runtime/springboot.yml` as a minimal valid ADO step template stub with a single `bash` step echoing `STUB: springboot.yml runtime not yet implemented`
- [x] 5.4 Create `platform-templates/steps/runtime/angular.yml` as a minimal valid ADO step template stub with a single `bash` step echoing `STUB: angular.yml runtime not yet implemented`
- [x] 5.5 Create `platform-templates/steps/runtime/react.yml` as a minimal valid ADO step template stub with a single `bash` step echoing `STUB: react.yml runtime not yet implemented`

## 6. Base Template — Runtime Dispatch Block

- [x] 6.1 Add `${{ if eq(parameters.runtimeType, 'go') }}` block in the Build stage calling `steps/runtime/go.yml`
- [x] 6.2 Add `${{ if eq(parameters.runtimeType, 'python') }}` block in the Build stage calling `steps/runtime/python.yml`
- [x] 6.3 Add `${{ if eq(parameters.runtimeType, 'springboot') }}` block in the Build stage calling `steps/runtime/springboot.yml`
- [x] 6.4 Add `${{ if eq(parameters.runtimeType, 'angular') }}` block in the Build stage calling `steps/runtime/angular.yml`
- [x] 6.5 Add `${{ if eq(parameters.runtimeType, 'react') }}` block in the Build stage calling `steps/runtime/react.yml`
- [x] 6.6 Verify there is no `else` or `default` branch in the dispatch block that falls through to an arbitrary template

## 7. Base Template — Core Step References

- [x] 7.1 Wire the `Setup` stage to call `steps/setup.yml` as a template step reference
- [x] 7.2 Wire the `Build` stage to call `steps/dockerfile-lint.yml` and `steps/docker-build.yml` (in that order) as template step references, before the runtime dispatch block
- [x] 7.3 Wire the `SignAndAttest` stage to call `steps/sbom-sign-publish.yml` (sign & attest portion)
- [x] 7.4 Wire the `Publish` stage to call `steps/sbom-sign-publish.yml` (publish portion)
- [x] 7.5 Wire the `Notify` stage to call `steps/sbom-sign-publish.yml` (notify portion)

## 8. Validation

- [x] 8.1 Run ADO YAML linter or `az pipelines validate` against `container-build-v2.yml` to confirm no parse or schema errors
- [x] 8.2 Create a minimal test tenant `azure-pipelines.yml` that uses `extends:` referencing the base template with `runtimeType: go` and `dryRun: true`; confirm it loads in ADO without errors
- [ ] 8.3 Queue the test tenant pipeline and verify all five stages appear in the ADO graph; confirm `SignAndAttest` and `Publish` are shown as skipped
- [ ] 8.4 Check build logs confirm the `go.yml` stub step ran and emitted `STUB: go.yml runtime not yet implemented`
- [ ] 8.5 Repeat 8.2–8.4 with at least one other `runtimeType` (e.g., `python`) to confirm dispatch routing is correct
