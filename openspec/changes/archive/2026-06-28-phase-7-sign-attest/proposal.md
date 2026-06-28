## Why

Phases 1–6 deliver a fully-built, locally-held image with a validated digest. Phase 7 implements the Sign & Attest stage: generating a CycloneDX SBOM with Syft, pushing the image to ACR under its primary SHA tag (required before Cosign can sign), signing the manifest digest with the platform Cosign key, attaching the SBOM as an OCI attestation, and verifying the signature before allowing Stage 4 to proceed. This establishes cryptographic image provenance as a platform standard (FR-7.1–FR-7.6, FR-6.1–FR-6.4).

## What Changes

- **Implement** `platform-templates/steps/sbom-sign-publish.yml` for the `signAndAttest` phase (the `publish` and `notify` phases remain stubs until Phase 8):
  - **Tool setup:** Download Syft at `$SYFT_VERSION` and Cosign at `$COSIGN_VERSION` from pinned versions resolved in Setup stage
  - **Key retrieval:** ADO `AzureKeyVault@2` task downloads the Cosign private key from platform-managed Azure Key Vault; key never stored in pipeline variables or accessible to tenant teams
  - **SBOM generation:** `syft <local-image-ref> -o cyclonedx-json=sbom.cdx.json` — scans the locally held image (not source), capturing all runtime dependencies including base-image packages
  - **SBOM artifact publish:** ADO `PublishPipelineArtifact` task retains `sbom.cdx.json` as a named pipeline artifact for audit
  - **ACR push (primary tag only):** Push `<acr>/<tenant>/<app>@sha256:<build-digest>` to ACR — required before Cosign can read the manifest digest; captured in `MANIFEST_DIGEST` for downstream
  - **Sign:** `cosign sign --key /tmp/cosign.key <acr>/<tenant>/<app>@<manifest-digest>` — signs the manifest digest, not a tag; produces `<image>:<sha>.sig` in ACR
  - **Attest:** `cosign attest --key /tmp/cosign.key --predicate sbom.cdx.json --type cyclonedx <acr>/<tenant>/<app>@<manifest-digest>` — attaches SBOM as OCI attestation; produces `<image>:<sha>.att` in ACR
  - **Verify:** `cosign verify --key cosign.pub <acr>/<tenant>/<app>@<manifest-digest>` — verifies the signature immediately; non-zero exit blocks Stage 4
  - **Emit:** `MANIFEST_DIGEST` step output variable (sha256 from ACR push) for Phase 8

- **Modify** `platform-templates/container-build-v2.yml`:
  - Pass `acrHost`, `imageDigest` (Build stage output), `syftVersion`, `cosignVersion` to `sbom-sign-publish.yml` in the SignAndAttest stage call

## Capabilities

### New Capabilities

- `sbom-generation`: Syft SBOM generation from locally held image in CycloneDX JSON format, retention as named pipeline artifact.
- `image-signing`: Cosign key retrieval from AKV, image signing on manifest digest, SBOM OCI attestation, immediate signature verification.

### Modified Capabilities

- `stage-structure`: SignAndAttest stage call updated — additional parameters passed (`acrHost`, `imageDigest`, `syftVersion`, `cosignVersion`).

## Impact

- **Modified files:** `platform-templates/steps/sbom-sign-publish.yml` (signAndAttest phase implemented; publish/notify remain stubs), `platform-templates/container-build-v2.yml` (SignAndAttest template call updated)
- **Hard dependencies:** Cosign key pair in Azure Key Vault (D-2); platform AKV service connection provisioned; ACR service connection for `<tenantName>/*` push scope (D-1)
- **Produces for Phase 8:** `MANIFEST_DIGEST` step output variable (ACR manifest digest, different from build-time config digest from `--iidfile`)
- **ACR artifacts produced:** `<image>@sha256:<digest>` (primary SHA push), `<image>:<sha>.sig` (Cosign signature), `<image>:<sha>.att` (SBOM attestation)
- **dryRun:** Stage 3 is already conditioned `dryRun=false` in base template — no changes needed; when `dryRun=true` the entire stage is skipped
- **Key hygiene:** Cosign private key written to `/tmp/cosign.key` within the bash step only; removed immediately after use; never emitted as an ADO variable
