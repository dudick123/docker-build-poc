## ADDED Requirements

### Requirement: Cosign private key retrieved from Azure Key Vault at Stage 3 start
The `signAndAttest` phase SHALL retrieve the Cosign private key and public key from platform-managed Azure Key Vault using the ADO `AzureKeyVault@2` task before any signing operation. The key vault name, secret names, and service connection SHALL be platform-controlled values (from `platform-tool-versions` variable group or hardcoded in the template). The private key SHALL NOT be stored in ADO pipeline variables, variable groups, or any location accessible to tenant teams beyond the duration of the signing step.

#### Scenario: Keys retrieved successfully
- **WHEN** the `AzureKeyVault@2` task runs with valid platform service connection and vault configuration
- **THEN** `COSIGN_PRIVATE_KEY` and `COSIGN_PUBLIC_KEY` are available as pipeline secret variables for the duration of the stage

#### Scenario: AKV retrieval fails
- **WHEN** the AKV task cannot access the specified vault (misconfigured service connection, vault unavailable)
- **THEN** the task fails; Stage 3 fails; Stage 4 does not run; no unsigned image is published

### Requirement: Image pushed to ACR with primary SHA tag before signing
The `signAndAttest` phase SHALL push the locally held image to ACR as `<acrHost>/<tenantName>/<appName>:<sha>` (where `<sha>` is the first 12 characters of the full Git SHA) before invoking Cosign. The manifest digest returned by the ACR push SHALL be captured and emitted as `MANIFEST_DIGEST` on the step named `signAttest`.

#### Scenario: Image pushed and manifest digest captured
- **WHEN** `docker push` of the local image reference to ACR completes successfully
- **THEN** the ACR-returned manifest digest (`sha256:<64-hex-chars>`) is captured and set as the `MANIFEST_DIGEST` step output variable

#### Scenario: Push fails — signing does not proceed
- **WHEN** `docker push` to ACR fails (ACR unavailable, auth failure, network error)
- **THEN** the step exits non-zero; Cosign signing is not invoked; Stage 4 is blocked

### Requirement: Image manifest digest is signed with Cosign using AKV private key
The `signAndAttest` phase SHALL sign the image manifest digest using `cosign sign --key <keyfile> <acr>/<tenant>/<app>@<manifest-digest>`. Signing SHALL operate on the manifest digest, not a tag. The signed artifact SHALL be stored in ACR as `<image>:<sha>.sig`.

#### Scenario: Image signed successfully
- **WHEN** Cosign signs the manifest digest with the private key
- **THEN** ACR contains `<image>:<sha>.sig`; `cosign verify` with the corresponding public key exits zero

#### Scenario: Signing fails
- **WHEN** Cosign exits non-zero during signing (bad key, ACR write failure)
- **THEN** the step exits non-zero; attestation and verification steps do not run; Stage 4 is blocked

### Requirement: SBOM attached as OCI attestation using cosign attest
The `signAndAttest` phase SHALL attach `sbom.cdx.json` as an OCI attestation using `cosign attest --key <keyfile> --predicate sbom.cdx.json --type cyclonedx <acr>/<tenant>/<app>@<manifest-digest>`. The attestation SHALL be stored in ACR as `<image>:<sha>.att`.

#### Scenario: SBOM attestation attached successfully
- **WHEN** `cosign attest` completes with exit 0
- **THEN** ACR contains `<image>:<sha>.att`; the attestation is verifiable with `cosign verify-attestation`

### Requirement: Cosign signature verified immediately after signing
The `signAndAttest` phase SHALL run `cosign verify --key <pubkeyfile> <acr>/<tenant>/<app>@<manifest-digest>` immediately after signing. A non-zero exit from verify SHALL block Stage 4 from running.

#### Scenario: Verification passes — Stage 4 proceeds
- **WHEN** `cosign verify` exits zero
- **THEN** Stage 3 completes successfully; Stage 4 may proceed

#### Scenario: Verification fails — Stage 4 blocked
- **WHEN** `cosign verify` exits non-zero (signature not found, key mismatch)
- **THEN** Stage 3 fails; Stage 4 is blocked via the `dependsOn: SignAndAttest` + `condition: succeeded()` guard in the base template

### Requirement: Cosign private key not persisted beyond the signing step
The Cosign private key MAY be written to a temporary file (e.g., `/tmp/cosign.key`) within the signing bash step. If written to disk, it SHALL be deleted with `rm -f` in the same step before the step exits — whether the step succeeds or fails. The key SHALL NOT be emitted as an ADO output variable.

#### Scenario: Key file cleaned up on success
- **WHEN** signing and attestation complete successfully
- **THEN** `rm -f /tmp/cosign.key /tmp/cosign.pub` removes the key files before the step exits

#### Scenario: Key file cleaned up on failure
- **WHEN** signing fails mid-step
- **THEN** a `trap 'rm -f /tmp/cosign.key /tmp/cosign.pub' EXIT` ensures cleanup regardless of exit code
