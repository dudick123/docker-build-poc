# Test Cases: steps/docker-build.yml

**Template:** `platform-templates/steps/docker-build.yml`
**Stage:** Build (Stage 2) — second step
**Steps under test:** `buildImage` (bash)

---

## TC-BUILD-001 — Happy path: image builds successfully

**Level:** L2
**Type:** Positive

**Precondition:** `fixtures/dockerfiles/Dockerfile.go-valid` committed as `Dockerfile`. Valid `go.mod` present.

**Expected result:** `buildImage` step passes. Build log contains:
```
Building image: platform-test/test-app:pipeline-<BUILD_BUILDID>
Cache ref: <ACR_HOST>/platform-test/test-app:buildcache
IMAGE_DIGEST=sha256:<hash>
```

`IMAGE_DIGEST` output variable is emitted and is a non-empty `sha256:` string.

---

## TC-BUILD-002 — GIT_COMMIT_SHA build arg is injected

**Level:** L2
**Type:** Positive — build arg verification

**Precondition:** Go Dockerfile with:
```dockerfile
ARG GIT_COMMIT_SHA=dev
RUN echo "sha=${GIT_COMMIT_SHA}" > /sha.txt
```
And in the final stage: `COPY --from=build /sha.txt /sha.txt`

**Verification:** After the pipeline run, inspect the built image:
```bash
docker run --rm <image> cat /sha.txt
```

**Expected:** Output is `sha=<40-char-sha>` matching `$BUILD_SOURCEVERSION`. Default `dev` must NOT appear.

---

## TC-BUILD-003 — OCI labels are injected into the image

**Level:** L2
**Type:** Positive — label verification

**Verification:** After `buildImage` step:
```bash
docker inspect <local-image-ref> --format '{{ json .Config.Labels }}'
```

**Expected labels present:**

| Label | Expected value |
|---|---|
| `org.opencontainers.image.source` | ADO repository URI |
| `org.opencontainers.image.created` | UTC timestamp (ISO 8601) |
| `org.opencontainers.image.revision` | Full 40-char Git SHA |
| `org.opencontainers.image.title` | `platform-test/test-app` |

---

## TC-BUILD-004 — Image is NOT pushed to ACR during this step

**Level:** L2
**Type:** Negative (absence verification)

**Expected result:** After `buildImage` completes, confirm that the image does NOT appear in ACR under `<acrHost>/platform-test/test-app`. The image exists only as `platform-test/test-app:pipeline-<BUILD_BUILDID>` on the agent.

**Verification:** Check ACR after Stage 2 with `az acr repository show-tags --name <acr> --repository platform-test/test-app`. With `dryRun: true`, no tags should exist until a prior full run.

---

## TC-BUILD-005 — ACR layer cache is used on repeat build

**Level:** L2
**Type:** Positive — cache verification

**Precondition:** Run TC-BUILD-001 once (populates the `buildcache` tag in ACR). Run again with unchanged source.

**Expected result:** Second build is faster. Build log contains `--cache-from type=registry,ref=<acr>/platform-test/test-app:buildcache`. Docker layer output shows `CACHED` for layers that did not change.

---

## TC-BUILD-006 — npm secret is mounted (not stored as a layer)

**Level:** L2
**Type:** Security — secret non-persistence

**Precondition:** Angular or React Dockerfile that uses `--mount=type=secret,id=npm_token,env=NPM_TOKEN`.

**Verification:** After the build:
```bash
docker history <local-image-ref>
```

**Expected:** No layer in the history contains the `SYSTEM_ACCESSTOKEN` value. The secret must not appear in any layer metadata, `ENV`, `ARG`, or `RUN` cache key.

---

## TC-BUILD-007 — Docker build fails: invalid FROM

**Level:** L2
**Type:** Negative

**Precondition:** `Dockerfile` with an invalid base image:
```dockerfile
FROM does-not-exist:invalid AS build
```

**Expected result:** `buildImage` step fails with a Docker pull error. Stage 2 fails. `IMAGE_DIGEST` is NOT emitted. Stages 3–5 do not run.

---

## TC-BUILD-008 — IMAGE_DIGEST output variable is available in Stage 3

**Level:** L2/L3
**Type:** Positive — cross-stage variable propagation

**Verification:** In a full run, Stage 3 receives `imageDigest` parameter from `$(stageDependencies.Build.Build.outputs['buildImage.IMAGE_DIGEST'])`. Confirm the parameter is non-empty in the Stage 3 build log by adding a diagnostic `echo` step.
