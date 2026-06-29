# Template: steps/dockerfile-lint.yml

**Path:** `platform-templates/steps/dockerfile-lint.yml`
**Type:** Step template
**Called by:** `container-build-v2.yml` — Stage 2 (Build), Job: Build (first step)

---

## Description

Downloads a pinned version of Hadolint from GitHub releases and lints the tenant Dockerfile. The lint step runs before the Docker build, so Dockerfile quality issues are surfaced without consuming agent time on a full BuildKit build.

ERROR-level Hadolint findings cause the step — and therefore the Build stage — to fail. WARNING-level findings are written to the build log only and do not block the pipeline. Tenants can suppress specific rules project-wide by committing a `.hadolint.yaml` file to their build context.

---

## Summary

| Property | Value |
|---|---|
| Steps | 2 (bash) |
| Failure threshold | `error` (Hadolint exit code 1 on any ERROR finding) |
| Tool download source | GitHub releases (`hadolint-Linux-x86_64`) |
| Tool install location | `/tmp/hadolint` |
| Tenant config file | `.hadolint.yaml` in build context (optional) |
| Output variables | None |

---

## Parameters

| Name | Type | Default | Description |
|---|---|---|---|
| `dockerfilePath` | string | `Dockerfile` | Path to Dockerfile, relative to `buildContext` |
| `buildContext` | string | `.` | Docker build context directory |
| `hadolintVersion` | string | — | Pinned version to download; sourced from `resolveTools.HADOLINT_VERSION` |

---

## Steps

### Step 1 — Download Hadolint (bash)

Downloads the Hadolint binary for `linux-amd64` from:

```
https://github.com/hadolint/hadolint/releases/download/v<VERSION>/hadolint-Linux-x86_64
```

Saves to `/tmp/hadolint`, marks executable, and prints the version to confirm the download succeeded. Fails the step if the download or `chmod` fails (`set -euo pipefail`).

The version is received via the `HADOLINT_VERSION` environment variable (from `${{ parameters.hadolintVersion }}`), which in turn is the output of `resolveTools.HADOLINT_VERSION` from Stage 1.

---

### Step 2 — Run Hadolint (bash)

Resolves the target Dockerfile path as `$BUILD_CONTEXT/$DOCKERFILE_PATH` and runs:

```bash
/tmp/hadolint --failure-threshold error --format tty [--config ...] <dockerfile>
```

**Config file detection:** If `.hadolint.yaml` exists at `$BUILD_CONTEXT/.hadolint.yaml`, it is passed to Hadolint via `--config`. This allows tenants to suppress specific rules without modifying the platform template.

**Failure threshold:** `--failure-threshold error` means:
- `ERROR` findings → non-zero exit code → step fails → Build stage fails
- `WARNING` findings → printed to log → step passes

**Format:** `tty` produces human-readable output suitable for ADO log viewing.

---

## Hadolint Rule Categories

Common Hadolint rules that cause ERROR-level failures on this platform:

| Rule | Description |
|---|---|
| `DL3020` | Use `COPY` instead of `ADD` for local files |
| `DL3025` | Use JSON for `CMD` and `ENTRYPOINT` |
| `SC2086` | Double-quote variables to prevent word splitting (ShellCheck integration) |

Rules that are typically WARNING level and do not block the build include unpinned base image versions (`DL3007`) and `apt-get install` without version pinning (`DL3008`). Platform engineering sets the default thresholds; tenants may further adjust via `.hadolint.yaml`.

---

## Tenant Configuration

To suppress specific rules project-wide, create `.hadolint.yaml` in the build context directory:

```yaml
# .hadolint.yaml
ignore:
  - DL3008   # Allow unpinned apt-get packages

# Optionally override failure threshold for specific rules
override:
  warning:
    - DL3007  # Treat unpinned base image as warning (already default)
```

The pipeline automatically detects this file if it exists at `<buildContext>/.hadolint.yaml`. No pipeline parameter changes are required.

---

## Output Variables

None. This step does not emit any output variables. It either passes or fails the Build stage.

---

## Usage

This template is called exclusively by `container-build-v2.yml`. It is not intended to be called directly by tenant pipelines.

```yaml
# Inside container-build-v2.yml
- template: steps/dockerfile-lint.yml
  parameters:
    dockerfilePath: ${{ parameters.dockerfilePath }}
    buildContext: ${{ parameters.buildContext }}
    hadolintVersion: $(stageDependencies.Setup.Setup.outputs['resolveTools.HADOLINT_VERSION'])
```
