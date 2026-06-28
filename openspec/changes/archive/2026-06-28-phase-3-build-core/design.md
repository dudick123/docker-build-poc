## Context

Both `steps/dockerfile-lint.yml` and `steps/docker-build.yml` are currently Phase 1 stubs. They sit in the Build stage's single job, called in sequence: lint runs first, then build. Both step templates receive `tenantName`, `appName`, `dockerfilePath`, and `buildContext` as parameters from the base template. They also need the tool version output variables resolved in Phase 2 (`DOCKER_BUILDKIT_VERSION`, `HADOLINT_VERSION`) — these are consumed via the cross-stage output variable reference syntax established in Phase 2.

The Build job runs on the platform agent image, which has Docker with BuildKit support. No language toolchain is installed on the agent for Pattern A runtimes — the full build happens inside the Dockerfile.

Stakeholders: platform engineering (owns the templates), tenant teams (author the Dockerfiles).

## Goals / Non-Goals

**Goals:**
- Run Hadolint at the pinned version, differentiate ERROR vs. WARNING findings, surface findings in ADO build log
- Run a rootless BuildKit build with OCI labels, `GIT_COMMIT_SHA` build arg, and ACR layer cache
- Capture the exact `sha256:<digest>` of the built image as a pipeline output variable
- Hold the image locally — no ACR push in this phase

**Non-Goals:**
- Signing, attesting, or publishing the image (Phases 7–8)
- Runtime-specific pre-build steps (Phases 4–6); these execute in the same Build job but in separate step templates after `docker-build.yml`
- Installing or version-managing Docker/BuildKit on the agent (agent image is pre-provisioned)
- Enforcing specific Hadolint rules — tenants can override via `.hadolint.yaml`; the platform enforces only the severity gate

## Decisions

### Decision 1: Hadolint runs as a downloaded binary, pinned to `HADOLINT_VERSION`

The `dockerfile-lint.yml` step downloads Hadolint from GitHub releases at the version specified by `$(stageDependencies.Setup.Setup.outputs['resolveTools.HADOLINT_VERSION'])`, verifies the binary is executable, then runs it. It does not rely on a system-installed Hadolint.

**Rationale:** Pinning to an exact version (from the `platform-tool-versions` variable group) guarantees reproducible lint results across all tenant builds and allows the platform team to upgrade Hadolint centrally. Alternative (use system-installed Hadolint) was rejected because agent images may lag behind platform-required versions.

### Decision 2: Hadolint `--format tty` with exit-code differentiation for ERROR vs. WARNING

Hadolint is invoked with `--format tty` and `--failure-threshold error`. This causes Hadolint to exit non-zero only when findings at ERROR level or above are present; WARNING-level findings print but do not cause a non-zero exit code.

**Rationale:** `--failure-threshold error` is the built-in Hadolint mechanism for severity gating, cleaner than parsing output or using `--no-fail` with manual checks. Alternative (always fail on any finding) was rejected as too strict for initial rollout — tenants need time to resolve WARNING-level findings.

### Decision 3: `.hadolint.yaml` discovered from build context root; absent is not an error

If `<buildContext>/.hadolint.yaml` exists, it is passed to Hadolint via `--config`. If absent, Hadolint runs with its built-in defaults. The platform does not ship a default `.hadolint.yaml`.

**Rationale:** Tenants have legitimate needs to ignore specific rules for their runtime (e.g., `DL3008` for apt pinning in Python images). Forcing a platform default config would require every tenant to override it. Absence is not an error because Hadolint defaults are reasonable starting points.

### Decision 4: BuildKit build uses `DOCKER_BUILDKIT=1` env var, not the `docker buildx` subcommand

The build step sets `DOCKER_BUILDKIT=1` and calls `docker build` (not `docker buildx build`). Cache flags use the `--cache-from type=registry` and `--cache-to type=registry,mode=max` syntax.

**Rationale:** `docker build` with `DOCKER_BUILDKIT=1` is the stable, widely-supported BuildKit invocation path on the platform agent image. `docker buildx` requires a buildx builder instance to be created and managed, adding setup steps that are not needed for single-platform builds. The `type=registry` cache backend is compatible with ACR.

### Decision 5: OCI labels injected via `--label` flags, not `LABEL` instructions in the Dockerfile

The four OCI labels (`org.opencontainers.image.source`, `.created`, `.revision`, `.title`) are injected at build time via `docker build --label` flags. Tenants do not need to add `LABEL` instructions to their Dockerfiles.

**Rationale:** Keeping labels out of the Dockerfile avoids coupling tenant Dockerfiles to platform metadata conventions. Labels injected via `--label` override any same-named `LABEL` in the Dockerfile, so tenants who do have `LABEL` instructions are not broken. Alternative (require tenants to add `LABEL` instructions) was rejected because it creates a migration burden and a fragile convention to enforce.

### Decision 6: Image digest captured by inspecting the built image, not parsing `docker build` output

After `docker build` completes, the digest is captured by running `docker inspect --format='{{index .RepoDigests 0}}' <image-ref>` or by using `docker build --iidfile` to write the image ID, then converting to a digest via `docker inspect`. The digest is emitted as `IMAGE_DIGEST` step output variable on a step named `buildImage`.

**Rationale:** Parsing `docker build` stdout for the digest is fragile and BuildKit output format varies across versions. `--iidfile` writes the image config digest reliably. The config digest is the canonical identifier used by Cosign for signing. Alternative (use the image tag) was rejected because tags are mutable; the digest is immutable and required for all downstream operations.

### Decision 7: ACR cache scope is `<acrHost>/<tenantName>/<appName>:buildcache`

Cache flags: `--cache-from type=registry,ref=<acrHost>/<tenantName>/<appName>:buildcache` and `--cache-to type=registry,ref=<acrHost>/<tenantName>/<appName>:buildcache,mode=max`. The ACR host is read from a variable group variable `ACR_HOST` (added to `platform-tool-versions`).

**Rationale:** Per-tenant-per-app cache scope prevents cross-tenant cache pollution and keeps cache size bounded. `mode=max` caches all intermediate layers, maximising cache hit rate on incremental code changes. Cache miss on first run is not an error — `--cache-from` is advisory.

## Risks / Trade-offs

- **`--iidfile` writes image config digest, not manifest digest** — The config digest and manifest digest differ. Cosign signs by manifest digest. Mitigation: after build, run `docker inspect` on the image ref to get the manifest digest if the registry has already been pushed; but since we're pre-push in Phase 3, we emit the config digest as `IMAGE_DIGEST` and Phase 7 will re-resolve to the manifest digest post-push. This is a known two-phase pattern with Cosign.
- **ACR cache unavailable on first run** — `--cache-from` silently no-ops if the tag doesn't exist. Build succeeds, just slower. No mitigation needed.
- **Hadolint download requires internet access from agent** — If the platform agent is on a restricted network, the binary download will fail. Mitigation: document that the agent image should pre-install Hadolint, or the download URL should point to an internal mirror. Flagged as a deployment configuration item, not a template defect.

## Open Questions

None. The `ACR_HOST` variable is added to `platform-tool-versions` as part of this phase — it was implicitly required from Phase 3 onward per the implementation plan (D-1).
