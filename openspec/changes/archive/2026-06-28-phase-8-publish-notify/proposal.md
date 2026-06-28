## Why

Stage 4 (Publish) and Stage 5 (Notify) are currently stubs in `sbom-sign-publish.yml`. The pipeline can build, sign, and attest images but cannot complete the delivery loop: pushing the full tag set to ACR, verifying the digest, surfacing provenance to the PR, or triggering the downstream security scan pipeline.

## What Changes

- Implement the `publish` phase in `steps/sbom-sign-publish.yml`: push full-SHA, alias, and version tags to ACR; assert `latest` tag is never pushed; verify manifest digest matches Stage 3; emit `IMAGE_REF` output variable; publish provenance summary as pipeline artifact.
- Implement the `notify` phase in `steps/sbom-sign-publish.yml`: post build summary PR comment (digest, tags, SBOM status, signing status, security scan note); post Teams channel notification via per-tenant webhook; tag the ADO pipeline run with the image digest.
- Update `container-build-v2.yml` to pass `acrHost`, `manifestDigest`, and runtime-detected version strings from Stage 2 outputs to the Publish and Notify stages.

## Capabilities

### New Capabilities

- `acr-publish`: Multi-tag push of the signed image to ACR (full SHA primary tag, alias tag, version tag conditional on branch and file), `latest`-tag prohibition assertion, digest verification against Stage 3 `MANIFEST_DIGEST`, and `IMAGE_REF` output variable emission.
- `pipeline-notify`: PR comment with build provenance summary (digest, ACR path, tags, SBOM artifact location, Cosign status, security scan advisory link); Teams webhook notification (success and failure); ADO pipeline run build tag set to image digest.

### Modified Capabilities

- `stage-structure`: Publish and Notify template calls must pass `acrHost` and `manifestDigest` (from Stage 3 output). Publish must receive runtime version string outputs from Stage 2 to determine the version tag. Notify must receive build outcome metadata and `imageRef`.

## Impact

- `platform-templates/steps/sbom-sign-publish.yml` — publish and notify phase branches replaced (stubs removed)
- `platform-templates/container-build-v2.yml` — Publish and Notify stage template calls gain new parameters
- New parameters added to `sbom-sign-publish.yml`: `manifestDigest`, `imageRef`, `runtimeVersion`, `branchName`
- Hard dependency: per-tenant variable group `tenant-<tenantName>-notifications` must contain `TEAMS_WEBHOOK_URL` for Teams notifications
- Hard dependency: ADO PR ID (`Build.Reason == PullRequest`) required for PR comment; Notify step skips comment gracefully on non-PR builds
