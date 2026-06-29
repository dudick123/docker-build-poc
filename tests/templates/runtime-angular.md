# Test Cases: steps/runtime/angular.yml

**Template:** `platform-templates/steps/runtime/angular.yml`
**Pattern:** B — npm ci runs on agent; Angular CLI build runs on agent before docker build
**Stage:** Build (Stage 2) — runs before docker-build.yml
**Steps under test:** `validateAngularProject` (bash), `extractAngularVersion` (bash)

---

## TC-ANG-001 — Happy path: package.json with @angular/core, version extracted

**Level:** L2
**Type:** Positive

**Precondition:** `fixtures/projects/package.json.angular` committed as `package.json` in the build context. Content:
```json
{
  "name": "test-app",
  "version": "3.2.1",
  "dependencies": {
    "@angular/core": "^17.0.0"
  },
  "scripts": {
    "build": "ng build --output-path=dist/app"
  }
}
```

**Expected result:**
- `validateAngularProject` step passes
- Build log contains: `@angular/core detected: ^17.0.0`
- `extractAngularVersion` emits `RUNTIME_VERSION=3.2.1`
- Pipeline continues to `dockerfile-lint.yml`

---

## TC-ANG-002 — package.json absent: pipeline fails

**Level:** L2
**Type:** Negative

**Precondition:** No `package.json` in the build context.

**Expected result:** `validateAngularProject` step fails. Build log contains:
```
##[error]package.json not found at <buildContext>/package.json — Angular runtime requires a package.json
```

---

## TC-ANG-003 — package.json present but @angular/core missing: advisory warning

**Level:** L2
**Type:** Positive (advisory)

**Precondition:** `package.json` with `react` and `react-dom` dependencies but no `@angular/core`.

**Expected result:** `validateAngularProject` step passes (exit 0). Build log contains:
```
Warning: @angular/core not found in package.json dependencies. Ensure this is intentional — runtimeType 'angular' expects an Angular project.
```

Pipeline continues. This is advisory only — the pipeline does not fail.

---

## TC-ANG-004 — npm secret mount: SYSTEM_ACCESSTOKEN passed via env, not --build-arg

**Level:** L2
**Type:** Security — secret handling

**Precondition:** Angular Dockerfile using:
```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=secret,id=npm_token,env=NPM_TOKEN npm config set ...
```

**Expected result:**
- Docker build step passes `--secret id=npm_token,env=SYSTEM_ACCESSTOKEN`
- Build log does NOT contain the `SYSTEM_ACCESSTOKEN` value in plaintext
- `docker history` on the resulting image does NOT reveal the token in any layer

**Verification:** Run `docker history <image>` and grep for `SYSTEM_ACCESSTOKEN`. Expected: no match.

---

## TC-ANG-005 — runtimeVersion propagates to Stage 4 (cross-stage variable)

**Level:** L2/L3
**Type:** Positive — cross-stage variable propagation

**Precondition:** `package.json` with `"version": "5.0.0"`.

**Verification:** In Stage 4, `RUNTIME_VERSION` arrives via:
```
$(stageDependencies.Build.Build.outputs['extractAngularVersion.RUNTIME_VERSION'])
```

Confirm the value is `5.0.0` in a diagnostic `echo` step at the start of Stage 4.
