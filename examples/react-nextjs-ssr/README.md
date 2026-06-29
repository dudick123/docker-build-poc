# Example: Next.js SSR (Server-Side Rendering)

A complete, annotated example of the platform container build pipeline for a Next.js application running in server-side rendering mode — using React Server Components, API routes, `getServerSideProps`, or any feature that requires a running Node.js server.

For Next.js in static export mode (`output: 'export'`), or for plain React (Vite, CRA), see [../react/](../react/).

---

## Files in This Example

| File | Purpose |
|---|---|
| `azure-pipelines.yml` | ADO pipeline definition |
| `Dockerfile` | Three-stage Dockerfile: `deps` → `build` → `final` (Node.js runtime) |
| `README.md` | This document |

---

## How This Differs from the Static React Example

| Aspect | Static React / Next.js export | Next.js SSR (this example) |
|---|---|---|
| Final base image | `nginx:1.27-alpine` (~25MB) | `node:22-alpine` (~250–400MB) |
| Runtime process | nginx | Node.js (`server.js`) |
| `next.config.js` | `output: 'export'` | `output: 'standalone'` |
| Can use API routes | No | Yes |
| Can use `getServerSideProps` | No | Yes |
| Can use Server Components | No | Yes |
| AKS resource requirements | Low | Medium (Node.js GC overhead) |
| Pipeline warning | None | Advisory warning about missing `output: export` (expected — see below) |

---

## Required: next.config.js

```js
/** @type {import('next').NextConfig} */
const nextConfig = {
  // 'standalone' produces a self-contained production server at .next/standalone/
  // that includes only the runtime node_modules Next.js actually needs.
  // Without this, the Dockerfile cannot produce a minimal production image.
  output: 'standalone',
};

module.exports = nextConfig;
```

### Advisory warning from the pipeline

Because the React runtime validator checks for `output: export` (not `output: standalone`), it will emit an advisory warning on every build:

```
Next.js app detected without 'output: export' in next.config.js. SSR apps require a Node.js runtime in the final image, not nginx.
```

This warning is expected and accurate — the Dockerfile in this example correctly uses a Node.js final image. The warning does **not** fail the pipeline. It is informational, reminding teams to confirm their Dockerfile pattern matches their Next.js output mode.

---

## Required: package.json

The pipeline's React validator checks for `"next"` in the `package.json` dependencies. Ensure it is in `dependencies` (not just `devDependencies`):

```json
{
  "name": "storefront",
  "version": "1.5.0",
  "dependencies": {
    "next": "^14.2.0",
    "react": "^18.3.0",
    "react-dom": "^18.3.0"
  },
  "scripts": {
    "build": "next build",
    "start": "next start",
    "dev": "next dev"
  }
}
```

---

## How the Three-Stage Dockerfile Works

### Stage 1: `deps`

Installs all dependencies (including `devDependencies` needed to compile TypeScript). This stage is cached separately from the build — if `package.json` and `package-lock.json` are unchanged, `npm ci` is not re-run on the next build.

### Stage 2: `build`

Runs `next build`. With `output: 'standalone'`, the Next.js compiler produces:

```
.next/
  standalone/           ← self-contained server (server.js + minimal node_modules)
    node_modules/       ← only runtime deps; MUCH smaller than full node_modules
    server.js           ← production server entry point
    .next/server/       ← SSR modules
  static/               ← compiled static assets (hashed filenames)
public/                 ← pass-through static files
```

### Stage 3: `final`

Copies only the three directories needed at runtime:
- `.next/standalone/` — the server and its minimal dependencies
- `.next/static/` — static assets served at `/_next/static/`
- `public/` — static files served at `/`

The full `node_modules/` (hundreds of MB) stays in the `build` stage and is never copied to the final image.

---

## Version Tagging

The pipeline reads `"version"` from `package.json`:

```json
{
  "name": "storefront",
  "version": "1.5.0",
  ...
}
```

| Branch | Tags pushed to ACR |
|---|---|
| Feature branch, version found | `<sha>`, `<branch>-<short-sha>`, `1.5.0-<short-sha>` |
| `main`, tag `1.5.0` is new | `<sha>`, `main-<short-sha>`, `1.5.0` |
| `main`, tag `1.5.0` exists | **Pipeline fails** — bump the version before merging |
| No `"version"` field | `<sha>`, `<branch>-<short-sha>` |

---

## Customizing the Dockerfile

### Environment variables at runtime

The standalone server reads environment from the container environment (Kubernetes ConfigMap / Secret), not from `.env` files. Do not `COPY .env` into the final image — configure the running pod instead.

Variables prefixed with `NEXT_PUBLIC_` that are needed at runtime must be set at build time (they are inlined into the JS bundle by the Next.js compiler):

```dockerfile
ARG GIT_COMMIT_SHA=dev
ENV NEXT_PUBLIC_GIT_COMMIT_SHA=${GIT_COMMIT_SHA}
RUN npm run build
```

Variables that are only read server-side (`process.env.DATABASE_URL`, `process.env.API_SECRET`) should NOT be set in the Dockerfile — set them in the Kubernetes pod spec via Secret references.

### Custom server port

The `PORT` environment variable controls which port `server.js` listens on. The default is 3000. Change it at the pod level:

```yaml
env:
  - name: PORT
    value: "8080"
```

Update `EXPOSE 3000` in the Dockerfile and the Kubernetes Service `targetPort` to match.

### Health checks

Add a `/healthz` API route in Next.js:

```typescript
// app/api/healthz/route.ts (App Router)
export async function GET() {
  return Response.json({ status: 'ok' });
}

// pages/api/healthz.ts (Pages Router)
export default function handler(req, res) {
  res.status(200).json({ status: 'ok' });
}
```

Configure the Kubernetes liveness probe:

```yaml
livenessProbe:
  httpGet:
    path: /api/healthz
    port: 3000
  initialDelaySeconds: 30
  periodSeconds: 10
```

### Adding middleware

Next.js middleware (Edge Runtime) works with `output: 'standalone'`. No special Dockerfile changes are needed. Middleware runs on the Node.js server in standalone mode.

---

## npm Credential Injection

The platform pipeline injects credentials into every Docker build. The `deps` stage consumes them:

```dockerfile
RUN --mount=type=secret,id=npm_token,env=NPM_TOKEN \
    npm config set registry "${NPM_REGISTRY}" && \
    npm config set "//${NPM_REGISTRY#https://}:_authToken" "${NPM_TOKEN}"
```

The auth token is mounted in memory for the duration of this `RUN` instruction only. It is never written to a layer in `deps`, `build`, or `final`.

---

## AKS Resource Recommendations

Next.js SSR applications consume more memory than static sites (the Node.js V8 heap grows with request load). As a starting point:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

Monitor memory usage under load with `kubectl top pod` and adjust accordingly. The Node.js garbage collector can cause spiky memory usage — set `--max-old-space-size` if the pod hits memory limits:

```yaml
env:
  - name: NODE_OPTIONS
    value: "--max-old-space-size=384"
```

---

## What the Pipeline Produces

For a successful non-dry-run run on `main` with `"version": "1.5.0"`:

### ACR tags (at `<platform-acr>/frontend-platform/storefront`)

| Tag | Use |
|---|---|
| `<40-char-sha>` | Kustomize manifests (immutable) |
| `main-<12-char-sha>` | Human navigation |
| `1.5.0` | Release version (main branch only) |

### Pipeline artifacts

| Artifact | Contents |
|---|---|
| `sbom-frontend-platform-storefront` | CycloneDX JSON SBOM |
| `provenance-frontend-platform-storefront` | JSON provenance record |

---

## Troubleshooting

**Advisory warning: `Next.js app detected without 'output: export'`**
This is expected for SSR apps. The warning fires because the platform validator checks for `output: export`, not `output: standalone`. The pipeline still succeeds. If this warning is confusing, document it in your repository's `README.md` so team members understand it is expected.

**`next build` fails with "Missing output configuration"**
`output: 'standalone'` must be in `next.config.js`. Without it, the `COPY --from=build /app/.next/standalone` in the final stage will fail because the directory does not exist.

**`server.js` not found at runtime**
The standalone directory was not copied correctly. Verify that `output: 'standalone'` is in `next.config.js` and that the `COPY --from=build /app/.next/standalone ./` path is correct. Check the build stage output for any Next.js compilation errors.

**Static assets return 404 (`/_next/static/...`)**
The `.next/static/` directory must be copied separately from standalone — it is not included in the standalone output. Verify the `COPY --from=build /app/.next/static ./.next/static` line is present in the final stage.

**`public/` files return 404 (`/images/...`, `/fonts/...`)**
The `public/` directory must also be copied separately. Add `COPY --from=build /app/public ./public` to the final stage.

**Container OOMKilled in AKS**
Increase the memory limit and/or set `NODE_OPTIONS=--max-old-space-size=<MB>`. The Node.js heap defaults to ~1.5GB on machines with sufficient RAM, which often exceeds the container limit.

**`npm ci` fails with 401 Unauthorized**
The `--mount=type=secret` syntax is missing or `# syntax=docker/dockerfile:1` is not the first line of the Dockerfile. Both are required. Check that the Dockerfile starts with exactly `# syntax=docker/dockerfile:1` on its own line.
