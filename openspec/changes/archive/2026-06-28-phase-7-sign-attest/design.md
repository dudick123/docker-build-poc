## Context

`steps/sbom-sign-publish.yml` is a single template called in three stages (SignAndAttest, Publish, Notify) with a `phase` parameter selecting the active steps. Phase 7 implements the `signAndAttest` phase only; `publish` and `notify` remain stubs until Phase 8.

The template executes in the Build agent context (Stage 3). The locally held image from Stage 2 is present in the Docker daemon. The `IMAGE_DIGEST` output from Stage 2's `buildImage` step is a config digest (from `--iidfile`). Cosign requires the manifest digest — a different value — which is only available after the image is pushed to ACR. Stage 3 therefore must include an initial ACR push before signing can proceed.

Key infrastructure constraints:
- Cosign private key: stored in Azure Key Vault, retrieved at runtime via ADO `AzureKeyVault@2` task; never stored in pipeline variables
- Cosign public key: also in AKV (for verification) or published as a platform ConfigMap/ClusterPolicy per FR-7.6
- ACR service connection: scoped to `<tenantName>/*` push; per-tenant; already required for Stage 4 push (same connection used for Stage 3 initial push)
- Syft and Cosign tool versions: resolved in Stage 1 (Setup), passed as parameters to this template

Stakeholders: platform engineering (key management, template ownership), security/compliance (SBOM audit chain, signing verification), tenant teams (pipeline feedback, provenance summary).

## Goals / Non-Goals

**Goals:**
- Download Syft and Cosign at pinned versions
- Retrieve Cosign private and public keys from AKV at stage start
- Generate CycloneDX JSON SBOM from local image using Syft
- Publish SBOM as named pipeline artifact
- Push image to ACR with primary SHA tag (prerequisite for Cosign signing)
- Sign manifest digest with Cosign using AKV-retrieved private key
- Attach SBOM as OCI attestation via `cosign attest`
- Verify signature before allowing Stage 4 to proceed
- Emit `MANIFEST_DIGEST` (ACR manifest digest) for Phase 8 use

**Non-Goals:**
- Pushing alias or version tags (Phase 8)
- Provenance summary or PR comment (Phase 8)
- SPDX format SBOM (optional per FR-6.2, deferred)
- Keyless / OIDC-based signing (deferred per PRD Appendix C)
- Security scanning (Trivy, Nexus, Fortify — separate pipeline)

## Decisions

### Decision 1: ACR push of primary SHA tag occurs in Stage 3, not Stage 4

The initial `docker push` of the primary SHA-tagged reference (`<acr>/<tenant>/<app>@sha256:<config-digest>`) happens at the start of Stage 3, before SBOM generation. This is necessary because Cosign can only sign images already in a registry — it reads the manifest digest from the registry. Stage 4 pushes the remaining tags (alias, version).

**Rationale:** This is the only practical ordering. The alternative (sign a local image without pushing) would require Cosign's `--attachment-tag-prefix` workaround with a separate signature push — more complex and non-standard. The PRD's "push to ACR only after SBOM generated and signed" refers to the final published state (all tags present, signature verified), not the internal ordering of Stage 3 operations.

**Note:** The `docker push <local-ref>` command pushes layers and returns the manifest digest. This manifest digest (different from the `--iidfile` config digest from Stage 2) is what Cosign operates on.

### Decision 2: Cosign key written to temp file, deleted immediately after signing

The AKV task downloads the Cosign private key secret into pipeline variable `COSIGN_PRIVATE_KEY`. Within the bash step, it is written to `/tmp/cosign.key` (base64-decoded if stored as base64 in AKV), used for signing and attestation, then removed with `rm -f /tmp/cosign.key`. The public key follows the same pattern (`/tmp/cosign.pub`).

**Rationale:** Cosign's `--key` flag accepts a file path. Writing to `/tmp` within the step and deleting immediately keeps the key off disk outside the step. The alternative (`cosign sign --key azurekv://...`) requires Azure SDK dependencies and OIDC credential configuration on the agent — more complex and not compatible with all agent images. The file approach is simpler and equally secure for the v1 implementation.

### Decision 3: SBOM generated from the locally held image, not from source

`syft <local-image-ref>` (using the Docker daemon reference `$TENANT_NAME/$APP_NAME:pipeline-$BUILD_BUILDID`) scans the image that was built — including all installed packages in the base image layers. This runs before the image is pushed to ACR.

**Rationale:** FR-6.3 requires SBOM from the locally held image. Source-only SBOM would miss runtime dependencies from base images (e.g., Alpine packages). The local image reference is available in the Docker daemon throughout Stage 3.

### Decision 4: Signature verification uses public key from AKV

After signing, `cosign verify --key /tmp/cosign.pub <image>@<manifest-digest>` verifies the signature. The public key is also retrieved from AKV via the `AzureKeyVault@2` task (separate secret name). Non-zero exit from verify blocks Stage 4 via `set -euo pipefail`.

**Rationale:** FR-7.5 requires immediate verification before Stage 4. Using the AKV-retrieved public key ensures the same key pair used for signing is used for verification, catching any key retrieval or signing errors before the image is considered published.

### Decision 5: `MANIFEST_DIGEST` emitted as step output for Phase 8

After the initial ACR push, `docker inspect --format='{{index .RepoDigests 0}}' <pushed-ref>` or parsing the push output extracts the manifest digest. This is emitted as `MANIFEST_DIGEST` on a step named `signAttest`. Phase 8 uses this value — not the build-time `IMAGE_DIGEST` — for all downstream references (additional tag pushes, provenance summary).

**Rationale:** The manifest digest is the correct reference for Cosign operations and downstream image verification. The build-time config digest (from `--iidfile`) is not the same value and must not be used for Cosign or ACR-level operations.

### Decision 6: All parameter values via env: block

Consistent with the security pattern established in Phase 2. No `${{ parameters.xxx }}` expressions appear in bash script bodies.

## Risks / Trade-offs

- **AKV key format** — The private key must be stored in AKV in a format Cosign can consume (PEM-encoded PKCS8 or Cosign's own key format). If stored as base64-encoded, the bash step must decode it. Platform engineering must document the expected AKV secret format when provisioning.
- **Manifest digest extraction** — `docker push` output format is not guaranteed stable across Docker versions. Parsing it with `grep` or `awk` may break on version upgrades. Mitigation: use `docker inspect --format='{{index .RepoDigests 0}}'` after push as a more stable alternative; fall back to `skopeo inspect` if available.
- **ACR push in Stage 3** — If Stage 3 fails after the initial push but before signing, the image is in ACR without a signature. Stage 4 is blocked (depends on Stage 3 succeeding). A cleanup mechanism to remove the unsigned image is not in scope for v1 — the image is present but not tagged with any consumer-visible tag.
- **Cosign version compatibility** — Cosign's CLI flags have changed significantly across versions. The implementation targets Cosign v2.x (the version pinned in `platform-tool-versions`). Earlier or later versions may require flag adjustments.

## Open Questions

None. All decisions align with PRD FR-6.x, FR-7.x and the implementation plan Phase 7 scope.
