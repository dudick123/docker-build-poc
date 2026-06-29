# Quick Start: Container Build Pipeline v2

This guide walks a tenant team through onboarding an application to the shared container build pipeline. It covers prerequisites, the minimal `azure-pipelines.yml`, what the pipeline produces, and common failure modes.

For runtime-specific Dockerfile requirements, see the guide for your language:

- [Go](QUICK-START-GO.md)
- [Python](QUICK-START-PYTHON.md)
- [Angular](QUICK-START-ANGULAR.md)
- [React / Next.js](QUICK-START-REACT.md)
- [Spring Boot (Gradle)](QUICK-START-GRADLE.md)
- [Spring Boot (Maven)](QUICK-START-MAVEN.md)

---

## Prerequisites

Before you add a pipeline file, confirm the following with your platform engineering contact:

| Item | What to check |
|---|---|
| ADO template repository | `platform-templates` repository exists in your ADO organization and is referenced as a resource named `platform-templates` in your project |
| Tenant service connection | An ADO service connection scoped to `<tenantName>/*` in the shared ACR exists for your team |
| `platform-tool-versions` variable group | The group exists and is accessible to your ADO project |
| Cosign AKV variables | `COSIGN_AKV_SERVICE_CONNECTION` and `COSIGN_KEY_VAULT_NAME` are set in your pipeline environment |

If any of these are missing, open a platform engineering ticket before proceeding.

---

## Step 1 — Add azure-pipelines.yml

Create `azure-pipelines.yml` at the root of your application repository:

```yaml
trigger:
  branches:
    include:
      - main

resources:
  repositories:
    - repository: platform-templates
      type: git
      name: <YourADOProject>/platform-templates

extends:
  template: container-build-v2.yml@platform-templates
  parameters:
    tenantName: my-team          # lowercase, hyphens allowed, min 2 chars
    appName: my-service          # lowercase, hyphens allowed, min 2 chars
    runtimeType: go              # go | python | springboot | angular | react
```

That is the complete pipeline file for the common case. The platform controls everything else.

### Parameter rules

`tenantName` and `appName` must match `^[a-z0-9][a-z0-9-]*[a-z0-9]$`:
- Lowercase alphanumeric characters and interior hyphens only
- Minimum two characters
- Cannot start or end with a hyphen

Examples that pass: `payments`, `api-gateway`, `data-platform`
Examples that fail: `Payments`, `api_gateway`, `a`, `-payments`

---

## Step 2 — Add your Dockerfile

The pipeline builds whatever Dockerfile it finds at `<buildContext>/<dockerfilePath>`. The defaults are `buildContext: .` and `dockerfilePath: Dockerfile`, so a `Dockerfile` at the repository root requires no extra parameters.

The Dockerfile must pass Hadolint at the ERROR level before the image build runs. Common Hadolint rules that cause hard failures:

- `ARG` or `ENV` used to pass secrets (use BuildKit secrets instead)
- `ADD` used where `COPY` is appropriate
- `apt-get install` without pinned versions in some configurations

To suppress a specific rule project-wide, create `.hadolint.yaml` in your build context:

```yaml
ignore:
  - DL3008   # Allow unpinned apt packages (example only — prefer pinning)
```

See the [runtime-specific guides](#runtime-specific-guides) for reference Dockerfile examples.

---

## Step 3 — Optional: Dockerfile location

If your Dockerfile is not at the repository root, set `buildContext` and/or `dockerfilePath`:

```yaml
extends:
  template: container-build-v2.yml@platform-templates
  parameters:
    tenantName: my-team
    appName: my-service
    runtimeType: python
    buildContext: services/api    # Docker build context directory
    dockerfilePath: Dockerfile    # relative to buildContext
```

The pipeline resolves the Dockerfile at `<buildContext>/<dockerfilePath>` on the agent workspace. If that path does not exist, Stage 1 (Setup) fails with a clear error before any build work starts.

---

## Step 4 — Optional: Teams notifications

If your team wants a Teams channel notification on every build, add a variable group named `tenant-<tenantName>-notifications` to your ADO project and set:

```
TEAMS_WEBHOOK_URL = https://your-org.webhook.office.com/webhookb2/...
```

The Notify stage reads this value automatically. If the variable is absent or empty, Teams notification is silently skipped — it is never a blocking condition.

---

## Step 5 — Verify the pipeline

Queue the pipeline manually for the first run, or open a PR to main to trigger it automatically.

### Dry run first

For your first run, set `dryRun: true` to verify lint and build without touching ACR:

```yaml
extends:
  template: container-build-v2.yml@platform-templates
  parameters:
    tenantName: my-team
    appName: my-service
    runtimeType: go
    dryRun: true
```

With `dryRun: true`:
- Stage 1 (Setup) and Stage 2 (Build) run normally
- Stage 3 (Sign & Attest) and Stage 4 (Publish) are skipped — no writes to ACR
- Stage 5 (Notify) runs and posts a "Dry Run" PR comment

Once the dry run passes, remove `dryRun: true` (or leave it out entirely — the default is `false`).

---

## What the Pipeline Produces

A successful (non-dry-run) run produces:

### Image tags in ACR

All tags land at `<acr>/<tenantName>/<appName>`:

| Tag | Example | Use |
|---|---|---|
| Full SHA (primary) | `a3f9e21b7c...` (40 chars) | Use in Kustomize manifests |
| Alias | `main-a3f9e21b7c12` | Human navigation, CI dashboards |
| Version (main branch) | `1.4.2` | Omitted if no version file |
| Version (other branches) | `1.4.2-a3f9e21b7c12` | Omitted if no version file |

The `latest` tag is never pushed. The pipeline asserts this before any push and fails hard if any computed tag equals `latest`.

### ADO pipeline artifacts

| Artifact | What it contains |
|---|---|
| `sbom-<tenantName>-<appName>` | CycloneDX JSON SBOM of every package in the image |
| `provenance-<tenantName>-<appName>` | JSON with digest, tags, SBOM artifact name, git commit, cosign status |

### Cosign objects in ACR

| Object | Tag suffix | Description |
|---|---|---|
| Signature | `<digest>.sig` | Detached Cosign signature of the manifest digest |
| Attestation | `<digest>.att` | SBOM attached as OCI attestation |

### PR comment

If the pipeline was triggered by a pull request, Stage 5 posts a Markdown summary table to the PR thread showing status, tenant/app, runtime, image ref, manifest digest, Cosign status, and SBOM artifact name.

### ADO build tag

The manifest digest (`sha256:...`) is set as an ADO pipeline run build tag, linking the pipeline run to the exact image it produced.

---

## Version Tagging

The version tag is derived from your project metadata file — the pipeline never reads the Git tag. The extraction logic per runtime:

| Runtime | Source file | What is extracted |
|---|---|---|
| Go | `VERSION` (in build context root) | First non-empty line, whitespace trimmed |
| Python | `pyproject.toml` | `version = "x.y.z"` under `[project]` or `[tool.poetry]`; falls back to `setup.cfg` |
| Spring Boot (Gradle) | `build.gradle` or `gradle.properties` | `version = 'x.y.z'` or `version=x.y.z` |
| Spring Boot (Maven) | `pom.xml` | First `<version>` element |
| Angular | `package.json` | `"version": "x.y.z"` field |
| React | `package.json` | `"version": "x.y.z"` field |

If no version is found, the version tag is skipped — the image is still published with the full-SHA and alias tags. If the version is found and the branch is `main`, the pipeline checks whether that tag already exists in ACR and fails if it does (prevents silent overwrites; bump the version in your project file before merging).

---

## Common Failures and Fixes

### Stage 1 (Setup) failures

**`tenantName 'X' does not match required pattern`**
Rename the value to lowercase alphanumeric with hyphens only. No underscores, no uppercase, minimum two characters.

**`runtimeType 'X' is not supported`**
The value must be exactly one of: `go`, `python`, `springboot`, `angular`, `react`.

**`dockerfilePath not found: /agent/...`**
The pipeline resolved `<buildContext>/<dockerfilePath>` and did not find a file there. Check that the path is correct relative to the repository root and that the Dockerfile is committed.

**`Variable ACR_HOST is missing or empty in platform-tool-versions`**
The `platform-tool-versions` variable group is not configured or is not accessible to your pipeline. Contact platform engineering.

---

### Stage 2 (Build) failures

**Hadolint error — `DL3020: Use COPY instead of ADD`**
Replace `ADD` with `COPY` in your Dockerfile, or add a `.hadolint.yaml` to ignore specific rules. Note: Hadolint errors block the build; warnings do not.

**`go.mod not found`**
For `runtimeType: go`, a `go.mod` file must exist at the build context root. This is a hard fail. See [QUICK-START-GO.md](QUICK-START-GO.md).

**`Dockerfile does not contain a 'test-export' stage`**
For `runtimeType: springboot`, the Dockerfile must have a stage named `test-export` (e.g., `FROM scratch AS test-export`). See [QUICK-START-GRADLE.md](QUICK-START-GRADLE.md) or [QUICK-START-MAVEN.md](QUICK-START-MAVEN.md).

**Docker build fails**
Check the build log for the underlying error. Common causes: base image pull failures (check network/proxy config), missing `COPY` source files, build argument issues.

---

### Stage 3 (Sign & Attest) failures

**AzureKeyVault@2 task fails**
The `COSIGN_AKV_SERVICE_CONNECTION` or `COSIGN_KEY_VAULT_NAME` variable is missing, the service connection has insufficient permissions, or the secrets `cosign-private-key` / `cosign-public-key` do not exist in the vault. Contact platform engineering.

**`cosign verify` exits non-zero**
Signing succeeded but verification failed — this indicates a key mismatch. Contact platform engineering; do not attempt to bypass this check.

---

### Stage 4 (Publish) failures

**`Tag 'X' equals 'latest'. Pipeline aborted`**
Your version file contains the string `latest`. Change your project version to a valid semver string.

**`Version tag 'X' already exists in ACR`**
On a main branch merge, the pipeline found an existing ACR tag matching your version string. Bump the version in your project file (`build.gradle`, `pyproject.toml`, `package.json`, `VERSION`, etc.) and re-run.

**Manifest digest mismatch**
The digest of the primary SHA tag does not match the digest captured during Stage 3. This should not occur under normal conditions. Contact platform engineering if you see this consistently.
