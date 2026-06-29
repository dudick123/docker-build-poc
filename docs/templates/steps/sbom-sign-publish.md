# Template: steps/sbom-sign-publish.yml

**Path:** `platform-templates/steps/sbom-sign-publish.yml`
**Type:** Step template (multi-phase)
**Called by:** `container-build-v2.yml` — Stages 3, 4, and 5

---

## Description

A single step template that covers three distinct pipeline phases, selected at call time via the `phase` parameter. Separating phases into one file rather than three keeps the Syft/Cosign/ACR logic co-located and ensures consistent parameter naming across the sign → publish → notify chain.

The three phases are:

| Phase | Stage | Purpose |
|---|---|---|
| `signAndAttest` | Stage 3 — Sign & Attest | SBOM generation, Cosign sign, attest, verify |
| `publish` | Stage 4 — Publish | Tag push, digest verify, provenance artifact |
| `notify` | Stage 5 — Notify | PR comment, ADO build tag, Teams notification |

At compile time, `${{ if eq(parameters.phase, '...') }}` blocks ensure that only the steps for the active phase are included in the compiled pipeline YAML. Steps from other phases are not present at runtime.

---

## Parameters

| Name | Type | Default | Used by phases | Description |
|---|---|---|---|---|
| `phase` | string (enum) | — | all | `signAndAttest`, `publish`, or `notify` |
| `tenantName` | string | — | all | Tenant slug |
| `appName` | string | — | all | Application name |
| `dryRun` | boolean | `false` | publish, notify | Dry run flag for status detection in notify |
| `acrHost` | string | `''` | signAndAttest, publish, notify | ACR FQDN |
| `imageDigest` | string | `''` | signAndAttest | Config digest from `buildImage` step (passed but not used for signing) |
| `syftVersion` | string | `''` | signAndAttest | Pinned Syft version to download |
| `cosignVersion` | string | `''` | signAndAttest | Pinned Cosign version to download |
| `manifestDigest` | string | `''` | publish, notify | `sha256:...` manifest digest from `signAttest` step |
| `runtimeVersion` | string | `''` | publish | Version string for optional version tag |
| `imageRef` | string | `''` | notify | `<acr>/<tenant>/<app>@sha256:...` from `publish` step |
| `runtimeType` | string | `''` | notify | Used in PR comment and Teams notification |

---

## Phase: signAndAttest

### Called from

```yaml
# Stage 3 in container-build-v2.yml
- template: steps/sbom-sign-publish.yml
  parameters:
    phase: signAndAttest
    tenantName: ${{ parameters.tenantName }}
    appName: ${{ parameters.appName }}
    dryRun: ${{ parameters.dryRun }}
    acrHost: $(stageDependencies.Setup.Setup.outputs['resolveTools.ACR_HOST'])
    imageDigest: $(stageDependencies.Build.Build.outputs['buildImage.IMAGE_DIGEST'])
    syftVersion: $(stageDependencies.Setup.Setup.outputs['resolveTools.SYFT_VERSION'])
    cosignVersion: $(stageDependencies.Setup.Setup.outputs['resolveTools.COSIGN_VERSION'])
```

### Steps

**1. Download Syft (bash)**

Downloads the Syft binary from GitHub releases:
```
https://github.com/anchore/syft/releases/download/v<VERSION>/syft_<VERSION>_linux_amd64.tar.gz
```
Extracts to `/tmp/syft` and prints version to confirm.

**2. Download Cosign (bash)**

Downloads the Cosign binary from GitHub releases:
```
https://github.com/sigstore/cosign/releases/download/v<VERSION>/cosign-linux-amd64
```
Saves to `/tmp/cosign` and prints version to confirm.

**3. Retrieve Cosign Keys from AKV (`AzureKeyVault@2`)**

Retrieves two secrets from the platform Azure Key Vault:
- `cosign-private-key` → available as `$(cosign-private-key)` in subsequent steps
- `cosign-public-key` → available as `$(cosign-public-key)` in subsequent steps

Requires pipeline variables `COSIGN_AKV_SERVICE_CONNECTION` and `COSIGN_KEY_VAULT_NAME` to be set in the pipeline environment.

**4. Generate SBOM — `generateSbom` (bash)**

Runs Syft against the local image (which still exists from Stage 2):
```bash
/tmp/syft "<tenantName>/<appName>:pipeline-<BUILD_BUILDID>" \
  -o "cyclonedx-json=$(System.DefaultWorkingDirectory)/sbom.cdx.json"
```
Produces a CycloneDX JSON SBOM at `sbom.cdx.json`.

**5. Publish SBOM (`PublishPipelineArtifact@1`)**

Publishes `sbom.cdx.json` as ADO pipeline artifact:
- Artifact name: `sbom-<tenantName>-<appName>`

**6. Push, Sign, Attest, Verify — `signAttest` (bash)**

This is the core signing step. Key behaviors:

```bash
trap 'rm -f /tmp/cosign.key /tmp/cosign.pub' EXIT
```
Key files are always cleaned up, regardless of step outcome.

```bash
docker tag "$LOCAL_IMAGE_REF" "$ACR_IMAGE_REF:$SHORT_SHA"
docker push "$ACR_IMAGE_REF:$SHORT_SHA"
```
The image must be in ACR before Cosign can sign it. A short-SHA (12-character) staging tag is used. This is the first ACR write in the pipeline.

```bash
MANIFEST_DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' "$ACR_IMAGE_REF:$SHORT_SHA" \
  | grep -oP 'sha256:[a-f0-9]+')
```
After push, `docker inspect` returns the manifest digest (`RepoDigests[0]`). This is the correct digest for Cosign — not the config digest from `--iidfile`.

```bash
/tmp/cosign sign --key /tmp/cosign.key "$ACR_IMAGE_REF@$MANIFEST_DIGEST"
/tmp/cosign attest --key /tmp/cosign.key --predicate sbom.cdx.json --type cyclonedx "$ACR_IMAGE_REF@$MANIFEST_DIGEST"
/tmp/cosign verify --key /tmp/cosign.pub "$ACR_IMAGE_REF@$MANIFEST_DIGEST"
```
Signs the manifest digest, attaches the SBOM as a CycloneDX attestation, then verifies the signature. The pipeline hard-fails if verification does not pass.

**Output variable:**

| Variable | Step name | Cross-stage reference |
|---|---|---|
| `MANIFEST_DIGEST` | `signAttest` | `$(stageDependencies.SignAndAttest.SignAndAttest.outputs['signAttest.MANIFEST_DIGEST'])` |

---

## Phase: publish

### Called from

```yaml
# Stage 4 in container-build-v2.yml
- template: steps/sbom-sign-publish.yml
  parameters:
    phase: publish
    tenantName: ${{ parameters.tenantName }}
    appName: ${{ parameters.appName }}
    dryRun: ${{ parameters.dryRun }}
    acrHost: $(stageDependencies.Setup.Setup.outputs['resolveTools.ACR_HOST'])
    manifestDigest: $(stageDependencies.SignAndAttest.SignAndAttest.outputs['signAttest.MANIFEST_DIGEST'])
    runtimeVersion: $(RUNTIME_VERSION)   # job-level variable selected at compile time per runtimeType
```

### Steps

**1. Assert no `latest` tag — `assertTags` (bash)**

Computes all candidate tags (full SHA, alias, version if applicable) and asserts that none equals the string `latest`. Fails immediately if any candidate equals `latest`. This check runs before any ACR write.

**2. Push tags and verify — `publish` (bash)**

Executes in this order:

1. Tag and push the primary (full 40-char SHA) tag
2. Read the manifest digest after push and compare to `$MANIFEST_DIGEST` from Stage 3 — fail if mismatch
3. Tag and push the alias tag (`<branch>-<12-char-sha>`)
4. If `runtimeVersion` is non-empty:
   - On `main` branch: check if tag already exists in ACR (`docker manifest inspect`); fail if it does; push bare version tag
   - On other branches: push `<version>-<12-char-sha>` (always safe, never collides)
5. Emit `IMAGE_REF` as `<acrHost>/<tenantName>/<appName>@<manifestDigest>`

**Output variable:**

| Variable | Step name | Cross-stage reference |
|---|---|---|
| `IMAGE_REF` | `publish` | `$(stageDependencies.Publish.Publish.outputs['publish.IMAGE_REF'])` |

**3. Write provenance — `writeProvenance` (bash)**

Uses `jq -n` to build `provenance.json` with these fields:

| Field | Value |
|---|---|
| `imageRef` | `<acr>/<tenant>/<app>@sha256:...` |
| `manifestDigest` | `sha256:...` |
| `tags` | JSON array of all pushed tags |
| `sbomArtifact` | `sbom-<tenantName>-<appName>` |
| `cosignStatus` | `"signed"` |
| `pipelineRunId` | ADO `BUILD_BUILDID` |
| `acrRepository` | `<acr>/<tenant>/<app>` |
| `gitCommit` | Full 40-char Git SHA |

**4. Publish Provenance (`PublishPipelineArtifact@1`)**

Publishes `provenance.json` as ADO pipeline artifact:
- Artifact name: `provenance-<tenantName>-<appName>`

---

## Phase: notify

### Called from

```yaml
# Stage 5 in container-build-v2.yml
- template: steps/sbom-sign-publish.yml
  parameters:
    phase: notify
    tenantName: ${{ parameters.tenantName }}
    appName: ${{ parameters.appName }}
    dryRun: ${{ parameters.dryRun }}
    runtimeType: ${{ parameters.runtimeType }}
    acrHost: $(stageDependencies.Setup.Setup.outputs['resolveTools.ACR_HOST'])
    manifestDigest: $(stageDependencies.SignAndAttest.SignAndAttest.outputs['signAttest.MANIFEST_DIGEST'])
    imageRef: $(stageDependencies.Publish.Publish.outputs['publish.IMAGE_REF'])
```

### Steps

**1. Post PR comment — `postPrComment` (bash)**

Skips gracefully if `BUILD_REASON != PullRequest`.

Determines status from available context:
- `IMAGE_REF` non-empty → Succeeded ✅
- `PIPELINE_DRYRUN = True` → Dry Run 🔵
- Otherwise → Failed ❌

Builds a Markdown table with tenant/app, runtime, status, image ref, manifest digest, Cosign status, and SBOM artifact name. Posts to the PR thread via the ADO REST API:

```
POST <collection>/<project>/_apis/git/repositories/<repoId>/pullRequests/<prId>/threads?api-version=7.1
Authorization: Bearer $(System.AccessToken)
```

Uses `jq -n --arg body` to construct the JSON payload (avoids heredoc/YAML indentation conflicts).

**2. Set ADO build tag — `setBuildTag` (bash)**

Emits an ADO logging command to set the pipeline run build tag:
- Non-dry-run: `##vso[build.addbuildtag]<MANIFEST_DIGEST>` — links the pipeline run to the exact image manifest
- Dry run: `##vso[build.addbuildtag]dryrun-<BUILD_BUILDID>`

**3. Send Teams notification — `notifyTeams` (bash)**

Reads `$(TEAMS_WEBHOOK_URL)` from the environment. If empty, logs a warning and exits cleanly — Teams notification is never a blocking condition.

Builds an Adaptive Card (Teams webhook JSON format) via `jq -n` with:
- Title: `Build <Status>: <tenantName>/<appName>`
- Color: `Good` (succeeded), `Accent` (dry run), `Attention` (failed)
- Facts: runtime, status, detail text
- Action button: "View Pipeline Run" (links to ADO build results)

---

## Output Variables Summary

| Variable | Phase | Step | Cross-stage reference |
|---|---|---|---|
| `MANIFEST_DIGEST` | `signAndAttest` | `signAttest` | `$(stageDependencies.SignAndAttest.SignAndAttest.outputs['signAttest.MANIFEST_DIGEST'])` |
| `IMAGE_REF` | `publish` | `publish` | `$(stageDependencies.Publish.Publish.outputs['publish.IMAGE_REF'])` |

---

## Pipeline Artifacts Produced

| Artifact name | Phase | Contents |
|---|---|---|
| `sbom-<tenantName>-<appName>` | signAndAttest | CycloneDX JSON SBOM |
| `provenance-<tenantName>-<appName>` | publish | JSON provenance record |

---

## ACR Writes Per Phase

| Phase | Tags/objects written to ACR |
|---|---|
| `signAndAttest` | `<image>:<short-sha>` (staging tag for signing), `<image>:<sha>.sig`, `<image>:<sha>.att` |
| `publish` | `<image>:<full-sha>`, `<image>:<branch>-<short-sha>`, `<image>:<version>` (conditional) |
| `notify` | None |

---

## Design Notes

- Cosign requires the image to be in a registry before signing — it cannot sign a locally held image. Stage 3 pushes under a short-SHA staging tag first, then reads the manifest digest from `docker inspect`.
- `--iidfile` from `docker build` captures the *config digest*, not the *manifest digest*. These are different values. Only the manifest digest (from `docker inspect` after ACR push) is suitable for Cosign.
- Key files are written to `/tmp/cosign.key` and `/tmp/cosign.pub` with a `trap 'rm -f ...' EXIT` to guarantee deletion whether the step succeeds or fails.
- All JSON construction uses `jq -n --arg`/`--argjson`. Python heredocs and bash heredocs were ruled out due to YAML block scalar indentation conflicts when content appears at column 0.
- The Notify stage uses `in(dependencies.Publish.result, 'Succeeded', 'Failed', 'SucceededWithIssues', 'Skipped')` rather than `succeededOrFailed()`. The latter does not fire when the upstream stage is in a Skipped state (dry run).
