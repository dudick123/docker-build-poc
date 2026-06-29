# Template: steps/runtime/angular.yml

**Path:** `platform-templates/steps/runtime/angular.yml`
**Type:** Step template (runtime validator)
**Called by:** `container-build-v2.yml` — Stage 2 (Build), Job: Build, when `runtimeType: angular`

---

## Description

Validates the Angular project structure and extracts the application version from `package.json` for optional ACR version tagging. All checks are advisory — no hard failures are possible from this step.

Angular uses Pattern A: `npm ci` and `ng build` run entirely inside the Dockerfile using BuildKit secrets for npm authentication. No Node.js toolchain runs on the pipeline agent. npm credentials for the platform Azure Artifacts feed are injected by `docker-build.yml` (which runs before this step) and are available to all runtimes — Angular Dockerfiles simply use them.

---

## Summary

| Property | Value |
|---|---|
| Steps | 1 (bash, named `angularRuntime`) |
| Hard failures | None |
| Advisory warnings | Missing `package.json`; `ng` not in `package.json` scripts |
| Output variables | `ANGULAR_VERSION` (empty string if not found) |

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

### Step 1 — `angularRuntime` (bash)

**`package.json` presence check (advisory):**

If `package.json` is not found at `$BUILD_CONTEXT/package.json`:
- Emits `##vso[task.logissue type=warning]No package.json found in <context>. Expected package.json for runtimeType: angular.`
- Emits `ANGULAR_VERSION` as empty string
- Exits cleanly (exit 0)

**`ng` in scripts check (advisory):**

If `package.json` exists but the string `"ng` does not appear in the file:
- Emits `##vso[task.logissue type=warning]'ng' not found in package.json scripts. Confirm this is an Angular project for runtimeType: angular.`

The check uses a simple grep (`grep -q '"ng'`) — it looks for the presence of the string anywhere in the file, not specifically in the `scripts` object. This is sufficient to detect standard Angular CLI setups.

**Version extraction:**

Reads the `"version"` field from `package.json` using `grep -oP '"version"\s*:\s*"\K[^"]+'`. Emits the first match as `ANGULAR_VERSION`.

If no `version` field is found, emits an empty string and logs:
```
No 'version' field found in package.json — version tagging will be skipped for this build.
```

---

## Output Variables

| Variable | Step name | Cross-stage reference | Notes |
|---|---|---|---|
| `ANGULAR_VERSION` | `angularRuntime` | `$(stageDependencies.Build.Build.outputs['angularRuntime.ANGULAR_VERSION'])` | Empty string if `package.json` absent or has no `version` field |

---

## npm Credential Injection

npm credentials are injected by `docker-build.yml` (which runs before this step), not by this template. Two values are passed to every `docker build` invocation:

| Value | Mechanism | Available in Dockerfile as |
|---|---|---|
| Platform npm registry URL | `--build-arg NPM_REGISTRY=<url>` | `ARG NPM_REGISTRY` |
| Azure Artifacts auth token | `--secret id=npm_token,env=SYSTEM_ACCESSTOKEN` | `--mount=type=secret,id=npm_token,env=NPM_TOKEN` |

The Angular Dockerfile must consume these to authenticate against the platform Azure Artifacts feed.

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

This template is called exclusively by `container-build-v2.yml` when `runtimeType: angular`.

```yaml
# Inside container-build-v2.yml
- ${{ if eq(parameters.runtimeType, 'angular') }}:
  - template: steps/runtime/angular.yml
    parameters:
      buildContext: ${{ parameters.buildContext }}
      dockerfilePath: ${{ parameters.dockerfilePath }}
      tenantName: ${{ parameters.tenantName }}
      appName: ${{ parameters.appName }}
```
