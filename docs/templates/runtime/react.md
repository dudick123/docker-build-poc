# Template: steps/runtime/react.yml

**Path:** `platform-templates/steps/runtime/react.yml`
**Type:** Step template (runtime validator)
**Called by:** `container-build-v2.yml` — Stage 2 (Build), Job: Build, when `runtimeType: react`

---

## Description

Validates the React or Next.js project structure, detects SSR-mode Next.js apps that may require a different Dockerfile pattern, and extracts the application version from `package.json` for optional ACR version tagging. All checks are advisory — no hard failures are possible from this step.

`runtimeType: react` covers both plain React (Vite, Create React App) and Next.js. The same validator runs for both.

React and Next.js use Pattern A: `npm ci` and the framework build command run entirely inside the Dockerfile using BuildKit secrets for npm authentication. No Node.js toolchain runs on the pipeline agent.

---

## Summary

| Property | Value |
|---|---|
| Steps | 1 (bash, named `reactRuntime`) |
| Hard failures | None |
| Advisory warnings | Missing `package.json`; Next.js app without `output: export` |
| Output variables | `REACT_VERSION` (empty string if not found) |

---

## Parameters

| Name | Type | Default | Description |
|---|---|---|---|
| `buildContext` | string | `.` | Build context directory |
| `dockerfilePath` | string | `Dockerfile` | Not used in logic; present for parameter consistency |
| `tenantName` | string | — | Not used in logic; present for parameter consistency |
| `appName` | string | — | Not used in logic; present for parameter consistency |

---

## Steps

### Step 1 — `reactRuntime` (bash)

**`package.json` presence check (advisory):**

If `package.json` is not found at `$BUILD_CONTEXT/package.json`:
- Emits `##vso[task.logissue type=warning]No package.json found in <context>. Expected package.json for runtimeType: react.`
- Emits `REACT_VERSION` as empty string
- Exits cleanly (exit 0)

**Next.js SSR detection (advisory):**

If `package.json` exists and contains the string `"next"` (indicating Next.js is a dependency):

1. Checks `next.config.js` and `next.config.mjs` for the pattern `output.*export`
2. If neither config file contains this pattern, emits:

```
##vso[task.logissue type=warning]Next.js app detected without 'output: export' in next.config.js. SSR apps require a Node.js runtime in the final image, not nginx. See platform reference Dockerfile at docs/reference-dockerfiles/react-ssr.Dockerfile.
```

This warning is important: a Next.js SSR app served by nginx will return HTML that the browser cannot execute (server components, API routes, etc.). The warning is not a pipeline failure — it is guidance to the tenant to use the correct Dockerfile pattern.

**Version extraction:**

Reads the `"version"` field from `package.json` using `grep -oP '"version"\s*:\s*"\K[^"]+'`. Emits the first match as `REACT_VERSION`.

If no `version` field is found, emits an empty string and logs:
```
No 'version' field found in package.json — version tagging will be skipped for this build.
```

---

## Output Variables

| Variable | Step name | Cross-stage reference | Notes |
|---|---|---|---|
| `REACT_VERSION` | `reactRuntime` | `$(stageDependencies.Build.Build.outputs['reactRuntime.REACT_VERSION'])` | Empty string if `package.json` absent or has no `version` field |

---

## Next.js SSR vs Static Export

The pipeline detects this distinction based on `next.config.js` content:

| Next.js mode | `next.config.js` | Correct final image | Pipeline behavior |
|---|---|---|---|
| Static export | `output: 'export'` | nginx:alpine | No warning |
| Standalone SSR | `output: 'standalone'` | node:alpine | Warning emitted |
| SSR (default, no output config) | *(absent)* | node:alpine | Warning emitted |

The warning does not block the build. If the tenant is intentionally using SSR and has authored the correct Node.js final image, the warning can be disregarded. Contact platform engineering if the warning is a persistent concern.

---

## npm Credential Injection

npm credentials are injected by `docker-build.yml` (which runs before this step). Both values are available to React/Next.js Dockerfiles via `ARG NPM_REGISTRY` and `--mount=type=secret,id=npm_token`:

| Value | Mechanism | Available in Dockerfile as |
|---|---|---|
| Platform npm registry URL | `--build-arg NPM_REGISTRY=<url>` | `ARG NPM_REGISTRY` |
| Azure Artifacts auth token | `--secret id=npm_token,env=SYSTEM_ACCESSTOKEN` | `--mount=type=secret,id=npm_token,env=NPM_TOKEN` |

---

## Version Tagging Behavior

| Condition | Tags pushed to ACR |
|---|---|
| No `"version"` in `package.json` | `<40-char-sha>`, `<branch>-<12-char-sha>` |
| Version found, non-main branch | `<40-char-sha>`, `<branch>-<12-char-sha>`, `<version>-<12-char-sha>` |
| Version found, `main` branch, tag is new | `<40-char-sha>`, `main-<12-char-sha>`, `<version>` |
| Version found, `main` branch, tag exists | Pipeline fails — bump the version in `package.json` before merging |

---

## Usage

This template is called exclusively by `container-build-v2.yml` when `runtimeType: react`.

```yaml
# Inside container-build-v2.yml
- ${{ if eq(parameters.runtimeType, 'react') }}:
  - template: steps/runtime/react.yml
    parameters:
      buildContext: ${{ parameters.buildContext }}
      dockerfilePath: ${{ parameters.dockerfilePath }}
      tenantName: ${{ parameters.tenantName }}
      appName: ${{ parameters.appName }}
```
