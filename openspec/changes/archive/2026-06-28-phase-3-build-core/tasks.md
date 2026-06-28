## 1. Variable Group Update

- [x] 1.1 Add `ACR_HOST` variable to the `platform-tool-versions` ADO variable group definition (document the required variable name so platform engineers know to provision it alongside the existing five)
- [x] 1.2 Update the `resolveTools` step in `platform-templates/steps/setup.yml` to also read and emit `ACR_HOST` as a step output variable (same pattern as the existing five: check for empty, emit with `isOutput=true`)

## 2. Base Template — Pass ACR_HOST to Build Steps

- [x] 2.1 Add `acrHost` as a parameter to `platform-templates/steps/docker-build.yml` (string, required) so the step can construct the cache registry reference
- [x] 2.2 Update the `docker-build.yml` template reference in `container-build-v2.yml`'s Build stage to pass `acrHost: $(stageDependencies.Setup.Setup.outputs['resolveTools.ACR_HOST'])` alongside the existing parameters

## 3. dockerfile-lint.yml Implementation

- [x] 3.1 Replace the stub body of `platform-templates/steps/dockerfile-lint.yml`; add a `hadolintVersion` parameter (string, required) to receive the pinned version from the Build stage
- [x] 3.2 Add a bash step that downloads the Hadolint binary from `https://github.com/hadolint/hadolint/releases/download/<version>/hadolint-Linux-x86_64`, writes it to a temp path, and `chmod +x`
- [x] 3.3 Add logic to detect `.hadolint.yaml` in `<buildContext>/`; if present, pass `--config <buildContext>/.hadolint.yaml` to Hadolint; if absent, omit the flag
- [x] 3.4 Invoke Hadolint with `--failure-threshold error --format tty -f ${{ parameters.dockerfilePath }}` (relative to `buildContext`) so ERROR findings exit non-zero and WARNING findings do not
- [x] 3.5 Update the `dockerfile-lint.yml` template reference in `container-build-v2.yml`'s Build stage to pass `hadolintVersion: $(stageDependencies.Setup.Setup.outputs['resolveTools.HADOLINT_VERSION'])`

## 4. docker-build.yml Implementation

- [x] 4.1 Replace the stub body of `platform-templates/steps/docker-build.yml`; build the image reference as `${{ parameters.tenantName }}/${{ parameters.appName }}:pipeline-$(Build.BuildId)` for the local tag (no ACR host prefix yet — local only)
- [x] 4.2 Add the `buildImage` bash step with `DOCKER_BUILDKIT=1`; construct the full `docker build` command with all required flags
- [x] 4.3 Inject all four OCI labels via `--label` flags using ADO predefined variables for source, created, revision, and title; use `$(date -u +%Y-%m-%dT%H:%M:%SZ)` for `image.created`
- [x] 4.4 After `docker build`, write the image ID to a temp file using `--iidfile /tmp/image.iid`, read it back to get the `sha256:<digest>` form
- [x] 4.5 Emit the digest as `echo "##vso[task.setvariable variable=IMAGE_DIGEST;isOutput=true]$DIGEST"` from the `buildImage` step so downstream stages can reference it as `$(stageDependencies.Build.Build.outputs['buildImage.IMAGE_DIGEST'])`
- [x] 4.6 Ensure all parameter values used in the bash script body are passed through the step `env:` block (not inlined as `${{ parameters.xxx }}` in script text) to prevent template injection

## 5. YAML and Shell Validation

- [x] 5.1 Run `python3 -c "import yaml; yaml.safe_load(open('platform-templates/steps/dockerfile-lint.yml'))"` — confirm no parse errors
- [x] 5.2 Run `python3 -c "import yaml; yaml.safe_load(open('platform-templates/steps/docker-build.yml'))"` — confirm no parse errors
- [x] 5.3 Run `python3 -c "import yaml; yaml.safe_load(open('platform-templates/container-build-v2.yml'))"` — confirm no parse errors after parameter additions
- [x] 5.4 Verify `dockerfile-lint.yml` has no `${{ parameters.xxx }}` expansions in bash script body (all values via `env:` block)
- [x] 5.5 Verify `docker-build.yml` `buildImage` step has no `${{ parameters.xxx }}` expansions in bash script body (all values via `env:` block)

## 6. ADO End-to-End Validation

- [ ] 6.1 Queue a `dryRun=true` pipeline with a simple Go Dockerfile; confirm lint passes (no ERROR findings) and build produces a local image with the correct four OCI labels
- [ ] 6.2 Confirm `IMAGE_DIGEST` appears in the `buildImage` step log in `sha256:<hex>` format
- [ ] 6.3 Confirm the image is NOT present in ACR after the pipeline run (local only)
- [ ] 6.4 Introduce a Hadolint ERROR-level violation (e.g., `ADD` instead of `COPY`) in a test Dockerfile; confirm the lint step fails and the build step does not run
- [ ] 6.5 Verify ACR layer cache is written on a successful build and the next run shows a cache hit in the BuildKit output
