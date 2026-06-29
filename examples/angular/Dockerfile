# syntax=docker/dockerfile:1
# ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
# REQUIRED: This first line enables BuildKit syntax extensions.
# The '--mount=type=secret' syntax used below is a BuildKit feature and
# requires this directive. The platform pipeline always builds with
# DOCKER_BUILDKIT=1, so this directive activates the correct frontend.

# ============================================================================
# STAGE 1: build
#
# Installs npm dependencies using the platform Azure Artifacts feed and
# compiles the Angular application. All npm operations run inside the
# container — no Node.js is installed on the pipeline agent.
#
# Two values are injected by the platform into every docker build:
#   --build-arg NPM_REGISTRY=<azure-artifacts-npm-feed-url>
#   --secret id=npm_token,env=SYSTEM_ACCESSTOKEN
#
# SECURITY: The auth token is NEVER written to an image layer. The
# --mount=type=secret syntax mounts it as an in-memory file/env at build
# time only. Docker layer inspection (docker history, docker inspect) will
# not reveal the token.
# ============================================================================
FROM node:22-alpine AS build

WORKDIR /app

# Receive the platform Azure Artifacts npm registry URL.
# Using ARG (not ENV) means this value is not baked into the image.
ARG NPM_REGISTRY

# ── npm credential configuration ─────────────────────────────────────────────
# Configure npm to use the platform Azure Artifacts feed as the registry and
# set the bearer token for authentication.
#
# The token is read from the BuildKit secret 'npm_token', which the platform
# populates from the ADO SYSTEM_ACCESSTOKEN (a scoped PAT with Packaging:Read
# permission). The RUN --mount form is the ONLY safe way to use secrets in
# Dockerfiles — alternatives (ARG, ENV, COPY) all persist the secret in a layer.
#
# The auth config key format is: //<registry-host-without-https>:_authToken
# npm strips the https:// prefix when constructing the per-registry key.
RUN --mount=type=secret,id=npm_token,env=NPM_TOKEN \
    npm config set registry "${NPM_REGISTRY}" && \
    npm config set "//${NPM_REGISTRY#https://}:_authToken" "${NPM_TOKEN}"

# ── Dependency caching ───────────────────────────────────────────────────────
# Copy package manifests before source code. The npm ci step is cached
# unless package.json or package-lock.json changes. This is the most
# impactful layer cache optimization for npm-based builds.
#
# 'npm ci' (clean install):
# - Installs exactly what package-lock.json specifies
# - Never modifies package-lock.json
# - Removes node_modules before installing (reproducible)
# - Faster than 'npm install' in CI (does not resolve version ranges)
#
# '--prefer-offline': uses the local npm cache if available. Reduces
# bandwidth and speeds up builds after the first run on an agent.
COPY package.json package-lock.json ./
RUN npm ci --prefer-offline

# ── Source copy ──────────────────────────────────────────────────────────────
COPY . .

# GIT_COMMIT_SHA: injected by the platform pipeline as the full 40-char Git SHA.
# Make it available to the Angular build via an environment variable.
# To embed it in your app, reference it in environment.ts or pass it to
# Angular's --define flag in angular.json.
ARG GIT_COMMIT_SHA=dev
ENV GIT_COMMIT_SHA=${GIT_COMMIT_SHA}

# ── Angular build ────────────────────────────────────────────────────────────
# 'npm run build' invokes the 'build' script from package.json, which
# typically calls 'ng build' or 'ng build --configuration production'.
#
# '--output-path=dist/app': write the compiled output to dist/app so the
# COPY in the final stage has a stable path regardless of the project name.
# Adjust if your project uses a different output directory.
#
# The Angular CLI reads configuration from angular.json. Production builds
# typically enable: AOT compilation, tree shaking, minification, and
# file hashing for cache busting.
RUN npm run build -- --output-path=dist/app

# ============================================================================
# STAGE 2: final
#
# Minimal nginx image serving the pre-compiled static assets.
# There is no Node.js runtime in this image — Angular compiles to static
# HTML/JS/CSS. The only process running is nginx.
#
# Using nginx:1.27-alpine:
# - Alpine base keeps the image small (< 25MB)
# - Pin to a specific minor version; avoid :latest (Hadolint DL3007)
# ============================================================================
FROM nginx:1.27-alpine AS final

# Copy the compiled Angular output from the build stage.
# The destination /usr/share/nginx/html is nginx's default document root.
COPY --from=build /app/dist/app /usr/share/nginx/html

# Replace the default nginx configuration with one that supports Angular's
# HTML5 routing (pushState). Without this, direct URL navigation and page
# refreshes return 404 from nginx.
# See ./nginx.conf for the full configuration.
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
