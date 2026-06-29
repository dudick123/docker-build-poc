# Template: steps/runtime/go.yml

**Path:** `platform-templates/steps/runtime/go.yml`
**Type:** Step template (runtime validator)
**Called by:** `container-build-v2.yml` — Stage 2 (Build), Job: Build, when `runtimeType: go`

---

## Description

Validates that the build context contains the required Go project structure and extracts a version string for optional ACR version tagging. This step runs after the Docker build completes — it validates the source repository layout, not the built image.

Go uses Pattern A: the entire build toolchain runs inside the Dockerfile. No Go compiler, `go get`, or `go test` commands run on the pipeline agent.

---

## Summary

| Property | Value |
|---|---|
| Steps | 1 (bash, named `goRuntime`) |
| Hard failures | `go.mod` not found at build context root |
| Advisory warnings | None (version absence is a notice, not a warning) |
| Output variables | `GO_VERSION` (empty string if no `VERSION` file) |

---

## Parameters

| Name | Type | Default | Description |
|---|---|---|---|
| `buildContext` | string | `.` | Build context directory; `go.mod` must exist here |

---

## Steps

### Step 1 — `goRuntime` (bash)

**go.mod assertion (hard fail):**

Resolves the path as `$BUILD_SOURCESDIRECTORY/$BUILD_CONTEXT/go.mod` and fails if the file does not exist:

```
##vso[task.logissue type=error]go.mod not found at <path>. The build context root must contain go.mod for runtimeType: go.
exit 1
```

This is unconditional — there is no way to bypass this check within the pipeline.

**VERSION file read (optional):**

If `$BUILD_SOURCESDIRECTORY/$BUILD_CONTEXT/VERSION` exists, reads the first non-empty line and strips surrounding whitespace. Emits the result as `GO_VERSION`.

If the file does not exist, `GO_VERSION` is emitted as an empty string and a notice is written to the log:

```
No VERSION file found at <path> — version tag will be skipped for this build.
```

---

## Output Variables

| Variable | Step name | Cross-stage reference | Notes |
|---|---|---|---|
| `GO_VERSION` | `goRuntime` | `$(stageDependencies.Build.Build.outputs['goRuntime.GO_VERSION'])` | Empty string if no `VERSION` file; version tag skipped in Publish stage |

---

## Repository Requirements

### go.mod (required)

Must exist at `<buildContext>/go.mod`. If the module root is a subdirectory, the tenant must set the `buildContext` parameter accordingly.

### VERSION file (optional)

Plain text file at `<buildContext>/VERSION`. First non-empty line is used. Whitespace is trimmed.

```
# Example VERSION file
1.4.2
```

Rules:
- Must not equal `latest` (the Publish stage asserts this)
- On `main` branch: must not already be a tag in ACR (bumping is enforced before merge)
- On other branches: tag is pushed as `<version>-<12-char-sha>`, which is always unique

---

## Version Tagging Behavior

| Condition | Tags pushed to ACR |
|---|---|
| No `VERSION` file | `<40-char-sha>`, `<branch>-<12-char-sha>` |
| `VERSION` present, non-main branch | `<40-char-sha>`, `<branch>-<12-char-sha>`, `<version>-<12-char-sha>` |
| `VERSION` present, `main` branch, tag is new | `<40-char-sha>`, `main-<12-char-sha>`, `<version>` |
| `VERSION` present, `main` branch, tag exists | Pipeline fails — bump `VERSION` before merging |

---

## Usage

This template is called exclusively by `container-build-v2.yml` when `runtimeType: go`. It is not intended to be called directly by tenant pipelines.

```yaml
# Inside container-build-v2.yml
- ${{ if eq(parameters.runtimeType, 'go') }}:
  - template: steps/runtime/go.yml
    parameters:
      buildContext: ${{ parameters.buildContext }}
```
