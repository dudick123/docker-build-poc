# Example: React (Vite / Static Export)

A complete, annotated example of the platform container build pipeline for a React application that compiles to static HTML/JS/CSS. This example covers Vite, Create React App, and Next.js in static export mode.

For Next.js applications using server-side rendering (SSR), server components, or API routes, see [../react-nextjs-ssr/](../react-nextjs-ssr/).

---

## Files in This Example

| File | Purpose |
|---|---|
| `azure-pipelines.yml` | ADO pipeline definition |
| `Dockerfile` | Two-stage Dockerfile: `build` (Node.js/Vite) → `final` (nginx) |
| `nginx.conf` | nginx server block with SPA fallback routing |
| `README.md` | This document |

---

## Which Example to Use

| Framework | Mode | Use this example |
|---|---|---|
| Vite + React | Static | ✅ This example |
| Create React App | Static | ✅ This example (change `dist` to `build` in Dockerfile) |
| Next.js | `output: 'export'` | ✅ This example + add `next.config.js` |
| Next.js | SSR / `output: 'standalone'` | ❌ Use [react-nextjs-ssr](../react-nextjs-ssr/) |

### Next.js static export configuration

If using Next.js with this example, add `output: 'export'` to `next.config.js`:

```js
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: 'export',
  trailingSlash: true,    // recommended for nginx static file serving
};

module.exports = nextConfig;
```

Without this setting, the pipeline runtime validator emits an advisory warning:
> `Next.js app detected without 'output: export' in next.config.js. SSR apps require a Node.js runtime in the final image, not nginx.`

The build continues, but the resulting image will not work if the app uses SSR features.

Also update the COPY in the Dockerfile final stage — Next.js static export writes to `out/`:

```dockerfile
COPY --from=build /app/out /usr/share/nginx/html
```

---

## npm Credential Injection

The platform pipeline injects npm credentials into every Docker build:

```
--build-arg NPM_REGISTRY=<azure-artifacts-npm-feed-url>
--secret id=npm_token,env=SYSTEM_ACCESSTOKEN
```

The Dockerfile configures npm to use the platform feed before `npm ci` runs:

```dockerfile
RUN --mount=type=secret,id=npm_token,env=NPM_TOKEN \
    npm config set registry "${NPM_REGISTRY}" && \
    npm config set "//${NPM_REGISTRY#https://}:_authToken" "${NPM_TOKEN}"
```

The token is mounted in memory for this single `RUN` instruction only and is never written to a layer.

---

## Version Tagging

The pipeline reads `"version"` from `package.json`:

```json
{
  "name": "customer-portal",
  "version": "2.1.0",
  ...
}
```

| Branch | Tags pushed to ACR |
|---|---|
| Feature branch, version found | `<sha>`, `<branch>-<short-sha>`, `2.1.0-<short-sha>` |
| `main`, tag `2.1.0` is new | `<sha>`, `main-<short-sha>`, `2.1.0` |
| `main`, tag `2.1.0` exists | **Pipeline fails** — bump the version before merging |
| No `"version"` field | `<sha>`, `<branch>-<short-sha>` |

---

## Customizing the Dockerfile

### Create React App

CRA writes build output to `build/` rather than `dist/`:

```dockerfile
RUN npm run build
# Final stage:
COPY --from=build /app/build /usr/share/nginx/html
```

### Embedding the Git SHA in the app

Vite exposes build-time environment variables prefixed with `VITE_` to client code:

```dockerfile
ARG GIT_COMMIT_SHA=dev
ENV VITE_GIT_COMMIT_SHA=${GIT_COMMIT_SHA}
RUN npm run build
```

In your React component:

```tsx
const sha = import.meta.env.VITE_GIT_COMMIT_SHA ?? 'dev';
```

For CRA, use `REACT_APP_` prefix instead:

```dockerfile
ENV REACT_APP_GIT_COMMIT_SHA=${GIT_COMMIT_SHA}
```

```tsx
const sha = process.env.REACT_APP_GIT_COMMIT_SHA ?? 'dev';
```

### Multiple environments

To build for a specific environment (staging, production), map environment configurations to npm scripts in `package.json`:

```json
"scripts": {
  "build": "vite build",
  "build:staging": "vite build --mode staging",
  "build:production": "vite build --mode production"
}
```

Then call the appropriate script in the Dockerfile:

```dockerfile
RUN npm run build:production
```

Vite loads the corresponding `.env.production` or `.env.staging` file automatically.

### API proxy in nginx

If the React app calls a backend API and you want to proxy API requests through nginx (useful for avoiding CORS during development and for request signing in production):

```nginx
location /api/ {
    proxy_pass http://backend-service.namespace.svc.cluster.local:8080/api/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
```

In AKS, the `proxy_pass` target is the Kubernetes service DNS name.

---

## What the Pipeline Produces

For a successful non-dry-run run on `main` with `"version": "2.1.0"`:

### ACR tags (at `<platform-acr>/frontend-platform/customer-portal`)

| Tag | Use |
|---|---|
| `<40-char-sha>` | Kustomize manifests (immutable) |
| `main-<12-char-sha>` | Human navigation |
| `2.1.0` | Release version (main branch only) |

### Image contents

The final image is an nginx:alpine with the compiled static assets. It contains no Node.js, no source code, and no `node_modules`. Final image size is typically 30–50MB.

### Pipeline artifacts

| Artifact | Contents |
|---|---|
| `sbom-frontend-platform-customer-portal` | CycloneDX JSON SBOM |
| `provenance-frontend-platform-customer-portal` | JSON provenance record |

---

## Troubleshooting

**`npm ci` fails with 401 Unauthorized**
The `--mount=type=secret` syntax is missing, or `# syntax=docker/dockerfile:1` is not the first line. Both are required for BuildKit secrets.

**SPA routes return 404 on direct navigation or refresh**
The `nginx.conf` is missing the `try_files $uri $uri/ /index.html` directive. Copy the `nginx.conf` from this example as-is.

**Next.js advisory warning fires in every build**
If using Next.js with `output: 'standalone'` (SSR mode), this warning is expected and the image is still built. If you intended a static export, add `output: 'export'` to `next.config.js`. See [react-nextjs-ssr](../react-nextjs-ssr/) for the SSR pattern.

**Vite build output is in `dist/` but CRA output is in `build/`**
Check your `package.json` build script and adjust the `COPY --from=build` path in the final stage accordingly.

**The image is larger than expected**
Check for accidentally copied files. Common causes:
- `COPY . .` copies `.env` files, test directories, or CI config into the build stage (use a `.dockerignore` file)
- Source maps included in the build (set `sourcemap: false` in `vite.config.ts`)
- Assets not optimized (run an image optimizer on files in `src/assets/`)

### Recommended .dockerignore

Create `.dockerignore` in the repository root:

```
node_modules/
dist/
build/
.env*
!.env.example
coverage/
.nyc_output/
*.log
.git/
```

This prevents large directories from being included in the Docker build context, which reduces the context upload time and prevents accidental secret inclusion.
