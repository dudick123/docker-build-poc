# Template: steps/docker-build.yml

**Path:** `platform-templates/steps/docker-build.yml`
**Type:** Step template
**Called by:** `container-build-v2.yml` — Stage 2 (Build), Job: Build (second step, after dockerfile-lint)

---

## Description

Builds the container image using Docker BuildKit and holds it locally on the agent. The image is not pushed to ACR here — it remains agent-local until Stage 3 (Sign & Attest), where it is pushed under a short-SHA staging tag to obtain the manifest digest for signing.

Four standard OCI image labels are injected at build time. An ACR layer cache is configured to accelerate repeat builds. npm credentials for the platform Azure Artifacts feed are injected for all runtimes via a BuildKit secret — Dockerfiles that do not use npm simply ignore these values.

---

## Summary

| Property | Value |
|---|---|
| Steps | 1 (bash, named `buildImage`) |
| BuildKit required | Yes — `DOCKER_BUILDKIT=1` is set on every invocation |
| Image location after step | Agent-local only; not pushed to ACR |
| Image local tag | `<tenantName>/<appName>:pipeline-<BUILD_BUILDID>` |
| ACR layer cache | `<acrHost>/<tenantName>/<appName>:buildcache` (mode=max) |
| Output variables | `IMAGE_DIGEST` (config digest from `--iidfile`) |
| npm credentials | Injected for all runtimes via `--build-arg` and `--secret` |

---

## Parameters

| Name | Type | Default | Description |
|---|---|---|---|
| `tenantName` | string | — | Used in local image tag and ACR cache ref |
| `appName` | string | — | Used in local image tag and ACR cache ref |
| `dockerfilePath` | string | `Dockerfile` | Path to Dockerfile, relative to `buildContext` |
| `buildContext` | string | `.` | Docker build context directory |
| `acrHost` | string | — | FQDN of the shared ACR; sourced from `resolveTools.ACR_HOST` |
| `npmRegistryUrl` | string | — | Platform npm feed URL; sourced from `resolveTools.NPM_REGISTRY_URL` |

---

## Steps

### Step 1 — `buildImage` (bash)

Constructs the local image tag and cache reference, then runs a full BuildKit build:

**Local image tag:**
```
<tenantName>/<appName>:pipeline-<BUILD_BUILDID>
```
This tag is referenced by all subsequent steps and stages. The `BUILD_BUILDID` suffix ensures uniqueness if multiple builds run concurrently on the same agent.

**BuildKit invocation:**

```bash
DOCKER_BUILDKIT=1 docker build \
  -f "$BUILD_CONTEXT/$DOCKERFILE_PATH" \
  "$BUILD_CONTEXT" \
  --iidfile "$IIDFILE" \
  -t "$LOCAL_IMAGE_REF" \
  --build-arg "GIT_COMMIT_SHA=$BUILD_SOURCEVERSION" \
  --build-arg "NPM_REGISTRY=$NPM_REGISTRY_URL" \
  --secret id=npm_token,env=SYSTEM_ACCESSTOKEN \
  --cache-from "type=registry,ref=$CACHE_REF" \
  --cache-to "type=registry,ref=$CACHE_REF,mode=max" \
  --label "org.opencontainers.image.source=$BUILD_REPOSITORY_URI" \
  --label "org.opencontainers.image.created=$CREATED_AT" \
  --label "org.opencontainers.image.revision=$BUILD_SOURCEVERSION" \
  --label "org.opencontainers.image.title=$TENANT_NAME/$APP_NAME"
```

**Build arguments:**

| Arg | Value | Purpose |
|---|---|---|
| `GIT_COMMIT_SHA` | Full 40-char Git SHA (`$BUILD_SOURCEVERSION`) | Version embedding in binaries |
| `NPM_REGISTRY` | Platform Azure Artifacts npm feed URL | npm registry routing in Dockerfile |

**BuildKit secret:**

| Secret ID | Source | Purpose |
|---|---|---|
| `npm_token` | `SYSTEM_ACCESSTOKEN` (ADO-managed PAT) | Authenticates npm to the Azure Artifacts feed |

The secret is never stored in an image layer. Dockerfiles that consume it must use `RUN --mount=type=secret,id=npm_token,env=NPM_TOKEN ...`. Dockerfiles that do not reference the secret (Go, Python, Spring Boot) safely ignore it.

**OCI labels injected:**

| Label | Value |
|---|---|
| `org.opencontainers.image.source` | ADO repository URI |
| `org.opencontainers.image.created` | UTC timestamp at build time |
| `org.opencontainers.image.revision` | Full Git SHA |
| `org.opencontainers.image.title` | `<tenantName>/<appName>` |

**ACR layer cache:**

```
--cache-from type=registry,ref=<acrHost>/<tenantName>/<appName>:buildcache
--cache-to   type=registry,ref=<acrHost>/<tenantName>/<appName>:buildcache,mode=max
```

`mode=max` caches every intermediate layer, not just the final image layers. This maximizes cache effectiveness for multi-stage Dockerfiles (e.g., the `build` stage cache is preserved across runs even though the `build` stage image itself is discarded).

**Image digest capture:**

The `--iidfile /tmp/image.iid` flag causes BuildKit to write the config digest of the built image to a file. The step reads this file and emits it as the `IMAGE_DIGEST` output variable.

> **Important:** `--iidfile` writes the *config digest*, not the *manifest digest*. The manifest digest needed for Cosign signing is captured later in Stage 3, after the image is pushed to ACR and `docker inspect` is used to retrieve `RepoDigests[0]`.

---

## Output Variables

| Variable | Step name | Cross-stage reference |
|---|---|---|
| `IMAGE_DIGEST` | `buildImage` | `$(stageDependencies.Build.Build.outputs['buildImage.IMAGE_DIGEST'])` |

`IMAGE_DIGEST` is the config digest (`sha256:<hash>`) written by `--iidfile`. It is passed to Stage 3 as the `imageDigest` parameter of `sbom-sign-publish.yml` but is not used for Cosign — Cosign uses the manifest digest obtained after the ACR push in Stage 3.

---

## Image Lifecycle

```
Stage 2 (Build)       → image exists as agent-local: <tenantName>/<appName>:pipeline-<buildId>
Stage 3 (SignAndAttest) → pushed to ACR as: <acrHost>/<tenantName>/<appName>:<short-sha>  (for signing only)
Stage 4 (Publish)     → pushed to ACR as: <full-sha>, <branch>-<short-sha>, [<version>]
```

The image is never pushed to ACR before signing is complete.

---

## Dependencies

- Docker with BuildKit support must be available on the agent
- The agent must have push permission to the ACR cache repository (`<tenantName>/<appName>:buildcache`)
- `SYSTEM_ACCESSTOKEN` must be available (it is always present for ADO pipelines)

---

## Usage

This template is called exclusively by `container-build-v2.yml`. It is not intended to be called directly by tenant pipelines.

```yaml
# Inside container-build-v2.yml
- template: steps/docker-build.yml
  parameters:
    tenantName: ${{ parameters.tenantName }}
    appName: ${{ parameters.appName }}
    dockerfilePath: ${{ parameters.dockerfilePath }}
    buildContext: ${{ parameters.buildContext }}
    acrHost: $(stageDependencies.Setup.Setup.outputs['resolveTools.ACR_HOST'])
    npmRegistryUrl: $(stageDependencies.Setup.Setup.outputs['resolveTools.NPM_REGISTRY_URL'])
```
