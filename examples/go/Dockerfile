# syntax=docker/dockerfile:1
# ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
# Required first line. Enables BuildKit syntax extensions (--mount=type=secret,
# heredoc support, etc.). The platform pipeline always builds with
# DOCKER_BUILDKIT=1, so this line activates the latest BuildKit frontend.

# ============================================================================
# STAGE 1: build
#
# Compiles the Go binary inside a container. Nothing from this stage appears
# in the final image — only the compiled binary is copied forward.
#
# Using a versioned Alpine-based Go image:
# - Alpine keeps the image small and avoids unnecessary glibc tooling
# - Pin to a specific minor version (not :latest) so builds are reproducible
# - Hadolint will warn on :latest tags (DL3007)
# ============================================================================
FROM golang:1.22-alpine AS build

WORKDIR /src

# ── Dependency caching ───────────────────────────────────────────────────────
# Copy go.mod and go.sum BEFORE the source code. Docker layer caching means
# the 'go mod download' step is only re-executed when these files change, not
# on every source code change. This significantly speeds up repeat builds.
COPY go.mod go.sum ./
RUN go mod download

# ── Source copy ──────────────────────────────────────────────────────────────
# Copy the full source tree after dependencies are cached.
COPY . .

# ── Build argument: GIT_COMMIT_SHA ───────────────────────────────────────────
# The platform pipeline injects the full 40-character Git SHA as this build
# arg on every pipeline run. The ARG default 'dev' makes local docker build
# commands work without specifying --build-arg.
#
# Embed it in the binary via -ldflags to make it visible at runtime
# (e.g., via a /version endpoint or structured log field).
ARG GIT_COMMIT_SHA=dev

# ── Binary compilation ───────────────────────────────────────────────────────
# CGO_ENABLED=0: required for the distroless final image (no libc available)
# GOOS=linux: cross-compile explicitly even if building on Linux (safe habit)
# -ldflags "-s -w": strip debug symbols and DWARF info to reduce binary size
# -X main.version: embed the git SHA as a package-level variable
# -o /out/app: write to a predictable path outside the source tree
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-s -w -X main.version=${GIT_COMMIT_SHA}" \
    -o /out/app \
    ./cmd/server

# ============================================================================
# STAGE 2: final
#
# The production image. Uses distroless rather than scratch or alpine:
# - No shell, no package manager, no unnecessary binaries → smallest attack surface
# - :nonroot variant runs as UID 65532 by default → no root in production
# - static-debian12 has no libc (suitable for CGO_ENABLED=0 binaries)
#
# If your binary requires glibc (e.g., CGO, SQLite), use:
#   gcr.io/distroless/base-debian12:nonroot
# ============================================================================
FROM gcr.io/distroless/static-debian12:nonroot AS final

# Carry the SHA forward so it is visible in 'docker inspect' environment output.
ARG GIT_COMMIT_SHA=dev
ENV GIT_COMMIT_SHA=${GIT_COMMIT_SHA}

# Copy only the compiled binary from the build stage.
# /app is the conventional location in distroless images.
COPY --from=build /out/app /app

# Document the port the application listens on.
# This is metadata only — it does not publish the port. AKS services and
# ingress resources define actual exposure.
EXPOSE 8080

# JSON array form is required for distroless (no shell to invoke string form).
ENTRYPOINT ["/app"]
