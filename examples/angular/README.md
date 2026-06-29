# Example: Angular

A complete, annotated example of the platform container build pipeline for an Angular application. All files can be copied into an application repository root and customized.

---

## Files in This Example

| File | Purpose |
|---|---|
| `azure-pipelines.yml` | ADO pipeline definition |
| `Dockerfile` | Two-stage Dockerfile: `build` (Node.js) → `final` (nginx) |
| `nginx.conf` | nginx server block with HTML5 routing, caching, and security headers |
| `README.md` | This document |

---

## How It Works

### npm credential injection

The platform pipeline automatically injects npm credentials into every Docker build. No changes to `azure-pipelines.yml` are needed to enable this — it is always on.

Two values are passed via the BuildKit invocation:

```
--build-arg NPM_REGISTRY=<azure-artifacts-npm-feed-url>
--secret id=npm_token,env=SYSTEM_ACCESSTOKEN
```

Your Dockerfile must consume these. The critical pattern is:

```dockerfile
RUN --mount=type=secret,id=npm_token,env=NPM_TOKEN \
    npm config set registry "${NPM_REGISTRY}" && \
    npm config set "//${NPM_REGISTRY#https://}:_authToken" "${NPM_TOKEN}"
```

The `--mount=type=secret` syntax is a BuildKit feature. It makes the secret available **only during this RUN instruction** as an environment variable (`NPM_TOKEN`). The secret is never written to any image layer, never visible in `docker history`, and never accessible from the running container.

### Stage 2: Runtime validator

After the Docker build, `steps/runtime/angular.yml` runs and performs two advisory checks:
1. Warns if `package.json` is missing
2. Warns if `ng` is not found in `package.json` scripts

Neither warning fails the pipeline. The step then extracts the `"version"` field from `package.json` for ACR version tagging.

---

## Prerequisites

1. **`# syntax=docker/dockerfile:1`** must be the first line of your Dockerfile. BuildKit secrets require this directive.
2. **`package.json`** should exist at the build context root.
3. **`package-lock.json`** must exist for `npm ci` to work. Run `npm install` locally to generate it if absent.
4. **`ng` in `package.json` scripts**. The standard Angular CLI setup produces:
   ```json
   "scripts": {
     "ng": "ng",
     "build": "ng build",
     "test": "ng test",
     "lint": "ng lint"
   }
   ```

---

## Version Tagging

The pipeline reads the `"version"` field from `package.json`:

```json
{
  "name": "admin-portal",
  "version": "4.2.0",
  ...
}
```

| Branch | Tags pushed to ACR |
|---|---|
| Feature branch, version found | `<sha>`, `<branch>-<short-sha>`, `4.2.0-<short-sha>` |
| `main`, tag `4.2.0` is new | `<sha>`, `main-<short-sha>`, `4.2.0` |
| `main`, tag `4.2.0` exists | **Pipeline fails** — bump the version before merging |
| No `"version"` field | `<sha>`, `<branch>-<short-sha>` |

---

## Customizing the Dockerfile

### Different output directory

The Angular CLI writes output to the directory specified by `outputPath` in `angular.json`. If your project uses a path other than `dist/<project-name>`, either:

a) Override in the build command (as done in this example):
   ```dockerfile
   RUN npm run build -- --output-path=dist/app
   ```

b) Or update the COPY in the final stage to match whatever `angular.json` specifies:
   ```dockerfile
   COPY --from=build /app/dist/admin-portal /usr/share/nginx/html
   ```

### Multi-project Angular workspace

If the workspace contains multiple projects, specify the project name:

```dockerfile
RUN npm run build -- --project=admin-portal --output-path=dist/admin-portal
COPY --from=build /app/dist/admin-portal /usr/share/nginx/html
```

### Build configuration (environments)

Angular supports named build configurations (e.g., `production`, `staging`) defined in `angular.json`. The default `npm run build` typically maps to `--configuration=production`. To use a different configuration:

```dockerfile
RUN npm run build -- --configuration=staging --output-path=dist/app
```

Or add a script to `package.json`:
```json
"scripts": {
  "build:staging": "ng build --configuration=staging"
}
```

Then call it:
```dockerfile
RUN npm run build:staging -- --output-path=dist/app
```

### Embedding the Git SHA in the Angular app

Option 1 — as an environment variable read at runtime (not recommended for static builds):
The `GIT_COMMIT_SHA` build arg is set as an `ENV` in the build stage. Angular's SSR or an nginx custom header can surface it, but static builds cannot read environment variables.

Option 2 — inject via Angular's `environment.ts` at build time using a define:

In `angular.json`, add to the `build` options:
```json
"define": {
  "process.env.GIT_SHA": "\"$GIT_COMMIT_SHA\""
}
```

Then in `environment.ts`:
```typescript
export const environment = {
  production: true,
  gitSha: process.env['GIT_SHA'] ?? 'dev'
};
```

### Customizing nginx

The `nginx.conf` in this example is a production-ready starting point. Common customizations:

**API proxy:** Route `/api/*` requests to a backend service:
```nginx
location /api/ {
    proxy_pass http://backend-service:8080/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
}
```

**Custom error pages:**
```nginx
error_page 404 /index.html;   # Let Angular handle 404s client-side
```

**HTTPS termination:** TLS is handled by the AKS ingress controller (e.g., NGINX Ingress + cert-manager), not by the application container. Do not add TLS configuration to the container nginx.

---

## Hadolint Considerations

| Rule | Issue | Fix |
|---|---|---|
| `DL3016` | `npm install -g` (global package install) | Use `npx` or local `./node_modules/.bin/` |
| `DL3018` | `apk add` without `--no-cache` | Add `--no-cache` to Alpine `apk` commands |
| `SC2046` | Unquoted command substitution | Quote `$()` subshells in RUN scripts |

If you add additional `RUN` steps with shell scripts, Hadolint applies ShellCheck rules. Common issues in multi-command `RUN` steps: unquoted variables, missing `set -e`, `set -u`.

---

## What the Pipeline Produces

For a successful non-dry-run run on `main` with `"version": "4.2.0"`:

### ACR tags (at `<platform-acr>/frontend-platform/admin-portal`)

| Tag | Use |
|---|---|
| `<40-char-sha>` | Kustomize manifests (immutable) |
| `main-<12-char-sha>` | Human navigation |
| `4.2.0` | Release version (main branch only) |

### Image contents

The final image contains only:
- nginx binary and its dependencies (~25MB total)
- Compiled Angular static assets in `/usr/share/nginx/html`
- The custom `nginx.conf`

`node_modules/`, TypeScript source, and Angular CLI are NOT in the final image.

### Pipeline artifacts

| Artifact | Contents |
|---|---|
| `sbom-frontend-platform-admin-portal` | CycloneDX JSON SBOM |
| `provenance-frontend-platform-admin-portal` | JSON provenance record |

---

## Troubleshooting

**`npm ci` fails with 401 Unauthorized**
The `--mount=type=secret` in the Dockerfile is missing, or the `# syntax=docker/dockerfile:1` directive is absent from line 1. Both are required. The token is not passed via `--build-arg` — it is only available through the BuildKit secret mount.

**`ng: not found` during `npm run build`**
Angular CLI is not in `node_modules/.bin`. Verify `@angular/cli` is in `devDependencies` in `package.json` and the `npm ci` step completed successfully.

**`No 'ng' found in package.json scripts` warning but build succeeds**
The advisory check greps for the string `"ng` in `package.json`. If your build script calls `ng` indirectly (e.g., via a Makefile or a wrapper script), the grep may not match. The warning is advisory — ignore it if your build is working correctly.

**Angular routing returns 404 on page refresh**
The `nginx.conf` is missing or the `try_files $uri $uri/ /index.html` line is absent. Copy the `nginx.conf` from this example and ensure it is included in the Dockerfile via `COPY nginx.conf /etc/nginx/conf.d/default.conf`.

**Large final image size**
Run `docker history <image>` to identify the large layers. Common causes:
- `node_modules/` accidentally copied to the final stage (check your `COPY` sources)
- Source maps included in the Angular build (set `"sourceMap": false` in `angular.json` production configuration)
- Unoptimized images included in `src/assets/`
