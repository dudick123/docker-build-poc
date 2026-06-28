## ADDED Requirements

### Requirement: SBOM generated from locally held image in CycloneDX JSON format
The `signAndAttest` phase of `steps/sbom-sign-publish.yml` SHALL generate an SBOM using Syft at the pinned version resolved from `platform-tool-versions`. The SBOM SHALL be generated from the locally held image in the Docker daemon (not from source), SHALL be in CycloneDX JSON format (`-o cyclonedx-json`), and SHALL be written to `sbom.cdx.json` in the working directory.

#### Scenario: SBOM generated from local image
- **WHEN** the signAndAttest phase runs against a successfully built local image
- **THEN** Syft scans the image layers (including base image packages) and produces `sbom.cdx.json` in CycloneDX JSON format

#### Scenario: SBOM includes base image packages
- **WHEN** the image is built on `alpine:3.19` with additional packages installed
- **THEN** the SBOM includes both the Alpine base packages and any packages installed by the Dockerfile's `RUN` steps

### Requirement: SBOM published as a named pipeline artifact
After generation, the SBOM file SHALL be published as an ADO pipeline artifact named `sbom-<tenantName>-<appName>` using the `PublishPipelineArtifact` task. The artifact SHALL be retained according to the pipeline's artifact retention policy (independent of whether the image push succeeds).

#### Scenario: SBOM artifact retained after successful pipeline run
- **WHEN** the full pipeline completes successfully
- **THEN** the SBOM is available in the ADO pipeline run's artifacts section as `sbom-<tenantName>-<appName>`

#### Scenario: SBOM artifact retained even if Stage 4 fails
- **WHEN** Stage 3 (Sign & Attest) completes but Stage 4 (Publish) fails
- **THEN** the SBOM artifact published in Stage 3 is still accessible in the pipeline run artifacts
