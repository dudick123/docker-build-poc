## ADDED Requirements

### Requirement: PR comment posted with build provenance summary
The `notify` phase SHALL post a comment to the ADO pull request that triggered the build. The comment SHALL include: build status (success/failure), `runtimeType`, image digest (`sha256:...`), ACR image reference, list of tags pushed, SBOM artifact name and location, Cosign signing status, and a note that Trivy/Nexus/Fortify security scanning runs in a separate advisory-only pipeline. On pipeline failure, the failing stage and reason SHALL be included. If `Build.Reason` is not `PullRequest`, the PR comment step SHALL skip gracefully with a log message (non-blocking, exit zero).

#### Scenario: PR comment posted on successful build
- **WHEN** the pipeline completes Stage 4 successfully and `Build.Reason == PullRequest`
- **THEN** a comment appears on the triggering ADO PR containing the image digest, ACR reference, tags pushed, and security scan advisory note

#### Scenario: PR comment skipped on non-PR build
- **WHEN** the pipeline is triggered by a direct push to main (not a PR)
- **THEN** the PR comment step logs "Skipping PR comment: not a pull request build" and exits zero

#### Scenario: PR comment includes failure details on pipeline failure
- **WHEN** Stage 3 or Stage 4 failed and the Notify stage runs via `succeededOrFailed()`
- **THEN** the PR comment includes the failing stage name and the pipeline run URL

### Requirement: ADO pipeline run tagged with image digest
The `notify` phase SHALL set an ADO pipeline run build tag equal to the manifest digest using the logging command `echo "##vso[build.addbuildtag]<manifestDigest>"`. This enables correlation between pipeline runs and ACR image manifests from the ADO UI.

#### Scenario: Build tag visible in ADO pipeline run
- **WHEN** the notify phase runs on a successful build
- **THEN** the ADO pipeline run UI shows the manifest digest as a build tag

#### Scenario: Build tag set even on dry run
- **WHEN** the pipeline runs with `dryRun=true` and no image was pushed
- **THEN** the build tag is set to `dryrun-<pipelineRunId>` to indicate a dry run (no manifest digest available)

### Requirement: Teams webhook notification sent on build completion
The `notify` phase SHALL send a Teams channel notification via webhook. The webhook URL SHALL be read from `$(TEAMS_WEBHOOK_URL)` in the per-tenant variable group `tenant-<tenantName>-notifications`. If `TEAMS_WEBHOOK_URL` is empty or unset, the step SHALL emit a warning log and exit zero (non-blocking). The notification payload SHALL include build status, image reference (on success), pipeline run URL, and `runtimeType`.

#### Scenario: Teams notification sent on success
- **WHEN** the pipeline completes successfully and `TEAMS_WEBHOOK_URL` is set
- **THEN** a Teams Adaptive Card message appears in the configured channel with a green status indicator and image reference

#### Scenario: Teams notification sent on failure
- **WHEN** the pipeline fails at any stage and `TEAMS_WEBHOOK_URL` is set
- **THEN** a Teams Adaptive Card message appears in the configured channel with a red status indicator and the failing stage name

#### Scenario: Teams notification skipped when webhook not configured
- **WHEN** `TEAMS_WEBHOOK_URL` is empty or unset for the tenant
- **THEN** the step logs a warning "TEAMS_WEBHOOK_URL not set for tenant — skipping Teams notification" and exits zero

### Requirement: Notify stage runs regardless of upstream success or failure
The `notify` phase SHALL execute whether Stages 3 and 4 succeeded, failed, or were skipped (dryRun). The base template's `Notify` stage condition SHALL be `succeededOrFailed()` applied to the `Publish` stage.

#### Scenario: Notify runs after dry run (Publish skipped)
- **WHEN** `dryRun=true` causes Publish to be skipped
- **THEN** the Notify stage runs and the PR comment and Teams message indicate a dry-run build (no image pushed)

#### Scenario: Notify runs after Publish failure
- **WHEN** Stage 4 (Publish) fails
- **THEN** the Notify stage runs, the PR comment includes the failure reason, and Teams receives a failure notification
