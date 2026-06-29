# Example: Go

A complete, annotated example of the platform container build pipeline for a Go application. All files in this directory are intended to be copied into the root of an application repository and customized.

---

## Files in This Example

| File | Purpose |
|---|---|
| `azure-pipelines.yml` | ADO pipeline definition; the only pipeline file your repository needs |
| `Dockerfile` | Multi-stage Dockerfile; compiles the binary and produces a distroless final image |
| `README.md` | This document |

---

## How It Works

### Pipeline file

The `azure-pipelines.yml` file contains three things:
1. A trigger (run on every push to `main`)
2. A `resources` block pointing at the `platform-templates` ADO repository
3. An `extends` block with your six parameters

The `extends` pattern in ADO prevents adding arbitrary stages or jobs — the platform template owns the full pipeline shape. Your repository supplies parameters; the template supplies everything else.

### Stages executed

```
Setup → Build → Sign & Attest → Publish → Notify
```

| Stage | What runs | Time estimate |
|---|---|---|
| Setup | Tool-version resolution, parameter validation | ~15s |
| Build | Hadolint lint, BuildKit image build, go.mod assertion | ~60–120s (first run); ~20s (cache hit) |
| Sign & Attest | Syft SBOM, Cosign sign + attest + verify | ~45s |
| Publish | Tag push (3 tags), digest verify, provenance artifact | ~30s |
| Notify | PR comment, ADO build tag, Teams notification | ~10s |

---

## Prerequisites

Before queueing the pipeline:

1. **`go.mod` exists** at the repository root (or at `buildContext` root if overridden). This is a hard pipeline assertion — Stage 2 fails if absent.

2. **Dockerfile exists** at the repository root. Stage 1 validates the path and fails if the file is not committed.

3. **Platform engineering has provisioned** your tenant:
   - ADO service connection scoped to `payments/*` in the shared ACR
   - `platform-tool-versions` variable group accessible to your ADO project
   - `COSIGN_AKV_SERVICE_CONNECTION` and `COSIGN_KEY_VAULT_NAME` pipeline variables set

4. **(Optional) `VERSION` file** at the repository root for version tagging. See [Version Tagging](#version-tagging) below.

---

## Customizing the Pipeline File

### Change the tenant and app name

```yaml
parameters:
  tenantName: my-team        # your platform slug
  appName: my-service        # your application name
  runtimeType: go
```

Both names must match `^[a-z0-9][a-z0-9-]*[a-z0-9]$`: lowercase, alphanumeric, interior hyphens only, minimum two characters.

### Non-root build context

If your Go module lives in a subdirectory:

```yaml
parameters:
  tenantName: my-team
  appName: my-service
  runtimeType: go
  buildContext: services/api    # go.mod must be at services/api/go.mod
  dockerfilePath: Dockerfile    # resolved as services/api/Dockerfile
```

### Run a dry run on PRs

To validate lint and build without touching ACR, set `dryRun: true` in a branch-specific pipeline or override:

```yaml
parameters:
  tenantName: my-team
  appName: my-service
  runtimeType: go
  dryRun: true
```

Sign & Attest and Publish stages are skipped. Notify still runs with a "Dry Run" PR comment.

---

## Customizing the Dockerfile

### Changing the Go version

Update the `FROM golang:...` line in the build stage:

```dockerfile
FROM golang:1.23-alpine AS build
```

Hadolint will warn on `:latest`. Pin to a specific minor version.

### Changing the entry point

If your `main` package is at the module root rather than `./cmd/server`:

```dockerfile
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-s -w -X main.version=${GIT_COMMIT_SHA}" \
    -o /out/app \
    .
```

### Requiring glibc (CGO or SQLite)

Replace the final stage base image:

```dockerfile
FROM gcr.io/distroless/base-debian12:nonroot AS final
```

Remove `CGO_ENABLED=0` from the build command and ensure the build stage uses a non-Alpine Go image (Alpine uses musl, not glibc):

```dockerfile
FROM golang:1.22 AS build   # Debian-based; has glibc
```

### Multiple binaries

Build each binary separately:

```dockerfile
RUN CGO_ENABLED=0 GOOS=linux go build -o /out/server ./cmd/server
RUN CGO_ENABLED=0 GOOS=linux go build -o /out/worker ./cmd/worker
```

Then copy both:

```dockerfile
COPY --from=build /out/server /server
COPY --from=build /out/worker /worker
```

Only one `ENTRYPOINT` can be defined. Use a supervisor or produce two images (two pipeline runs with different `appName` values) for independent deployables.

### Embedding static files

If your application serves embedded static files (`embed.FS`):

```dockerfile
# The COPY . . already copies them; embed reads from the source tree at compile time.
# No special Dockerfile changes are needed.
```

---

## Version Tagging

Create a `VERSION` file at the repository root (or `buildContext` root):

```
1.4.2
```

The pipeline reads the first non-empty line, trims whitespace, and uses it as the version tag.

| Branch | Tags pushed |
|---|---|
| Any branch, no `VERSION` file | `<40-char-sha>`, `<branch>-<12-char-sha>` |
| Feature branch with `VERSION` | `<40-char-sha>`, `<branch>-<12-char-sha>`, `1.4.2-<12-char-sha>` |
| `main` branch, tag `1.4.2` does not exist in ACR | `<40-char-sha>`, `main-<12-char-sha>`, `1.4.2` |
| `main` branch, tag `1.4.2` already exists in ACR | **Pipeline fails** — bump the version before merging |

To use the version in your binary:

```go
// main.go
var version = "dev"  // overridden by -ldflags at build time

func main() {
    log.Printf("starting payment-processor version=%s", version)
    // ...
}
```

The `-X main.version=${GIT_COMMIT_SHA}` ldflags arg in the Dockerfile injects the Git SHA, not the `VERSION` file value. If you want the version string from `VERSION` embedded in the binary, pass it as a separate build arg:

```dockerfile
ARG APP_VERSION=dev
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-s -w -X main.version=${APP_VERSION}" \
    -o /out/app \
    ./cmd/server
```

Then pass it from the pipeline by committing a `VERSION` file and using the value in your `Makefile` or `docker build` invocation. The platform pipeline does not automatically pass `VERSION` file contents as a build arg — it passes `GIT_COMMIT_SHA` only.

---

## What the Pipeline Produces

For a successful non-dry-run run on `main` with `VERSION=1.4.2`:

### ACR tags

All at `<platform-acr>/payments/payment-processor`:

```
sha256:a3f9e21b7c4d...  ← primary (immutable; use this in Kustomize)
a3f9e21b7c4d...         ← 40-char SHA tag
main-a3f9e21b7c4d       ← alias tag (12-char SHA)
1.4.2                   ← version tag (main branch only)
```

### Cosign objects in ACR

```
<image>:<sha256digest>.sig   ← detached signature
<image>:<sha256digest>.att   ← SBOM as OCI attestation
```

### ADO pipeline artifacts

| Artifact | Contents |
|---|---|
| `sbom-payments-payment-processor` | `sbom.cdx.json` — CycloneDX SBOM of every package in the image |
| `provenance-payments-payment-processor` | `provenance.json` — digest, tags, SBOM name, git commit, cosign status |

### PR comment

If the pipeline was triggered by a PR, Stage 5 posts a Markdown table to the PR thread:

```
✅ Container Build — Succeeded

| Field           | Value                                              |
|-----------------|----------------------------------------------------|
| Tenant / App    | payments/payment-processor                         |
| Runtime         | go                                                 |
| Status          | ✅ Succeeded                                       |
| Image Ref       | <acr>/payments/payment-processor@sha256:a3f9...   |
| Manifest Digest | sha256:a3f9e21b...                                 |
| Cosign Status   | signed ✅                                          |
| SBOM Artifact   | sbom-payments-payment-processor                    |
```

### ADO build tag

The pipeline run is tagged with the manifest digest (`sha256:a3f9e21b...`) in the ADO pipeline run metadata, linking the run to the exact image pushed.

---

## Key Build Log Messages

These messages in the pipeline log confirm that key steps succeeded:

| Stage | Log message | Meaning |
|---|---|---|
| Setup | `Tool versions resolved: Docker/BuildKit=...` | Variable group read successfully |
| Setup | `All parameters valid. tenantName=payments, appName=payment-processor, runtimeType=go` | Validation passed |
| Build | `Dockerfile lint passed.` | No Hadolint errors |
| Build | `Building image: payments/payment-processor:pipeline-12345` | BuildKit started |
| Build | `IMAGE_DIGEST=sha256:...` | Build completed; config digest captured |
| Build | `GO_VERSION resolved to: 1.4.2` | VERSION file read (or "version tag will be skipped") |
| Sign & Attest | `SBOM generated at sbom.cdx.json` | Syft completed |
| Sign & Attest | `Cosign signature verified successfully.` | Sign + verify round-trip passed |
| Sign & Attest | `MANIFEST_DIGEST=sha256:...` | Manifest digest captured from ACR |
| Publish | `latest-tag assertion passed` | No `latest` tag in candidate set |
| Publish | `Digest verified: sha256:...` | Digest integrity confirmed after full-SHA push |
| Notify | `PR comment posted` | PR thread updated |
| Notify | `ADO build tag set to: sha256:...` | Pipeline run tagged |

---

## Troubleshooting

**`go.mod not found` — Stage 2 fails**
The pipeline checks for `go.mod` at `$BUILD_SOURCESDIRECTORY/<buildContext>/go.mod`. Ensure the file is committed and the `buildContext` parameter (if overridden) points to the directory containing it.

**`Dockerfile lint passed` but build fails**
Hadolint only validates Dockerfile syntax and best practices, not whether the build actually succeeds. Check the Docker build log for the specific error — common causes are network failures during `go mod download` or a wrong path in `COPY` or `ENTRYPOINT`.

**`Version tag 'X' already exists in ACR`**
Update the `VERSION` file to a new version string before merging to `main`. Once merged and tagged, that version string is permanently occupied in ACR.

**Binary exits immediately in AKS**
Verify the binary path in `ENTRYPOINT` matches the `-o` output path in the `go build` command. Distroless images have no shell — a wrong path produces a silent exit code 127 with no useful message. Test locally with `docker run --rm <image>`.
