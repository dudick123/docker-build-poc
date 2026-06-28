## 1. container-build-v2.yml — SignAndAttest Stage Parameters

- [x] 1.1 Update the `SignAndAttest` stage's `sbom-sign-publish.yml` template call to pass `acrHost: $(stageDependencies.Setup.Setup.outputs['resolveTools.ACR_HOST'])`
- [x] 1.2 Pass `imageDigest: $(stageDependencies.Build.Build.outputs['buildImage.IMAGE_DIGEST'])` to the SignAndAttest template call
- [x] 1.3 Pass `syftVersion: $(stageDependencies.Setup.Setup.outputs['resolveTools.SYFT_VERSION'])` to the SignAndAttest template call
- [x] 1.4 Pass `cosignVersion: $(stageDependencies.Setup.Setup.outputs['resolveTools.COSIGN_VERSION'])` to the SignAndAttest template call
- [x] 1.5 Run YAML validation on `platform-templates/container-build-v2.yml` after changes

## 2. sbom-sign-publish.yml — Parameters

- [x] 2.1 Add parameters to `platform-templates/steps/sbom-sign-publish.yml`: `acrHost` (string), `imageDigest` (string, default empty), `syftVersion` (string, default empty), `cosignVersion` (string, default empty)
- [x] 2.2 Confirm existing parameters (`phase`, `tenantName`, `appName`, `dryRun`) are retained unchanged

## 3. sbom-sign-publish.yml — Tool Download Steps (signAndAttest phase)

- [x] 3.1 Add a bash step (conditioned on `${{ if eq(parameters.phase, 'signAndAttest') }}`) that downloads Syft: `curl -sSfL "https://github.com/anchore/syft/releases/download/${SYFT_VERSION}/syft_${SYFT_VERSION}_linux_amd64.tar.gz" | tar -xz -C /tmp && chmod +x /tmp/syft`; pass `SYFT_VERSION` via `env:` block
- [x] 3.2 Add a bash step that downloads Cosign: `curl -sSfL "https://github.com/sigstore/cosign/releases/download/v${COSIGN_VERSION}/cosign-linux-amd64" -o /tmp/cosign && chmod +x /tmp/cosign`; pass `COSIGN_VERSION` via `env:` block

## 4. sbom-sign-publish.yml — AKV Key Retrieval

- [x] 4.1 Add an `AzureKeyVault@2` task (conditioned on `signAndAttest` phase) that downloads the Cosign private key and public key secrets from the platform AKV; use `$(COSIGN_KEY_VAULT_NAME)` and `$(COSIGN_AKV_SERVICE_CONNECTION)` from the `platform-tool-versions` variable group; `SecretsFilter` to include `cosign-private-key,cosign-public-key`

## 5. sbom-sign-publish.yml — SBOM Generation and Artifact Publish

- [x] 5.1 Add a bash step (conditioned on `signAndAttest` phase) named `generateSbom` that runs `/tmp/syft $LOCAL_IMAGE_REF -o cyclonedx-json=sbom.cdx.json`; construct `LOCAL_IMAGE_REF` as `$TENANT_NAME/$APP_NAME:pipeline-$BUILD_BUILDID`; pass `TENANT_NAME`, `APP_NAME` via `env:` block
- [x] 5.2 Add a `PublishPipelineArtifact@1` task step that publishes `sbom.cdx.json` as artifact named `sbom-$(tenantName)-$(appName)`; set `targetPath` to `$(System.DefaultWorkingDirectory)/sbom.cdx.json`

## 6. sbom-sign-publish.yml — ACR Push and Manifest Digest Capture

- [x] 6.1 Add a bash step (conditioned on `signAndAttest` phase) named `signAttest` that:
  - Tags the local image for ACR: `docker tag $LOCAL_IMAGE_REF $ACR_HOST/$TENANT_NAME/$APP_NAME:$SHORT_SHA` where `SHORT_SHA` is first 12 chars of `BUILD_SOURCEVERSION`
  - Pushes the image: `docker push $ACR_HOST/$TENANT_NAME/$APP_NAME:$SHORT_SHA`
  - Extracts the manifest digest: `MANIFEST_DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' $ACR_HOST/$TENANT_NAME/$APP_NAME:$SHORT_SHA | grep -oP 'sha256:[a-f0-9]+')`
  - Passes `ACR_HOST`, `TENANT_NAME`, `APP_NAME` via `env:` block

## 7. sbom-sign-publish.yml — Cosign Sign, Attest, Verify (in signAttest step)

- [x] 7.1 Within the `signAttest` bash step, write the Cosign private key to `/tmp/cosign.key` and public key to `/tmp/cosign.pub` using AKV-provided pipeline variables; set `trap 'rm -f /tmp/cosign.key /tmp/cosign.pub' EXIT` immediately after writing
- [x] 7.2 Run `cosign sign --key /tmp/cosign.key "$ACR_FULL_REF@$MANIFEST_DIGEST"` where `ACR_FULL_REF` is `$ACR_HOST/$TENANT_NAME/$APP_NAME`
- [x] 7.3 Run `cosign attest --key /tmp/cosign.key --predicate sbom.cdx.json --type cyclonedx "$ACR_FULL_REF@$MANIFEST_DIGEST"`
- [x] 7.4 Run `cosign verify --key /tmp/cosign.pub "$ACR_FULL_REF@$MANIFEST_DIGEST"` — non-zero exit fails the step and blocks Stage 4
- [x] 7.5 Emit `echo "##vso[task.setvariable variable=MANIFEST_DIGEST;isOutput=true]$MANIFEST_DIGEST"` at the end of the `signAttest` step
- [x] 7.6 Confirm `/tmp/cosign.key` and `/tmp/cosign.pub` are deleted by the `EXIT` trap (either on success or failure)

## 8. sbom-sign-publish.yml — Security Pattern and Stub Preservation

- [x] 8.1 Confirm all `${{ parameters.xxx }}` expressions are only in `env:` blocks or `${{ if ... }}` conditions — none inline in bash bodies
- [x] 8.2 Confirm the `publish` and `notify` phase branches of the stub are still intact and unchanged

## 9. YAML Validation

- [x] 9.1 Run `python3 -c "import yaml; yaml.safe_load(open('platform-templates/steps/sbom-sign-publish.yml'))"` — confirm no parse errors
- [x] 9.2 Confirm the `signAttest` step is named `signAttest` and `isOutput=true` is present on the `MANIFEST_DIGEST` emit
- [x] 9.3 Confirm the `AzureKeyVault@2` task is present and conditioned on the `signAndAttest` phase

## 10. ADO End-to-End Validation

- [ ] 10.1 Queue a `dryRun=false` pipeline with `runtimeType: go` against a simple Go repo; confirm Stage 3 runs, ACR contains `<image>:<sha>`, `<image>:<sha>.sig`, and `<image>:<sha>.att`; confirm `cosign verify` exits zero; confirm SBOM artifact is present in pipeline artifacts
- [ ] 10.2 Confirm Stage 4 does not run if `cosign verify` fails (test by temporarily using wrong public key)
- [ ] 10.3 Confirm `dryRun=true` skips Stage 3 entirely (no ACR push, no Cosign invocation)
- [ ] 10.4 Confirm Cosign key files are absent from the agent filesystem after Stage 3 completes
- [ ] 10.5 Confirm `MANIFEST_DIGEST` is accessible in Stage 4 via `$(stageDependencies.SignAndAttest.SignAndAttest.outputs['signAttest.MANIFEST_DIGEST'])`
