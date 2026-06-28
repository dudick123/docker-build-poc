# npm Dependency Caching via Azure Artifacts

**Status:** Approved pattern
**Applies to:** Angular, React runtimes (`runtimeType: angular`, `runtimeType: react`)
**Established:** 2026-06-28

---

## Problem

ADO build agents are self-hosted and ephemeral — the agent is destroyed after each run. Agent-local npm caches do not survive between builds. During high-demand periods (end of sprint, many teams building simultaneously), builds that cannot cache `node_modules` hit the public npm registry over the internet. Network contention at these times causes build times to spike significantly.

Docker layer caching on ACR partially addresses this — when `package-lock.json` is unchanged the `npm ci` layer is a cache hit and npm never touches the network. But when dependencies change, every build fetches fresh from the public registry.

---

## Solution

Route all npm traffic through the platform **Azure Artifacts npm feed** rather than the public registry. Azure Artifacts acts as a persistent proxy cache:

```
Docker build (ephemeral agent)
  │
  │  npm ci
  ▼
Azure Artifacts npm feed  ←──── cache hit (most requests)
  │
  │  cache miss only
  ▼
public npm registry (registry.npmjs.org)
```

Packages are fetched from the public registry once, then served from Azure Artifacts on every subsequent request — regardless of which agent or which build triggered it. External network dependency is reduced to first-fetch only per package version.

This combines with ACR Docker layer caching for maximum efficiency:

```
Build run N:   package-lock.json unchanged → ACR layer cache hit → npm never runs
Build run N+1: package-lock.json changed   → npm ci runs → Azure Artifacts cache hit
Build run N+2: new package added           → npm ci runs → Azure Artifacts miss → public registry fetch → cached for all future runs
```

---

## Secret Hygiene Requirement

Azure Artifacts npm feeds require authentication. Passing the auth token into a Docker build **must not** use `--build-arg` — build args are baked into the image layer history and visible via `docker history`. Hadolint enforces this at ERROR level.

The correct mechanism is **BuildKit `--secret`**, which passes a value into a specific `RUN` step in memory only. It never appears in any image layer, build log, or `docker history` output.

---

## Implementation

### Platform configuration (`platform-tool-versions` variable group)

Add the Azure Artifacts npm feed URL as a platform-managed variable:

```
NPM_REGISTRY_URL = https://pkgs.dev.azure.com/<org>/_packaging/<feed>/npm/registry/
```

The auth token is **not** stored in the variable group. It is retrieved at build time from the ADO service connection scoped to Azure Artifacts.

### Pipeline (`steps/docker-build.yml`)

```yaml
- script: |
    docker build \
      --build-arg NPM_REGISTRY=$(NPM_REGISTRY_URL) \
      --secret id=npm_token,env=AZURE_ARTIFACTS_TOKEN \
      --cache-from type=registry,ref=$(ACR_HOST)/$(tenantName)/$(appName):buildcache \
      --cache-to   type=registry,ref=$(ACR_HOST)/$(tenantName)/$(appName):buildcache,mode=max \
      --tag $(IMAGE_TAG) \
      --file $(dockerfilePath) \
      $(buildContext)
  displayName: Build image
  env:
    AZURE_ARTIFACTS_TOKEN: $(System.AccessToken)
```

`System.AccessToken` is the ADO pipeline token. It has read access to Azure Artifacts feeds within the same ADO organisation without additional configuration.

### Dockerfile (tenant-owned)

Tenants structure their Node build stage to use the platform registry and mount the secret:

```dockerfile
FROM node:20-alpine AS builder

WORKDIR /app

# Copy dependency manifests before source — maximises Docker layer cache reuse
COPY package.json package-lock.json ./

# Mount the Azure Artifacts token as a BuildKit secret.
# The token is available only within this RUN step and never written to any layer.
ARG NPM_REGISTRY
RUN --mount=type=secret,id=npm_token \
    npm config set registry ${NPM_REGISTRY} && \
    npm config set //${NPM_REGISTRY#https://}:_authToken=$(cat /run/secrets/npm_token) && \
    npm ci

# Copy source after npm ci — a source change does not bust the npm layer
COPY . .
RUN npm run build

# Final stage — static assets only
FROM nginx:alpine AS final
COPY --from=builder /app/dist /usr/share/nginx/html
```

Key structural points:
- `COPY package.json package-lock.json ./` appears **before** `COPY . .` — this is required for the ACR layer cache to work correctly. A source file change will not invalidate the `npm ci` layer if only `COPY . .` follows it.
- `ARG NPM_REGISTRY` is the registry URL (not a secret — safe as a build arg).
- `--mount=type=secret,id=npm_token` injects the token in-memory for this step only.
- The token config line uses shell parameter expansion to strip the `https://` prefix for the npm config key format.

---

## Platform-provided reference Dockerfiles

The platform templates repository includes reference Dockerfiles for Angular and React that implement this pattern correctly. Tenants are expected to start from these references. The `angular.yml` and `react.yml` step templates validate that the Dockerfile does not use `--build-arg` for the auth token (enforced by Hadolint).

---

## What this does not solve

- **Maven / Gradle dependency caching** — Azure Artifacts also supports Maven and Gradle feeds. If the platform provisions feeds for those, the same BuildKit secret pattern applies using `settings.xml` (Maven) or `init.d` Gradle configuration. This is a candidate for a follow-on pattern.
- **Private npm packages** — packages hosted in the Azure Artifacts feed itself (not proxied from public) are served the same way. No additional configuration is needed in the Dockerfile.
- **Node version management** — tenants pin their Node version in the `FROM` line of their Dockerfile. The platform does not manage Node versions on the agent.
