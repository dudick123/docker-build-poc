## Context

Stage 3 (`signAndAttest`) pushes the image to ACR under a 12-char short-SHA tag, captures the manifest digest, signs the digest with Cosign, and attaches the SBOM attestation. It emits `MANIFEST_DIGEST` as a step output variable. Stages 4 and 5 are stubs.

Stage 4 (`publish`) must push the remaining tags (full 40-char SHA primary, alias, version), assert `latest` is never pushed, verify the manifest digest is unchanged, and emit the canonical `IMAGE_REF` for downstream consumers. Stage 5 (`notify`) must post the PR comment and Teams webhook, and tag the ADO run.

The version string to use in the version tag was detected by the runtime step templates in Stage 2 and emitted as runtime-specific output variables (`GO_VERSION`, `PYTHON_VERSION`, etc.). Stage 4 needs a single resolved version string; the base template must select the right output by runtime type and forward it as `runtimeVersion`.

## Goals / Non-Goals

**Goals:**
- Push full SHA, alias, and version tags from Stage 4; version tag conditional on branch and runtime version availability
- Assert `latest` tag prohibition before any push in Stage 4
- Verify manifest digest post-push matches `MANIFEST_DIGEST` from Stage 3
- Emit `IMAGE_REF` as `isOutput=true` for downstream pipelines (security scan)
- Publish provenance summary as pipeline artifact (JSON) in Stage 4
- Post ADO PR comment with provenance summary in Stage 5 (skip gracefully on non-PR builds)
- Post Teams webhook notification in Stage 5 using per-tenant variable group
- Set ADO pipeline run build tag to image digest in Stage 5

**Non-Goals:**
- Triggering the downstream security scan pipeline directly — per PRD FR-5.2, the security scan pipeline owns the trigger via ADO pipeline completion trigger, not this pipeline
- Verifying Cosign signature in Stage 4 — already done in Stage 3; re-verification is redundant
- Pushing any tags in Stage 3 — Stage 3 pushes only the short-SHA tag needed for signing; Stage 4 owns all additional tags

## Decisions

**D-1: Version tag selection by runtimeType at compile time**
The base template uses `${{ if eq(parameters.runtimeType, 'go') }}` compile-time dispatch to select the correct stage output variable and pass it as `runtimeVersion` to the Publish template call. This avoids a runtime bash conditional over an unknown variable set. Each runtime has a distinct output variable name (`GO_VERSION`, `PYTHON_VERSION`, `SPRINGBOOT_VERSION`, `ANGULAR_VERSION`, `REACT_VERSION`).

**D-2: Version tag conditional on branch + version availability**
Inside the `publish` bash step:
- If `runtimeVersion` is empty (e.g., Go with no `VERSION` file): skip version tag silently.
- If on `main` branch and version tag already exists in ACR: fail the step with `##vso[task.logissue type=error]` before pushing.
- If on `main` branch and tag does not exist: push `$ACR_HOST/$TENANT_NAME/$APP_NAME:$RUNTIME_VERSION`.
- If on non-main branch: push `$ACR_HOST/$TENANT_NAME/$APP_NAME:$RUNTIME_VERSION-$SHORT_SHA` (always safe to overwrite).

Check tag existence via `docker manifest inspect "$ACR_HOST/$TENANT_NAME/$APP_NAME:$RUNTIME_VERSION"` — exit 0 means tag exists, non-zero means absent.

**D-3: `latest` tag prohibition enforced as pre-push assertion**
Before any `docker push` in Stage 4, assert that neither `$BUILD_SOURCEVERSION` (full SHA) nor the computed alias or version tags equal `latest`. This is a belt-and-suspenders check since the naming logic makes `latest` impossible, but the PRD mandates the assertion.

**D-4: Digest verification via `docker manifest inspect`**
After Stage 4 pushes the full SHA tag, run `docker manifest inspect "$ACR_HOST/$TENANT_NAME/$APP_NAME:$BUILD_SOURCEVERSION"` and extract the `config.digest` to compare against `MANIFEST_DIGEST` from Stage 3. A mismatch fails the step.

**D-5: Provenance summary as JSON artifact**
The provenance summary is written as `provenance.json` in the working directory and published via `PublishPipelineArtifact@1`. Fields: `imageRef`, `manifestDigest`, `tags`, `sbomArtifact`, `cosignStatus`, `pipelineRunId`, `acrRepository`, `gitCommit`. This machine-readable format is consumed by the PR comment step and the security scan pipeline handoff.

**D-6: PR comment via ADO REST API using System.AccessToken**
Post the PR comment using a `curl` call to the ADO REST API: `POST /_apis/git/repositories/{repoId}/pullRequests/{prId}/threads`. The PR ID is available as `$(System.PullRequest.PullRequestId)`. If `Build.Reason` is not `PullRequest`, the step skips with a log message (no error).

**D-7: Teams notification via per-tenant variable group**
The Teams webhook URL is read from `$(TEAMS_WEBHOOK_URL)` which is expected in a per-tenant variable group (`tenant-<tenantName>-notifications`). If the variable is empty, the step logs a warning and exits zero (non-blocking). The payload is a simple Adaptive Card with build status, image ref, and run URL.

**D-8: ADO build tag via `##vso[build.addbuildtag]`**
Set the ADO pipeline run tag using the logging command `echo "##vso[build.addbuildtag]$MANIFEST_DIGEST"`. This creates a searchable build tag in ADO.

## Risks / Trade-offs

- **[Risk] `docker manifest inspect` for tag existence check requires ACR auth** — The pipeline agent must be authenticated to ACR before Stage 4 runs. The Stage 3 push (`docker push`) already requires this auth; if Stage 3 succeeded, auth is established. Mitigation: document the dependency; no additional work needed.
- **[Risk] Version tag collision on main branch fails the pipeline intentionally** — This is the desired behavior (FR-8.3), but tenant teams may not expect it. Mitigation: emit a clear `##vso[task.logissue type=error]` message directing them to bump their version.
- **[Risk] Teams webhook URL absent for a tenant** — If the per-tenant variable group is not configured, Teams notification silently skips. Mitigation: warning log is emitted; the step is non-blocking by design.
- **[Risk] `MANIFEST_DIGEST` mismatch after Stage 4 push** — Theoretically impossible if the same image layers are pushed under a new tag. In practice it cannot happen unless ACR mutates the manifest, which is not expected. Mitigation: the verification is a safety gate; a mismatch is treated as a platform incident.

## Open Questions

- None. The PRD requirements for Stage 4 (FR-8.1–FR-8.7) and Stage 5 (FR-10.1–FR-10.4) are fully specified.
