## 1. docker-build.yml — npm Registry Build Arg and Secret

- [x] 1.1 Add `npmRegistryUrl` parameter to `platform-templates/steps/docker-build.yml` (type: string)
- [x] 1.2 Add `--build-arg "NPM_REGISTRY=$NPM_REGISTRY_URL"` to the `docker build` command in the `buildImage` bash step
- [x] 1.3 Add `--secret id=npm_token,env=SYSTEM_ACCESSTOKEN` to the `docker build` command in the `buildImage` bash step
- [x] 1.4 Add `NPM_REGISTRY_URL: ${{ parameters.npmRegistryUrl }}` and `SYSTEM_ACCESSTOKEN: $(System.AccessToken)` to the `env:` block of the `buildImage` step
- [x] 1.5 Confirm no `${{ parameters.npmRegistryUrl }}` appears in the bash script body — only in `env:` block

## 2. container-build-v2.yml — Base Template Wiring

- [x] 2.1 Pass `npmRegistryUrl: $(stageDependencies.Setup.Setup.outputs['resolveTools.NPM_REGISTRY_URL'])` to the `docker-build.yml` template call in the Build stage
- [x] 2.2 Update the Angular runtime dispatch to also pass `dockerfilePath: ${{ parameters.dockerfilePath }}` and `tenantName: ${{ parameters.tenantName }}` and `appName: ${{ parameters.appName }}`
- [x] 2.3 Update the React runtime dispatch to also pass `dockerfilePath: ${{ parameters.dockerfilePath }}` and `tenantName: ${{ parameters.tenantName }}` and `appName: ${{ parameters.appName }}`

## 3. angular.yml — Advisory Checks

- [x] 3.1 Replace the stub body of `platform-templates/steps/runtime/angular.yml`; add parameters: `buildContext` (existing), `dockerfilePath`, `tenantName`, `appName`
- [x] 3.2 Add a bash step named `angularRuntime` that checks for `$BUILD_SOURCESDIRECTORY/$BUILD_CONTEXT/package.json`; if absent, emit `##vso[task.logissue type=warning]No package.json found in <buildContext>. Expected package.json for runtimeType: angular.` and continue
- [x] 3.3 If `package.json` exists, check that `"ng` appears in its content using `grep -q '"ng'`; if not found, emit `##vso[task.logissue type=warning]'ng' not found in package.json scripts. Confirm this is an Angular project for runtimeType: angular.` and continue

## 4. angular.yml — Version Extraction and Output Variable

- [x] 4.1 Extract version from `package.json` using `grep -oP '"version"\s*:\s*"\K[^"]+' | head -1`; store in `ANGULAR_VER` (empty string if no match or file absent)
- [x] 4.2 If version found, log `ANGULAR_VERSION resolved to: $ANGULAR_VER`; if not found, log that version tagging will be skipped
- [x] 4.3 Emit `echo "##vso[task.setvariable variable=ANGULAR_VERSION;isOutput=true]$ANGULAR_VER"`
- [x] 4.4 Pass all parameter values through the `env:` block; confirm no `${{ parameters.xxx }}` in bash script body

## 5. angular.yml — YAML Validation

- [x] 5.1 Run `python3 -c "import yaml; yaml.safe_load(open('platform-templates/steps/runtime/angular.yml'))"` — confirm no parse errors
- [x] 5.2 Confirm the step is named `angularRuntime` and `isOutput=true` is present on the `ANGULAR_VERSION` emit

## 6. react.yml — Advisory Checks

- [x] 6.1 Replace the stub body of `platform-templates/steps/runtime/react.yml`; add parameters: `buildContext` (existing), `dockerfilePath`, `tenantName`, `appName`
- [x] 6.2 Add a bash step named `reactRuntime` that checks for `$BUILD_SOURCESDIRECTORY/$BUILD_CONTEXT/package.json`; if absent, emit `##vso[task.logissue type=warning]No package.json found in <buildContext>. Expected package.json for runtimeType: react.` and continue
- [x] 6.3 If `package.json` exists, check for `"next"` in its content using `grep -q '"next"'`; if found, set `IS_NEXTJS=true`
- [x] 6.4 If `IS_NEXTJS=true`, check `$BUILD_CONTEXT/next.config.js` and `$BUILD_CONTEXT/next.config.mjs` for `output.*export` using `grep -qE 'output.*export'`; if neither file matches, emit `##vso[task.logissue type=warning]Next.js app detected without 'output: export' in next.config.js. SSR apps require a Node.js runtime in the final image, not nginx. See platform reference Dockerfile at docs/reference-dockerfiles/react-ssr.Dockerfile.` and continue

## 7. react.yml — Version Extraction and Output Variable

- [x] 7.1 Extract version from `package.json` using `grep -oP '"version"\s*:\s*"\K[^"]+' | head -1`; store in `REACT_VER` (empty string if no match or file absent)
- [x] 7.2 If version found, log `REACT_VERSION resolved to: $REACT_VER`; if not found, log that version tagging will be skipped
- [x] 7.3 Emit `echo "##vso[task.setvariable variable=REACT_VERSION;isOutput=true]$REACT_VER"`
- [x] 7.4 Pass all parameter values through the `env:` block; confirm no `${{ parameters.xxx }}` in bash script body

## 8. react.yml — YAML Validation

- [x] 8.1 Run `python3 -c "import yaml; yaml.safe_load(open('platform-templates/steps/runtime/react.yml'))"` — confirm no parse errors
- [x] 8.2 Confirm the step is named `reactRuntime` and `isOutput=true` is present on the `REACT_VERSION` emit

## 9. Cross-File Validation

- [x] 9.1 Run YAML validation on `platform-templates/steps/docker-build.yml` after npm arg/secret additions
- [x] 9.2 Run YAML validation on `platform-templates/container-build-v2.yml` after base template wiring changes
- [x] 9.3 Confirm `SYSTEM_ACCESSTOKEN` is only in `env:` block of `docker-build.yml`, not in the bash script body

## 10. ADO End-to-End Validation

- [ ] 10.1 Queue a `dryRun=true` pipeline with `runtimeType: angular` against a repo with `package.json` containing `"version": "3.0.0"` and `"ng"` in scripts; confirm `ANGULAR_VERSION=3.0.0` in step log
- [ ] 10.2 Queue a `dryRun=true` pipeline with `runtimeType: angular` against a repo with no `package.json`; confirm the warning appears and the pipeline succeeds
- [ ] 10.3 Queue a `dryRun=true` pipeline with `runtimeType: react` against a Next.js repo without `output: 'export'`; confirm the SSR warning appears and pipeline succeeds
- [ ] 10.4 Queue a `dryRun=true` pipeline with `runtimeType: react` against a non-Next.js repo; confirm no SSR warning and `REACT_VERSION` is emitted
- [ ] 10.5 Confirm that a Go or Python `dryRun=true` pipeline still succeeds with the npm build arg and secret now present in `docker-build.yml` (no regression)
