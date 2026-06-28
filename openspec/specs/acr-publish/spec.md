## ADDED Requirements

### Requirement: latest tag assertion fires before any push in Stage 4
Before any `docker push` in the `publish` phase, the pipeline SHALL assert that no computed tag equals `latest`. If the naming logic would produce a `latest` tag for any reason, the step SHALL fail with `##vso[task.logissue type=error]` before any push executes.

#### Scenario: Naming logic never produces latest
- **WHEN** the publish step computes the full-SHA, alias, and version tags
- **THEN** none of the tags equals the string `latest`; the assertion passes and pushes proceed

#### Scenario: Assertion fires and blocks push
- **WHEN** (hypothetically) tag computation produces `latest` as any tag value
- **THEN** the step emits an error log and exits non-zero before any `docker push` runs

### Requirement: Full-SHA primary tag pushed to ACR in Stage 4
The `publish` phase SHALL push the locally held image to ACR as `<acrHost>/<tenantName>/<appName>:<git-sha>` where `<git-sha>` is the full 40-character Git commit SHA (`BUILD_SOURCEVERSION`). This tag is the canonical immutable reference for Kustomize manifests and Kyverno verification.

#### Scenario: Primary tag pushed successfully
- **WHEN** the `publish` phase runs with a valid ACR host and tenant/app name
- **THEN** ACR contains `<acrHost>/<tenantName>/<appName>:<full-40-char-sha>` pointing to the signed image manifest

### Requirement: Alias tag pushed to ACR in Stage 4
The `publish` phase SHALL push the image with the alias tag `<branch>-<short-sha>`, where `<branch>` is the sanitized source branch name and `<short-sha>` is the first 12 characters of the Git SHA. This tag is for human navigation only and is always pushed regardless of branch.

#### Scenario: Alias tag pushed on main branch
- **WHEN** the pipeline runs on the main branch
- **THEN** ACR contains `<acrHost>/<tenantName>/<appName>:main-<short-sha>`

#### Scenario: Alias tag pushed on feature branch
- **WHEN** the pipeline runs on a feature branch named `feature/add-auth`
- **THEN** ACR contains `<acrHost>/<tenantName>/<appName>:feature-add-auth-<short-sha>` (slashes sanitized to hyphens)

### Requirement: Version tag pushed conditionally in Stage 4
The `publish` phase SHALL push a version tag derived from the runtime-detected version string:
- On the `main` branch with a non-empty version string: push `<version>` as a bare tag. If that tag already exists in ACR, the step SHALL fail with an error instructing the tenant to bump the version.
- On a non-main branch with a non-empty version string: push `<version>-<short-sha>`. This tag is always unique and MUST NOT fail on existence.
- If the version string is empty (e.g., Go with no `VERSION` file): the version tag SHALL be skipped silently.

#### Scenario: Version tag pushed on main branch — tag absent
- **WHEN** the pipeline runs on main with `runtimeVersion: 1.4.2` and `1.4.2` does not exist in ACR
- **THEN** ACR contains `<acrHost>/<tenantName>/<appName>:1.4.2`

#### Scenario: Version tag push fails on main branch — tag already exists
- **WHEN** the pipeline runs on main with `runtimeVersion: 1.4.2` and `1.4.2` already exists in ACR
- **THEN** the step fails before pushing with `##vso[task.logissue type=error]` instructing the tenant to bump the version; no push occurs

#### Scenario: Version tag pushed on feature branch — always unique
- **WHEN** the pipeline runs on branch `feature/pay` with `runtimeVersion: 1.4.2`
- **THEN** ACR contains `<acrHost>/<tenantName>/<appName>:1.4.2-<short-sha>` regardless of whether `1.4.2` exists in ACR

#### Scenario: Version tag skipped when version string is empty
- **WHEN** the pipeline runs for a Go runtime with no `VERSION` file (empty `runtimeVersion`)
- **THEN** no version tag is pushed; the step proceeds without error

### Requirement: Manifest digest verified after primary tag push
After pushing the full-SHA primary tag, the `publish` phase SHALL verify that the manifest digest returned by ACR matches the `MANIFEST_DIGEST` emitted by the `signAttest` step in Stage 3. A mismatch SHALL fail the step.

#### Scenario: Digest matches — publish continues
- **WHEN** the manifest digest from ACR after the full-SHA push equals `MANIFEST_DIGEST` from Stage 3
- **THEN** the step proceeds to push alias and version tags

#### Scenario: Digest mismatch — publish fails
- **WHEN** the manifest digest from ACR after the full-SHA push does NOT equal `MANIFEST_DIGEST` from Stage 3
- **THEN** the step fails with `##vso[task.logissue type=error]`; alias and version tag pushes do not execute

### Requirement: IMAGE_REF emitted as step output variable
After all pushes succeed, the `publish` phase SHALL emit the full canonical image reference `<acrHost>/<tenantName>/<appName>@<manifestDigest>` as `IMAGE_REF` via `##vso[task.setvariable variable=IMAGE_REF;isOutput=true]`. This variable is the handoff contract for the downstream security scan pipeline.

#### Scenario: IMAGE_REF available to downstream stages
- **WHEN** Stage 4 completes successfully
- **THEN** `$(stageDependencies.Publish.Publish.outputs['publish.IMAGE_REF'])` resolves to a non-empty digest-pinned image reference in Stage 5

### Requirement: Provenance summary published as pipeline artifact
The `publish` phase SHALL write and publish a `provenance.json` artifact via `PublishPipelineArtifact@1` named `provenance-<tenantName>-<appName>`. The JSON SHALL include: `imageRef`, `manifestDigest`, `tags` (array), `sbomArtifact` name, `cosignStatus`, `pipelineRunId`, `acrRepository`, `gitCommit`.

#### Scenario: Provenance artifact present in pipeline run
- **WHEN** Stage 4 completes successfully
- **THEN** the ADO pipeline run's artifacts section contains `provenance-<tenantName>-<appName>` as a downloadable JSON file
