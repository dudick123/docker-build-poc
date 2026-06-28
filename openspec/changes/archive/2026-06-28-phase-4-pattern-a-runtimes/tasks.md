## 1. go.yml — go.mod Assertion

- [x] 1.1 Replace the stub body of `platform-templates/steps/runtime/go.yml`; retain the existing `buildContext` parameter declaration
- [x] 1.2 Add a bash step named `goRuntime` that constructs the resolved path `$BUILD_SOURCESDIRECTORY/$BUILD_CONTEXT/go.mod` and runs `test -f` against it
- [x] 1.3 If `go.mod` is not found, emit `##vso[task.logissue type=error]go.mod not found at <resolved-path>. The build context root must contain go.mod for runtimeType: go.` and `exit 1`

## 2. go.yml — VERSION File Read and Output Variable

- [x] 2.1 After the `go.mod` check, test for `$BUILD_SOURCESDIRECTORY/$BUILD_CONTEXT/VERSION`
- [x] 2.2 If `VERSION` exists, read its first non-empty line, strip leading/trailing whitespace, store in `GO_VER`; emit `echo "##vso[task.setvariable variable=GO_VERSION;isOutput=true]$GO_VER"`
- [x] 2.3 If `VERSION` does not exist, emit `echo "##vso[task.setvariable variable=GO_VERSION;isOutput=true]"` (empty string) and log a note that version tagging will be skipped
- [x] 2.4 Pass all parameter values through the `env:` block (`BUILD_CONTEXT: ${{ parameters.buildContext }}`); confirm no `${{ parameters.xxx }}` in the bash script body

## 3. go.yml — YAML Validation

- [x] 3.1 Run `python3 -c "import yaml; yaml.safe_load(open('platform-templates/steps/runtime/go.yml'))"` — confirm no parse errors
- [x] 3.2 Confirm the step is named `goRuntime` and `isOutput=true` is present on the `GO_VERSION` emit

## 4. python.yml — Dependency File Advisory Check

- [x] 4.1 Replace the stub body of `platform-templates/steps/runtime/python.yml`; retain the existing `buildContext` parameter declaration
- [x] 4.2 Add a bash step named `pythonRuntime` that checks for any of `requirements.txt`, `pyproject.toml`, or `poetry.lock` in `$BUILD_SOURCESDIRECTORY/$BUILD_CONTEXT/`
- [x] 4.3 If none found, emit `##vso[task.logissue type=warning]No recognized Python dependency file found in <buildContext>. Expected one of: requirements.txt, pyproject.toml, poetry.lock` and continue (do not exit non-zero)

## 5. python.yml — Version Extraction and Output Variable

- [x] 5.1 After the dependency check, attempt extraction from `pyproject.toml`: use `grep -E '^version\s*=\s*"[^"]+"'` to find a quoted version under `[project]` or `[tool.poetry]`; extract the value between quotes using `sed`
- [x] 5.2 If `pyproject.toml` yields no match, attempt extraction from `setup.cfg`: use `grep -E '^version\s*='` under `[metadata]`; extract the value after `=` and trim whitespace
- [x] 5.3 Emit the extracted value (or empty string) as `echo "##vso[task.setvariable variable=PYTHON_VERSION;isOutput=true]$PYTHON_VER"`
- [x] 5.4 Pass all parameter values through the `env:` block (`BUILD_CONTEXT: ${{ parameters.buildContext }}`); confirm no `${{ parameters.xxx }}` in the bash script body

## 6. python.yml — YAML Validation

- [x] 6.1 Run `python3 -c "import yaml; yaml.safe_load(open('platform-templates/steps/runtime/python.yml'))"` — confirm no parse errors
- [x] 6.2 Confirm the step is named `pythonRuntime` and `isOutput=true` is present on the `PYTHON_VERSION` emit

## 7. ADO End-to-End Validation

- [ ] 7.1 Queue a `dryRun=true` pipeline with `runtimeType: go` against a repo containing `go.mod` and a `VERSION` file; confirm `GO_VERSION` appears in step log with the expected value
- [ ] 7.2 Queue a `dryRun=true` pipeline with `runtimeType: go` against a repo with no `go.mod`; confirm Stage 1 fails with the path-specific error message
- [ ] 7.3 Queue a `dryRun=true` pipeline with `runtimeType: go` against a repo with `go.mod` but no `VERSION` file; confirm `GO_VERSION` is empty and the pipeline succeeds
- [ ] 7.4 Queue a `dryRun=true` pipeline with `runtimeType: python` against a repo with `pyproject.toml` containing a static version; confirm `PYTHON_VERSION` is emitted correctly
- [ ] 7.5 Queue a `dryRun=true` pipeline with `runtimeType: python` against a repo with no dependency files; confirm the warning appears and the pipeline does not fail
