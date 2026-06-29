# Quick Start: Go

This guide covers what the pipeline requires from a Go application repository. Read [QUICK-START.md](QUICK-START.md) first for prerequisites and the base `azure-pipelines.yml` structure.

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
    appName: my-service
    runtimeType: go
```

---

## Build Pattern

Go uses a self-contained multi-stage Dockerfile. The pipeline agent runs only `docker build` — no Go toolchain is installed on the agent. All compilation, dependency resolution, and testing happens inside the Dockerfile.

---

## Repository Requirements

### go.mod (required)

The pipeline validates that `go.mod` exists at the root of the build context before the image build starts. If it is absent, Stage 2 fails with:

```
go.mod not found at /agent/.../go.mod. The build context root must contain go.mod for runtimeType: go.
```

This is a hard failure — there is no bypass.

**Correct layout:**

```
my-service/
  go.mod          ← required here (at buildContext root)
  go.sum
  main.go
  Dockerfile
  azure-pipelines.yml
```

If your Go module root is a subdirectory, set `buildContext` to that directory:

```yaml
parameters:
  tenantName: my-team
  appName: my-service
  runtimeType: go
  buildContext: services/api   # go.mod must exist at services/api/go.mod
```

### VERSION file (optional)

To enable version tags in ACR, create a `VERSION` file at the build context root containing a single line with the version string:

```
1.4.2
```

Rules:
- The pipeline reads the first non-empty line and trims surrounding whitespace
- The value must not equal `latest` (the pipeline asserts this)
- On the `main` branch, the tag must not already exist in ACR (bump before merging)
- On other branches, the tag is pushed as `<version>-<12-char-sha>` (always safe)
- If the file is absent, the version tag is skipped — only the full SHA and alias tags are pushed

---

## Reference Dockerfile

```dockerfile
# syntax=docker/dockerfile:1

# ── build stage ──────────────────────────────────────────────────────────────
FROM golang:1.22-alpine AS build

WORKDIR /src

# Cache dependencies separately from source
COPY go.mod go.sum ./
RUN go mod download

COPY . .

# Build args injected by the pipeline
ARG GIT_COMMIT_SHA=dev

RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-s -w -X main.version=${GIT_COMMIT_SHA}" \
    -o /out/app \
    ./cmd/server

# ── final stage ──────────────────────────────────────────────────────────────
FROM gcr.io/distroless/static-debian12:nonroot AS final

COPY --from=build /out/app /app

EXPOSE 8080

ENTRYPOINT ["/app"]
```

### Key points

- **`GIT_COMMIT_SHA` build arg** — injected automatically by the pipeline as the full 40-character Git SHA. Use it for version embedding in the binary. The `ARG` default (`dev`) makes local builds work without the pipeline.
- **`CGO_ENABLED=0`** — required for distroless or scratch final images.
- **Distroless final image** — `gcr.io/distroless/static-debian12:nonroot` produces the smallest possible attack surface and runs as a non-root user by default. Use `gcr.io/distroless/base-debian12:nonroot` if your binary needs glibc.
- **No `latest` base image tags** — Hadolint will warn on unpinned tags. Use digest-pinned or explicit version tags for base images in production builds.

---

## What the Pipeline Validates

The Go runtime step (`steps/runtime/go.yml`) runs after the Docker build and performs:

1. Asserts `go.mod` exists at `<buildContext>/go.mod` — hard fail if absent
2. Reads the first non-empty line of `VERSION` if the file exists — emits as `GO_VERSION` for the Publish stage tag logic
3. Logs a notice if no `VERSION` file is found (version tag will be skipped)

This step does not run `go build`, `go test`, or any Go toolchain command on the agent. All Go compilation and testing happens inside the Dockerfile.

---

## Version Tagging Summary

| Scenario | Tags pushed to ACR |
|---|---|
| No `VERSION` file | `<40-char-sha>`, `<branch>-<12-char-sha>` |
| `VERSION` file present, non-main branch | `<40-char-sha>`, `<branch>-<12-char-sha>`, `<version>-<12-char-sha>` |
| `VERSION` file present, `main` branch, tag does not exist | `<40-char-sha>`, `main-<12-char-sha>`, `<version>` |
| `VERSION` file present, `main` branch, tag already exists | Pipeline fails — bump the `VERSION` file before merging |

---

## Common Issues

**`go.mod not found`**
The `go.mod` file is missing from the build context root or the `buildContext` parameter does not point to the correct directory. Verify the path and that the file is committed.

**`go: github.com/... is not in module cache`**
The module download (`go mod download`) is failing inside the Docker build. Check that your base Go image has network access to the Go module proxy or that a private proxy is configured via `GONOSUMCHECK` / `GOPROXY`.

**Binary not found in final image**
Verify the `COPY --from=build` path matches the `-o` output path in `go build`. The paths in the Dockerfile must be consistent.

**Build arg `GIT_COMMIT_SHA` shows `dev` in production**
This means the binary was built locally, not through the pipeline. The pipeline always passes `--build-arg GIT_COMMIT_SHA=<sha>`. The `ARG` default is intentionally `dev` to make local builds work.
