# Quick Start: Angular

This guide covers what the pipeline requires from an Angular application repository. Read [QUICK-START.md](QUICK-START.md) first for prerequisites and the base `azure-pipelines.yml` structure.

---

## Pipeline File

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
    tenantName: my-team
    appName: my-app
    runtimeType: angular
```

---

## Build Pattern

Angular uses a self-contained multi-stage Dockerfile. All `npm ci` and `ng build` commands run inside the Dockerfile — no Node.js toolchain is installed on the pipeline agent.

npm credentials for the platform Azure Artifacts feed are injected into every Docker build automatically via a BuildKit secret. Your Dockerfile must be written to consume those credentials (see the reference Dockerfile below).

---

## Repository Requirements

### package.json (recommended)

The pipeline performs an advisory check:
- Warns if `package.json` is not found in the build context
- Warns if `ng` does not appear in any `scripts` entry in `package.json`

Neither warning blocks the build. They are signals that the runtime detection may not be accurate.

### Version extraction

The pipeline reads the `"version"` field from `package.json` for optional ACR version tags:

```json
{
  "name": "my-app",
  "version": "2.3.1",
  ...
}
```

If `package.json` is absent or has no `version` field, the version tag is skipped.

---

## npm Credentials

The pipeline injects two values into every `docker build` invocation:

| Value | How it is passed | Purpose |
|---|---|---|
| Platform npm registry URL | `--build-arg NPM_REGISTRY=<url>` | Points npm at the Azure Artifacts feed |
| Azure Artifacts token | `--secret id=npm_token,env=SYSTEM_ACCESSTOKEN` | Auth token for private package download |

Your Dockerfile must use these in the `npm ci` step. The secret is passed using BuildKit's `--mount=type=secret` syntax and is **never stored in the image layer**.

---

## Reference Dockerfile

```dockerfile
# syntax=docker/dockerfile:1

# ── build stage ──────────────────────────────────────────────────────────────
FROM node:22-alpine AS build

WORKDIR /app

# Receive platform-provided npm registry URL
ARG NPM_REGISTRY

# Configure npm to use the platform Azure Artifacts feed.
# The auth token is mounted as a BuildKit secret — never written to the image.
RUN --mount=type=secret,id=npm_token,env=NPM_TOKEN \
    npm config set registry "${NPM_REGISTRY}" && \
    npm config set "//${NPM_REGISTRY#https://}:_authToken" "${NPM_TOKEN}"

COPY package.json package-lock.json ./
RUN npm ci --prefer-offline

COPY . .

ARG GIT_COMMIT_SHA=dev
ENV GIT_COMMIT_SHA=${GIT_COMMIT_SHA}

RUN npm run build -- --output-path=dist/app

# ── final stage ──────────────────────────────────────────────────────────────
FROM nginx:1.27-alpine AS final

COPY --from=build /app/dist/app /usr/share/nginx/html

# Optional: custom nginx config for Angular routing (HTML5 pushState)
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### nginx.conf for Angular routing

Angular apps that use HTML5 routing (the default) require nginx to return `index.html` for any path that does not match a static file:

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache static assets aggressively
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff2?)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

Place `nginx.conf` in your repository root (alongside your `Dockerfile`) and add the `COPY` line above to your Dockerfile.

### Key points

- **`--mount=type=secret,id=npm_token,env=NPM_TOKEN`** — BuildKit mounts the secret at build time. It is never written to any layer in the final image. This syntax requires `# syntax=docker/dockerfile:1` at the top of the Dockerfile (it enables BuildKit syntax extensions).
- **`ARG NPM_REGISTRY`** — the pipeline passes the platform Azure Artifacts URL here. Using `ARG` (not `ENV`) means the value is not baked into the final image.
- **Auth token scoping** — the `npm config set` command scopes the token to the specific registry host. Tokens for `registry.npmjs.org` are not affected.
- **`GIT_COMMIT_SHA` build arg** — injected by the pipeline. The `ARG` default (`dev`) keeps local builds working without the pipeline. Embed it in your app via `environment.ts` or an `APP_VERSION` injection in `angular.json`.
- **`npm ci --prefer-offline`** — `--prefer-offline` uses the local npm cache if available, reducing bandwidth. `npm ci` (not `npm install`) ensures the `package-lock.json` is respected exactly.

---

## What the Pipeline Validates

The Angular runtime step (`steps/runtime/angular.yml`) runs after the Docker build and performs:

1. Checks for `package.json` in the build context — advisory warning if absent
2. Checks for `ng` in `package.json` scripts — advisory warning if absent
3. Extracts `"version"` from `package.json` — emits as `ANGULAR_VERSION` for the Publish stage tag logic

This step does not run `npm`, `ng`, or any Node.js command on the agent.

---

## Version Tagging Summary

| Scenario | Tags pushed to ACR |
|---|---|
| No `"version"` in `package.json` | `<40-char-sha>`, `<branch>-<12-char-sha>` |
| Version found, non-main branch | `<40-char-sha>`, `<branch>-<12-char-sha>`, `<version>-<12-char-sha>` |
| Version found, `main` branch, tag does not exist | `<40-char-sha>`, `main-<12-char-sha>`, `<version>` |
| Version found, `main` branch, tag already exists | Pipeline fails — bump the version before merging |

---

## Common Issues

**`No package.json found` warning**
The build context does not contain `package.json`. If your Angular project root is a subdirectory, set `buildContext` accordingly:

```yaml
parameters:
  buildContext: frontend/angular-app
```

**npm install fails inside Docker build with 401 Unauthorized**
The `--mount=type=secret` syntax is not in the Dockerfile, or the `# syntax=docker/dockerfile:1` directive is missing from line 1. Both are required for BuildKit secrets to work. Verify the first line of your Dockerfile and that the `RUN --mount=type=secret` form is used for the `npm config set` step.

**`ng: not found` during `npm run build`**
The `@angular/cli` package is not in `devDependencies` in `package.json`. Add it, or call the local CLI via `npx ng build` or `./node_modules/.bin/ng build`.

**Large image size**
Ensure the `dist/` output from the build stage is the only content copied to the nginx final stage. The `node_modules/` directory must stay in the build stage only. If your final image is unexpectedly large, run `docker history <image>` to identify which layer is large.

**Angular routing returns 404 on page refresh**
Add the `nginx.conf` shown above (or equivalent) to serve `index.html` for all unmatched paths. Without this, direct URL navigation and page refreshes return 404 from nginx.
