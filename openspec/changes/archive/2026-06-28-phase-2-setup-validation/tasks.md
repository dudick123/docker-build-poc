## 1. Variable Group Linkage

- [x] 1.1 Add `platform-tool-versions` variable group to the `variables:` block of `container-build-v2.yml` so it is available to all stages without tenant action
- [x] 1.2 Verify the variable group reference uses the ADO `group:` syntax and is placed at the template root `variables:` level (not inside a stage)

## 2. Tool Version Resolution Step

- [x] 2.1 Replace the stub body of `platform-templates/steps/setup.yml` with a step named `resolveTools` that reads `DOCKER_BUILDKIT_VERSION`, `SYFT_VERSION`, `COSIGN_VERSION`, `HADOLINT_VERSION`, and `NPM_REGISTRY_URL` from the linked variable group
- [x] 2.2 For each of the five variables: check if the value is empty; if so, add an error message `"Variable <NAME> is missing or empty in platform-tool-versions variable group"` to the errors list
- [x] 2.3 Emit each non-empty variable as a step output variable using `echo "##vso[task.setvariable variable=<NAME>;isOutput=true]$(VARNAME)"` so downstream steps can reference them as `$(Setup.resolveTools.<NAME>)`
- [x] 2.4 Retain the `parameters:` block from Phase 1 stub (all five parameters); add them as inputs to the validation bash steps where needed

## 3. Naming Convention Validation

- [x] 3.1 Add a validation block that tests `tenantName` against `^[a-z0-9][a-z0-9-]*[a-z0-9]$` using bash `=~`; if it fails, append an error: `"tenantName '<value>' does not match required pattern ^[a-z0-9][a-z0-9-]*[a-z0-9]$ (lowercase alphanumeric and hyphens only, minimum 2 characters)"`
- [x] 3.2 Add an identical validation block for `appName` with the same pattern and an equivalent error message naming `appName`

## 4. runtimeType Allowlist Validation

- [x] 4.1 Add a validation block that checks `runtimeType` against the list `angular react springboot python go`; if the value is not in the list, append an error: `"runtimeType '<value>' is not supported. Allowed values: angular, react, springboot, python, go"`

## 5. dockerfilePath Existence Check

- [x] 5.1 Add a validation block that constructs the resolved path as `$(Build.SourcesDirectory)/${{ parameters.buildContext }}/${{ parameters.dockerfilePath }}` and runs `test -f` against it; if the file does not exist, append an error: `"dockerfilePath not found: <resolved-path>"`

## 6. Runtime Template Existence Check

- [x] 6.1 Add a validation block that checks for the existence of `$(Agent.BuildDirectory)/s/platform-templates/steps/runtime/${{ parameters.runtimeType }}.yml`; if missing, append an error: `"Runtime template not found: steps/runtime/${{ parameters.runtimeType }}.yml — contact platform engineering"`

## 7. Fail-All Error Reporting

- [x] 7.1 Wrap all validation blocks in a single bash step; use a `ERRORS` bash array to collect error strings across all checks
- [x] 7.2 After all checks complete, if `${#ERRORS[@]} -gt 0`: iterate the array and print each error prefixed with `##vso[task.logissue type=error]`, then `exit 1`
- [x] 7.3 If no errors, print a summary line confirming all parameters passed validation (e.g., `All parameters valid. tenantName=<value>, appName=<value>, runtimeType=<value>`)

## 8. YAML Validation

- [x] 8.1 Run `python3 -c "import yaml; yaml.safe_load(open('platform-templates/steps/setup.yml'))"` to confirm no parse errors
- [x] 8.2 Verify the `resolveTools` step name is present and that output variable declarations use `isOutput=true`

## 9. ADO End-to-End Validation

- [ ] 9.1 Queue a test pipeline with valid parameters (`tenantName: test-tenant`, `appName: test-app`, `runtimeType: go`) and confirm Stage 1 succeeds; verify the five tool version variables appear in the step log
- [ ] 9.2 Queue a test pipeline with `tenantName: MyTenant` (uppercase) and `runtimeType: dotnet` (invalid); confirm Stage 1 fails and both errors appear in the same step output
- [ ] 9.3 Queue a test pipeline with a `dockerfilePath` that does not exist on the agent; confirm the error message includes the fully resolved path
- [ ] 9.4 Confirm that downstream steps in Phase 3 stubs can reference `$(Setup.resolveTools.HADOLINT_VERSION)` without error (value may be empty until Phase 3 is implemented, but the reference should not break the pipeline)
