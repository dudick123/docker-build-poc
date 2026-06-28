## ADDED Requirements

### Requirement: Image is built with rootless BuildKit using the platform-pinned version
The `steps/docker-build.yml` template SHALL invoke `docker build` with `DOCKER_BUILDKIT=1` set as an environment variable, using the Docker version specified by `$(stageDependencies.Setup.Setup.outputs['resolveTools.DOCKER_BUILDKIT_VERSION'])`. The build SHALL use the `dockerfilePath` and `buildContext` parameters passed from the base template.

#### Scenario: BuildKit build executes with correct Dockerfile and context
- **WHEN** a tenant pipeline sets `dockerfilePath: Dockerfile` and `buildContext: .`
- **THEN** the build step runs `DOCKER_BUILDKIT=1 docker build -f ./Dockerfile .` (with additional flags) and exits zero on success

#### Scenario: Non-default Dockerfile path and build context are respected
- **WHEN** a tenant pipeline sets `dockerfilePath: docker/Dockerfile.prod` and `buildContext: services/api`
- **THEN** the build step uses `-f services/api/docker/Dockerfile.prod` with build context `services/api`

### Requirement: Four OCI standard image labels are injected at build time
The build step SHALL inject the following four labels via `--label` flags on the `docker build` invocation. Tenants are not required to add these labels to their Dockerfiles:
- `org.opencontainers.image.source` — set to the pipeline repository URL (`$(Build.Repository.Uri)`)
- `org.opencontainers.image.created` — set to the ISO 8601 UTC timestamp of the build (`$(date -u +%Y-%m-%dT%H:%M:%SZ)`)
- `org.opencontainers.image.revision` — set to the full 40-character Git SHA (`$(Build.SourceVersion)`)
- `org.opencontainers.image.title` — set to `<tenantName>/<appName>`

#### Scenario: Built image carries all four OCI labels
- **WHEN** the build step completes successfully
- **THEN** `docker inspect <image>` shows all four `org.opencontainers.image.*` labels with correct values

#### Scenario: Tenant Dockerfile already contains an OCI label
- **WHEN** the Dockerfile contains `LABEL org.opencontainers.image.source=https://example.com`
- **THEN** the `--label` flag from the build step overrides the Dockerfile value; the pipeline-injected value takes precedence

### Requirement: GIT_COMMIT_SHA build arg is passed to every build
The build step SHALL pass `--build-arg GIT_COMMIT_SHA=$(Build.SourceVersion)` to every `docker build` invocation. The Dockerfile MAY consume this arg via `ARG GIT_COMMIT_SHA`; if absent, Docker ignores the unused build arg.

#### Scenario: Dockerfile consumes GIT_COMMIT_SHA
- **WHEN** the Dockerfile declares `ARG GIT_COMMIT_SHA` and embeds it (e.g., in a binary or metadata file)
- **THEN** the built image contains the correct full Git SHA from the pipeline run

#### Scenario: Dockerfile does not declare GIT_COMMIT_SHA arg
- **WHEN** the Dockerfile has no `ARG GIT_COMMIT_SHA` instruction
- **THEN** the build proceeds without error; Docker silently ignores unused build args passed via `--build-arg`

### Requirement: ACR layer cache is used for every build
The build step SHALL include `--cache-from type=registry,ref=<ACR_HOST>/<tenantName>/<appName>:buildcache` and `--cache-to type=registry,ref=<ACR_HOST>/<tenantName>/<appName>:buildcache,mode=max` on every invocation. `ACR_HOST` SHALL be read from the `platform-tool-versions` variable group via the Setup stage output variable. Cache miss on a cold cache SHALL NOT fail the build.

#### Scenario: Cache exists from a previous run
- **WHEN** a prior build has written to `<ACR_HOST>/<tenantName>/<appName>:buildcache`
- **THEN** BuildKit uses cached layers for unchanged stages, reducing build time

#### Scenario: No cache exists on first run
- **WHEN** `<ACR_HOST>/<tenantName>/<appName>:buildcache` does not exist in ACR
- **THEN** the build completes successfully without cache; BuildKit treats the missing cache tag as a cache miss, not an error

### Requirement: Image is held locally and not pushed to ACR during the build step
After `docker build` completes, the image SHALL exist only in the local Docker daemon on the agent. The build step SHALL NOT run `docker push` or any equivalent. The image remains local until the Publish stage (Phase 8).

#### Scenario: Build step completes without pushing
- **WHEN** the build step exits successfully
- **THEN** the image is present in the local Docker daemon (`docker images` shows it) and is absent from ACR

### Requirement: Full image digest is captured and emitted as a step output variable
After `docker build` completes, the build step SHALL capture the image digest in `sha256:<hex>` format using `--iidfile` or `docker inspect`, and emit it as a step output variable named `IMAGE_DIGEST` on a step named `buildImage`. Downstream stages (Phase 7 sign, Phase 8 publish) SHALL reference this value.

#### Scenario: Image digest is captured after successful build
- **WHEN** the build step completes successfully
- **THEN** the step log shows `IMAGE_DIGEST=sha256:<64-hex-chars>` and the value is available as `$(stageDependencies.Build.Build.outputs['buildImage.IMAGE_DIGEST'])` to downstream stages

#### Scenario: Build failure does not emit a digest
- **WHEN** `docker build` exits non-zero (e.g., a RUN step fails)
- **THEN** no `IMAGE_DIGEST` output variable is set; the Build stage fails and downstream stages are skipped per their `dependsOn` conditions
