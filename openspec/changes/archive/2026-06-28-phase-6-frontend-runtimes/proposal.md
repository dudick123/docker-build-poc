## Why

Phase 5 completes the Spring Boot runtime. Angular and React are the final two runtimes and share a common requirement that no other runtime has: npm packages must be fetched through the Azure Artifacts npm feed during the Docker build. The credentials for that feed must never appear in build args, image layers, or build logs — they must be passed as a BuildKit secret. Phase 6 implements `angular.yml` and `react.yml` as validated pass-throughs, and enhances `docker-build.yml` to inject the npm registry URL and auth token into the BuildKit invocation for all runtimes.

## What Changes

- **Modify** `platform-templates/steps/docker-build.yml`:
  - Add `npmRegistryUrl` parameter (resolved from `platform-tool-versions` → `NPM_REGISTRY_URL`)
  - Add `--build-arg "NPM_REGISTRY=$NPM_REGISTRY_URL"` to the `docker build` command
  - Add `--secret id=npm_token,env=SYSTEM_ACCESSTOKEN` to the `docker build` command
  - Map `NPM_REGISTRY_URL` and `SYSTEM_ACCESSTOKEN` in the `env:` block of the bash step
  - These flags are harmless for Go/Python/Spring Boot Dockerfiles that do not reference `ARG NPM_REGISTRY` or `--mount=type=secret,id=npm_token`

- **Modify** `platform-templates/container-build-v2.yml`:
  - Pass `npmRegistryUrl: $(stageDependencies.Setup.Setup.outputs['resolveTools.NPM_REGISTRY_URL'])` to `docker-build.yml`
  - Update Angular and React runtime dispatch to pass `dockerfilePath`, `tenantName`, `appName`

- **Replace stub** `platform-templates/steps/runtime/angular.yml`:
  - Advisory check: warn if `package.json` is not found in `buildContext`
  - Advisory check: warn if `ng` does not appear in any `scripts` value in `package.json`
  - Version extraction: read `version` field from `package.json`; emit as `ANGULAR_VERSION` step output variable on step named `angularRuntime`
  - No docker build — `docker-build.yml` already ran with npm credentials injected

- **Replace stub** `platform-templates/steps/runtime/react.yml`:
  - Advisory check: warn if `package.json` is not found in `buildContext`
  - Advisory check: detect Next.js (by presence of `next` in `package.json` dependencies); if detected and `next.config.js` / `next.config.mjs` does not contain `output.*export`, emit warning that SSR apps require a Node runtime in the final image, not nginx
  - Version extraction: read `version` field from `package.json`; emit as `REACT_VERSION` step output variable on step named `reactRuntime`
  - No docker build — `docker-build.yml` already ran with npm credentials injected

## Capabilities

### New Capabilities

- `angular-runtime-validation`: Advisory checks and metadata extraction that `angular.yml` performs — `package.json` presence, `ng` script presence, and `ANGULAR_VERSION` output variable convention.
- `react-runtime-validation`: Advisory checks and metadata extraction that `react.yml` performs — `package.json` presence, Next.js SSR detection, and `REACT_VERSION` output variable convention.

### Modified Capabilities

- `buildkit-image-build`: Docker build command now includes `--build-arg NPM_REGISTRY` and `--secret id=npm_token,env=SYSTEM_ACCESSTOKEN` to enable Azure Artifacts npm caching inside the Dockerfile for Angular and React builds.

## Impact

- **Modified files:** `platform-templates/steps/docker-build.yml`, `platform-templates/container-build-v2.yml`, `platform-templates/steps/runtime/angular.yml`, `platform-templates/steps/runtime/react.yml`
- **Produces for Phase 8:** `ANGULAR_VERSION` and `REACT_VERSION` step output variables — Phase 8 uses these for version tag construction; empty string means skip version tag
- **npm auth pattern:** `System.AccessToken` (ADO pipeline token) provides read access to Azure Artifacts feeds in the same ADO organisation — no additional service connection required
- **Security:** npm auth token is passed via BuildKit secret only; never a build arg, never in image history; Hadolint (Phase 3) enforces this at ERROR level
- **Agent requirement:** Docker + BuildKit only; no Node.js, npm, or Angular CLI on agent
- **Prerequisite:** `NPM_REGISTRY_URL` present in `platform-tool-versions` variable group (already validated by setup.yml in Phase 2)
