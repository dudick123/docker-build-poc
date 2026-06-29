# Test Cases: steps/runtime/react.yml

**Template:** `platform-templates/steps/runtime/react.yml`
**Pattern:** B — npm ci runs on agent; framework build runs on agent before docker build
**Stage:** Build (Stage 2) — runs before docker-build.yml
**Steps under test:** `validateReactProject` (bash), `detectNextJs` (bash), `extractReactVersion` (bash)

---

## TC-REACT-001 — Happy path: Vite/CRA project, package.json with react dep, version extracted

**Level:** L2
**Type:** Positive

**Precondition:** `fixtures/projects/package.json.react` committed as `package.json`. Content:
```json
{
  "name": "test-app",
  "version": "1.0.0",
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "scripts": {
    "build": "vite build"
  }
}
```
No `next` dependency. No `next.config.js`.

**Expected result:**
- `validateReactProject` passes
- Build log contains: `react detected: ^18.2.0`
- `detectNextJs` emits `NEXT_JS=false`
- `extractReactVersion` emits `RUNTIME_VERSION=1.0.0`
- Pipeline continues to `dockerfile-lint.yml`

---

## TC-REACT-002 — package.json absent: pipeline fails

**Level:** L2
**Type:** Negative

**Precondition:** No `package.json` in the build context.

**Expected result:** `validateReactProject` step fails. Build log contains:
```
##[error]package.json not found at <buildContext>/package.json — React runtime requires a package.json
```

---

## TC-REACT-003 — package.json present but no react dependency: advisory warning

**Level:** L2
**Type:** Positive (advisory)

**Precondition:** `package.json` with `vue` and `vue-router` but no `react`.

**Expected result:** `validateReactProject` step passes (exit 0). Build log contains:
```
Warning: 'react' not found in package.json dependencies. Ensure this is intentional — runtimeType 'react' expects a React/Next.js project.
```

Pipeline continues.

---

## TC-REACT-004 — Next.js project detection: next.config.js present

**Level:** L2
**Type:** Positive

**Precondition:** `fixtures/projects/next.config.js` committed. Content (static export):
```js
/** @type {import('next').Config} */
const nextConfig = {
  output: 'export',
};
module.exports = nextConfig;
```

**Expected result:**
- `detectNextJs` detects `next.config.js`
- Build log contains: `Next.js project detected (next.config.js found)`
- `NEXT_JS=true` output variable emitted
- Build log contains the advisory: `output: 'export' detected — static export mode. Confirm your Dockerfile uses the correct COPY path (out/ not .next/).`

---

## TC-REACT-005 — Next.js SSR: next.config.js with output: 'standalone'

**Level:** L2
**Type:** Positive

**Precondition:** `next.config.js`:
```js
const nextConfig = {
  output: 'standalone',
};
module.exports = nextConfig;
```

**Expected result:**
- `detectNextJs` emits `NEXT_JS=true`
- Build log contains: `output: 'standalone' detected — SSR/standalone mode. Confirm your Dockerfile copies .next/standalone/, .next/static/, and public/.`

Pipeline does not fail. These are advisory messages only.

---

## TC-REACT-006 — npm secret mount: SYSTEM_ACCESSTOKEN passed via env, not --build-arg

**Level:** L2
**Type:** Security — secret handling

**Precondition:** React Dockerfile using:
```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=secret,id=npm_token,env=NPM_TOKEN npm config set ...
```

**Expected result:**
- Docker build step passes `--secret id=npm_token,env=SYSTEM_ACCESSTOKEN`
- Build log does NOT contain the `SYSTEM_ACCESSTOKEN` value in plaintext
- `docker history` on the built image does NOT reveal the token in any layer

---

## TC-REACT-007 — runtimeVersion propagates to Stage 4 (cross-stage variable)

**Level:** L2/L3
**Type:** Positive — cross-stage variable propagation

**Precondition:** `package.json` with `"version": "2.5.0"`.

**Verification:** In Stage 4, `RUNTIME_VERSION` arrives via:
```
$(stageDependencies.Build.Build.outputs['extractReactVersion.RUNTIME_VERSION'])
```

Add a diagnostic `echo` step at the beginning of Stage 4 to confirm the value is `2.5.0`.
