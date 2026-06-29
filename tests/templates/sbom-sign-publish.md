# Test Cases: steps/sbom-sign-publish.yml

**Template:** `platform-templates/steps/sbom-sign-publish.yml`
**Stages:** Sign & Attest (Stage 3), Publish (Stage 4), Notify (Stage 5)
**Steps under test:** `generateSbom`, `signAttest`, `assertTags`, `publish`, `writeProvenance`, `postPrComment`, `setBuildTag`, `notifyTeams`

---

## Phase: signAndAttest (Stage 3)

---

### TC-SIGN-001 — Happy path: SBOM generated and published as artifact

**Level:** L3
**Type:** Positive

**Precondition:** Stage 2 completed; image exists locally as `platform-test/test-app:pipeline-<id>`.

**Expected result:**
- `/tmp/syft` downloaded at the pinned version
- `sbom.cdx.json` written to `$(System.DefaultWorkingDirectory)/sbom.cdx.json`
- ADO pipeline artifact `sbom-platform-test-test-app` published
- Build log contains: `SBOM generated at sbom.cdx.json`

**SBOM content verification:** Download the artifact and validate:
```bash
jq '.bomFormat' sbom.cdx.json
# Expected: "CycloneDX"
jq '.specVersion' sbom.cdx.json
# Expected: "1.4" or later
jq '.components | length' sbom.cdx.json
# Expected: > 0 (packages detected)
```

---

### TC-SIGN-002 — Syft version is resolved from variable group

**Level:** L3
**Type:** Positive — tool version pin

**Precondition:** `SYFT_VERSION=1.4.1` in `platform-tool-versions`.

**Verification:** Build log contains download URL:
```
https://github.com/anchore/syft/releases/download/v1.4.1/syft_1.4.1_linux_amd64.tar.gz
```
And: `/tmp/syft version` output matches `1.4.1`.

---

### TC-SIGN-003 — Cosign version is resolved from variable group

**Level:** L3
**Type:** Positive — tool version pin

**Precondition:** `COSIGN_VERSION=2.2.4` in `platform-tool-versions`.

**Verification:** Build log contains download URL:
```
https://github.com/sigstore/cosign/releases/download/v2.2.4/cosign-linux-amd64
```

---

### TC-SIGN-004 — Image is pushed to ACR under short-SHA before signing

**Level:** L3
**Type:** Positive — ACR write verification

**Verification:** After Stage 3, confirm that `<acrHost>/platform-test/test-app:<12-char-sha>` exists in ACR.

```bash
az acr repository show-tags \
  --name <acr> \
  --repository platform-test/test-app
# Expected: short-SHA tag appears
```

---

### TC-SIGN-005 — MANIFEST_DIGEST is captured from ACR (not --iidfile)

**Level:** L3
**Type:** Positive — digest correctness

**Verification:** Build log contains:
```
Manifest digest: sha256:<hash>
MANIFEST_DIGEST=sha256:<hash>
```

The `sha256:` hash must be the **manifest digest**, not the config digest from `--iidfile`. Verify by running:
```bash
docker manifest inspect <acrHost>/platform-test/test-app:<short-sha> \
  | jq -r '.config.digest'
# This is the config digest. MANIFEST_DIGEST must differ from this.
# The manifest digest is the sha256 of the manifest JSON itself.
```

---

### TC-SIGN-006 — Cosign signature and attestation present in ACR

**Level:** L3
**Type:** Positive — Cosign output verification

**Verification:** After Stage 3:
```bash
az acr repository show-tags \
  --name <acr> \
  --repository platform-test/test-app
# Expected: tags ending in .sig and .att for the manifest digest
```

Or verify via Cosign:
```bash
cosign verify --key cosign.pub <acrHost>/platform-test/test-app@<MANIFEST_DIGEST>
# Expected: verified OK
```

---

### TC-SIGN-007 — Cosign key files are deleted after step (trap verification)

**Level:** L3
**Type:** Security — key cleanup

**Verification:** After `signAttest` step completes (success or failure), confirm:
```bash
ls /tmp/cosign.key /tmp/cosign.pub
# Expected: ls: No such file or directory
```

To test the failure case: introduce an error that causes the step to fail mid-execution (e.g., an invalid `cosign sign` flag). Confirm key files are still cleaned up via the `trap 'rm -f ...' EXIT` handler.

---

### TC-SIGN-008 — Stage 3 is skipped when dryRun: true

**Level:** L2
**Type:** Positive — dryRun behavior

**Input:** `dryRun: true`

**Expected result:** Stage 3 shows "Skipped" status in the ADO pipeline graph (grey icon). No Syft or Cosign downloads. No ACR writes. `MANIFEST_DIGEST` is NOT emitted.

---

### TC-SIGN-009 — Stage 3 is skipped when Stage 2 fails

**Level:** L2
**Type:** Positive — dependency chain

**Precondition:** Force Stage 2 to fail (e.g., broken Dockerfile).

**Expected result:** Stage 3 shows "Skipped" due to failed upstream dependency. Stage 4 and Stage 5 are also affected accordingly.

---

## Phase: publish (Stage 4)

---

### TC-PUB-001 — Happy path: all three tags pushed, provenance artifact published

**Level:** L3
**Type:** Positive

**Precondition:** Stage 3 completed. `MANIFEST_DIGEST` emitted. `runtimeVersion=1.4.2`. Branch is `main`. Version tag `1.4.2` does not yet exist in ACR.

**Expected result:**
- Three tags pushed to `<acrHost>/platform-test/test-app`:
  - `<40-char-sha>`
  - `main-<12-char-sha>`
  - `1.4.2`
- Build log contains: `Digest verified: sha256:...`
- Build log contains: `IMAGE_REF=<acrHost>/platform-test/test-app@sha256:...`
- `IMAGE_REF` output variable emitted
- ADO artifact `provenance-platform-test-test-app` published

---

### TC-PUB-002 — latest tag prohibition: version file contains 'latest'

**Level:** L3
**Type:** Negative

**Precondition:** Go `VERSION` file contains the string `latest`.

**Expected result:** `assertTags` step fails before any push. Build log contains:
```
##[error]Tag 'latest' equals 'latest'. Pipeline aborted per platform policy (FR-8.4).
```

No tags are pushed to ACR. `publish` step does not run.

---

### TC-PUB-003 — latest tag prohibition: full SHA happens to equal 'latest' (impossible but tested)

**Level:** L1
**Type:** Boundary

**Note:** A Git SHA can never equal the string `latest` (SHA is hex; `latest` contains non-hex characters). This test confirms the assertion logic is not accidentally triggered by the SHA tag.

**Verification (L1):** Trace through `assertTags` with `FULL_SHA=abc123...` (valid 40-char hex). The loop `for TAG in "${CANDIDATE_TAGS[@]}"` checks `if [ "$TAG" = "latest" ]`. A valid SHA will not match.

---

### TC-PUB-004 — latest tag prohibition: branch-sha alias 'main-latest' does not trigger

**Level:** L1
**Type:** Boundary

The alias tag format is `<branch>-<12-char-sha>`. If the branch is `main`, the alias is `main-<sha>`, which is not equal to `latest`. The assertion checks for exact equality (`"$TAG" = "latest"`), not substring match.

**Verification (L1):** `"main-abc123def456" = "latest"` → false. Assertion does not trigger.

---

### TC-PUB-005 — Version tag on main branch, tag already exists: fails

**Level:** L3
**Type:** Negative

**Precondition:** `<acrHost>/platform-test/test-app:1.4.2` already exists in ACR. Pipeline runs on `main` branch with `runtimeVersion=1.4.2`.

**Expected result:** `publish` step fails. Build log contains:
```
##[error]Version tag '1.4.2' already exists in ACR. Bump the version in your project file before merging to main.
```

Full-SHA and alias tags may or may not be pushed before this check depending on step ordering — verify the log to confirm the failure point.

---

### TC-PUB-006 — Version tag on non-main branch: version-sha alias pushed instead

**Level:** L3
**Type:** Positive

**Precondition:** `runtimeVersion=1.4.2`. Branch is `feature/my-change` (not `main`). Tag `1.4.2` may or may not already exist in ACR.

**Expected result:** Three tags pushed:
- `<40-char-sha>`
- `feature-my-change-<12-char-sha>` (slashes converted to hyphens)
- `1.4.2-<12-char-sha>`

`1.4.2` (bare version tag) is NOT pushed. No collision check performed.

---

### TC-PUB-007 — No runtimeVersion: only two tags pushed

**Level:** L3
**Type:** Positive

**Precondition:** `runtimeVersion` is empty (no `VERSION` file / `package.json` version / `pyproject.toml` version).

**Expected result:** Two tags pushed:
- `<40-char-sha>`
- `<branch>-<12-char-sha>`

Build log contains: `No runtime version detected — skipping version tag.`

---

### TC-PUB-008 — Manifest digest mismatch: publish fails

**Level:** L3
**Type:** Negative

**Scenario:** Stage 3 emits `MANIFEST_DIGEST=sha256:aaa...`. After Stage 4 pushes the full-SHA tag and inspects the result, the digest differs (simulated by a race condition or tampered image).

**Expected result:** `publish` step fails. Build log contains:
```
##[error]Manifest digest mismatch. Expected: sha256:aaa... Got: sha256:bbb...
```

**Note:** This scenario is difficult to reproduce organically in a controlled test. Consider mocking by modifying the `MANIFEST_DIGEST` parameter to a known-wrong value.

---

### TC-PUB-009 — Provenance JSON has correct structure

**Level:** L3
**Type:** Positive — artifact content

**Verification:** Download `provenance-platform-test-test-app` artifact and validate:

```bash
jq '{
  imageRef: .imageRef,
  manifestDigest: .manifestDigest,
  tagCount: (.tags | length),
  sbomArtifact: .sbomArtifact,
  cosignStatus: .cosignStatus,
  gitCommitLength: (.gitCommit | length)
}' provenance.json
```

**Expected:**
- `imageRef`: non-empty, format `<acr>/...@sha256:...`
- `manifestDigest`: `sha256:<hash>`
- `tagCount`: 2 or 3 depending on version presence
- `sbomArtifact`: `sbom-platform-test-test-app`
- `cosignStatus`: `"signed"`
- `gitCommitLength`: `40`

---

### TC-PUB-010 — Stage 4 is skipped when dryRun: true

**Level:** L2
**Type:** Positive

**Input:** `dryRun: true`

**Expected result:** Stage 4 shows "Skipped" in ADO pipeline graph. No ACR writes. `IMAGE_REF` NOT emitted.

---

## Phase: notify (Stage 5)

---

### TC-NOTIFY-001 — PR build: comment posted with Succeeded status

**Level:** L3
**Type:** Positive

**Precondition:** Pipeline triggered by a PR. `IMAGE_REF` is set (Stage 4 succeeded).

**Expected result:** A comment appears on the PR with a Markdown table containing:
- Status: `✅ Succeeded`
- Tenant/App: `platform-test/test-app`
- Image Ref: non-empty
- Manifest Digest: `sha256:...`
- Cosign Status: `signed ✅`
- SBOM Artifact: `sbom-platform-test-test-app`

---

### TC-NOTIFY-002 — Non-PR build: comment step skipped gracefully

**Level:** L2
**Type:** Positive

**Precondition:** Pipeline triggered manually (not by a PR). `BUILD_REASON=Manual`.

**Expected result:** `postPrComment` step passes (exit 0). Build log contains:
```
Skipping PR comment: not a pull request build (BUILD_REASON=Manual)
```

No API call is made to the ADO PR threads endpoint.

---

### TC-NOTIFY-003 — dryRun: true — comment shows Dry Run status

**Level:** L2
**Type:** Positive

**Precondition:** Pipeline triggered by a PR with `dryRun: true`. `IMAGE_REF` is empty (Stage 4 skipped).

**Expected result:** PR comment posted with:
- Status: `🔵 Dry Run`
- Detail: `Dry run — no image was pushed to ACR and the security scan pipeline was not triggered.`
- Image Ref row: NOT present
- Manifest Digest row: NOT present

---

### TC-NOTIFY-004 — Stage 3/4 failed: comment shows Failed status

**Level:** L3
**Type:** Positive (failure path)

**Precondition:** Pipeline triggered by PR. Stage 3 or Stage 4 failed. `IMAGE_REF` is empty. `dryRun` is false.

**Expected result:** PR comment posted with:
- Status: `❌ Failed`
- Detail: `Pipeline failed before image publish. Check the pipeline run for details.`

---

### TC-NOTIFY-005 — Build tag set to MANIFEST_DIGEST on full run

**Level:** L3
**Type:** Positive

**Precondition:** `MANIFEST_DIGEST=sha256:a3f9e21b...` emitted by Stage 3.

**Expected result:** ADO pipeline run has build tag `sha256:a3f9e21b...` visible in the pipeline run detail view.

**Verification:** ADO UI → Pipeline Run → Tags. Or via REST API:
```
GET <collection>/<project>/_apis/build/builds/<buildId>/tags?api-version=7.1
```

---

### TC-NOTIFY-006 — Build tag set to dryrun-<buildId> on dry run

**Level:** L2
**Type:** Positive

**Input:** `dryRun: true` → `MANIFEST_DIGEST` is empty.

**Expected result:** ADO pipeline run has build tag `dryrun-<BUILD_BUILDID>`.

---

### TC-NOTIFY-007 — Teams notification sent when TEAMS_WEBHOOK_URL is set

**Level:** L3
**Type:** Positive

**Precondition:** `tenant-platform-test-notifications` variable group exists with `TEAMS_WEBHOOK_URL` set to a valid Teams incoming webhook URL.

**Expected result:** `notifyTeams` step passes. Build log contains: `Teams notification sent`. An Adaptive Card appears in the configured Teams channel with the pipeline run status and a "View Pipeline Run" button.

---

### TC-NOTIFY-008 — Teams notification skipped when TEAMS_WEBHOOK_URL is absent

**Level:** L2
**Type:** Positive (graceful skip)

**Precondition:** `TEAMS_WEBHOOK_URL` variable is not set (variable group absent or key missing).

**Expected result:** `notifyTeams` step passes (exit 0). Build log contains:
```
Warning: TEAMS_WEBHOOK_URL not set for tenant — skipping Teams notification.
```

Pipeline is not blocked.

---

### TC-NOTIFY-009 — Notify stage runs when Publish is skipped (dryRun: true)

**Level:** L2
**Type:** Positive — stage condition coverage

**Scenario:** `dryRun: true` causes Stage 4 to be in `Skipped` state. Stage 5 uses:
```yaml
condition: in(dependencies.Publish.result, 'Succeeded', 'Failed', 'SucceededWithIssues', 'Skipped')
```

**Expected result:** Stage 5 (Notify) runs and completes. The build log shows the dry-run messaging. This test specifically validates the `in(...)` condition rather than `succeededOrFailed()` — the latter would NOT trigger when Stage 4 is Skipped.

**Verification:** Confirm Stage 5 appears as "Succeeded" (not "Skipped") in the ADO pipeline graph when `dryRun: true`.
