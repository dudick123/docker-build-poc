# Quick Start: React / Next.js

This guide covers what the pipeline requires from a React or Next.js application repository. Read [QUICK-START.md](QUICK-START.md) first for prerequisites and the base `azure-pipelines.yml` structure.

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
    runtimeType: react
```

`runtimeType: react` covers both plain React (Create React App, Vite) and Next.js. The same runtime validator runs for both — see the Next.js section below for the SSR-specific requirement.

---

## Build Pattern

React and Next.js use a self-contained multi-stage Dockerfile. All `npm ci` and build commands run inside the Dockerfile — no Node.js toolchain is installed on the pipeline agent.

npm credentials for the platform Azure Artifacts feed are injected into every Docker build automatically via a BuildKit secret. Your Dockerfile must be written to consume those credentials.

---

## Repository Requirements

### package.json (recommended)

The pipeline performs an advisory check:
- Warns if `package.json` is not found in the build context

### Version extraction

The pipeline reads the `"version"` field from `package.json` for optional ACR version tags:

```json
{
  "name": "my-app",
  "version": "1.8.0",
  ...
}
```

If `package.json` is absent or has no `version` field, the version tag is skipped.

### Next.js — static export required

If `"next"` appears in `package.json` dependencies, the pipeline checks for `output: 'export'` in `next.config.js` or `next.config.mjs`. If it is absent, a warning is written to the build log:

```
Next.js app detected without 'output: export' in next.config.js. SSR apps require a Node.js runtime in the final image, not nginx.
```

This is a warning, not a failure. But if you are running a Next.js SSR app and serving it with nginx, your application will not work at runtime — see the [SSR Dockerfile](#nextjs-ssr-node-runtime) below.

---

## npm Credentials

The pipeline injects two values into every `docker build` invocation:

| Value | How it is passed | Purpose |
|---|---|---|
| Platform npm registry URL | `--build-arg NPM_REGISTRY=<url>` | Points npm at the Azure Artifacts feed |
| Azure Artifacts token | `--secret id=npm_token,env=SYSTEM_ACCESSTOKEN` | Auth token for private package download |

Your Dockerfile must consume these in the `npm ci` step using BuildKit's `--mount=type=secret` syntax. The secret is **never stored in any image layer**.

---

## Reference Dockerfiles

### React (Vite / Create React App) — static export → nginx

```dockerfile
# syntax=docker/dockerfile:1

# ── build stage ──────────────────────────────────────────────────────────────
FROM node:22-alpine AS build

WORKDIR /app

ARG NPM_REGISTRY

RUN --mount=type=secret,id=npm_token,env=NPM_TOKEN \
    npm config set registry "${NPM_REGISTRY}" && \
    npm config set "//${NPM_REGISTRY#https://}:_authToken" "${NPM_TOKEN}"

COPY package.json package-lock.json ./
RUN npm ci --prefer-offline

COPY . .

ARG GIT_COMMIT_SHA=dev
ENV VITE_GIT_COMMIT_SHA=${GIT_COMMIT_SHA}

RUN npm run build

# ── final stage ──────────────────────────────────────────────────────────────
FROM nginx:1.27-alpine AS final

COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### Next.js — static export → nginx

Configure Next.js for static export in `next.config.js`:

```js
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: 'export',
  trailingSlash: true,
};

module.exports = nextConfig;
```

Then use this Dockerfile:

```dockerfile
# syntax=docker/dockerfile:1

# ── build stage ──────────────────────────────────────────────────────────────
FROM node:22-alpine AS build

WORKDIR /app

ARG NPM_REGISTRY

RUN --mount=type=secret,id=npm_token,env=NPM_TOKEN \
    npm config set registry "${NPM_REGISTRY}" && \
    npm config set "//${NPM_REGISTRY#https://}:_authToken" "${NPM_TOKEN}"

COPY package.json package-lock.json ./
RUN npm ci --prefer-offline

COPY . .

ARG GIT_COMMIT_SHA=dev
ENV NEXT_PUBLIC_GIT_COMMIT_SHA=${GIT_COMMIT_SHA}

RUN npm run build

# ── final stage ──────────────────────────────────────────────────────────────
FROM nginx:1.27-alpine AS final

# next export writes to /out when output: 'export' is set
COPY --from=build /app/out /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### Next.js SSR — Node.js runtime

If your Next.js app uses server-side rendering, server components, or API routes, it cannot be served as static HTML. Use a Node.js final image:

```dockerfile
# syntax=docker/dockerfile:1

# ── deps stage ────────────────────────────────────────────────────────────────
FROM node:22-alpine AS deps

WORKDIR /app

ARG NPM_REGISTRY

RUN --mount=type=secret,id=npm_token,env=NPM_TOKEN \
    npm config set registry "${NPM_REGISTRY}" && \
    npm config set "//${NPM_REGISTRY#https://}:_authToken" "${NPM_TOKEN}"

COPY package.json package-lock.json ./
RUN npm ci --prefer-offline

# ── build stage ──────────────────────────────────────────────────────────────
FROM node:22-alpine AS build

WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY . .

ARG GIT_COMMIT_SHA=dev
ENV NEXT_PUBLIC_GIT_COMMIT_SHA=${GIT_COMMIT_SHA}

RUN npm run build

# ── final stage ──────────────────────────────────────────────────────────────
FROM node:22-alpine AS final

WORKDIR /app

ENV NODE_ENV=production

COPY --from=build /app/public ./public
COPY --from=build /app/.next/standalone ./
COPY --from=build /app/.next/static ./.next/static

EXPOSE 3000

USER node

CMD ["node", "server.js"]
```

For the standalone output to work, add `output: 'standalone'` to `next.config.js`:

```js
const nextConfig = {
  output: 'standalone',
};
module.exports = nextConfig;
```

### nginx.conf for SPA routing

React apps and statically exported Next.js apps need nginx to serve `index.html` for unmatched paths:

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff2?)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

---

## What the Pipeline Validates

The React runtime step (`steps/runtime/react.yml`) runs after the Docker build and performs:

1. Checks for `package.json` in the build context — advisory warning if absent
2. If `"next"` is in `package.json` dependencies, checks for `output: export` in `next.config.js` or `next.config.mjs` — advisory warning if absent (SSR apps need a Node.js final image)
3. Extracts `"version"` from `package.json` — emits as `REACT_VERSION` for the Publish stage tag logic

This step does not run `npm`, `node`, or any Node.js command on the agent.

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
The build context does not contain `package.json`. If your React project root is a subdirectory, set `buildContext` accordingly:

```yaml
parameters:
  buildContext: frontend/react-app
```

**npm install fails inside Docker build with 401 Unauthorized**
The `--mount=type=secret` syntax is missing or the `# syntax=docker/dockerfile:1` directive is absent from line 1. Both are required. Verify the Dockerfile header and that the `npm config set` step uses the `RUN --mount=type=secret` form.

**Next.js warning: `output: export` not set**
This warning fires if your `package.json` lists `"next"` as a dependency and your `next.config.js` does not specify `output: 'export'`. If you are intentionally running an SSR app with the Node.js final image, you can add a `.hadolint.yaml` override — but the platform warning is still advisory and will appear in every build log. Contact platform engineering if this is a persistent concern.

**Next.js static export build fails with dynamic route error**
`output: 'export'` does not support dynamic routes that require server-side data fetching at request time (`getServerSideProps`, server components with dynamic data). Either use `getStaticProps` / `generateStaticParams` to pre-render those routes, or switch to the SSR Dockerfile pattern with `output: 'standalone'`.

**Large nginx image**
Ensure only the `dist/` or `out/` directory is copied to the nginx stage. `node_modules/` must remain in the build stage. If the final image is unexpectedly large, run `docker history <image>` to find the large layer.

**SPA routes return 404 on page refresh**
Add the `nginx.conf` shown above (or equivalent). Without `try_files $uri $uri/ /index.html`, nginx returns 404 for paths that do not correspond to static files.
