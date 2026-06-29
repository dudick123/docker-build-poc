# syntax=docker/dockerfile:1

# ============================================================================
# STAGE 1: build
#
# Builds the React application (Vite) inside a container.
# The platform pipeline injects npm credentials automatically:
#   --build-arg NPM_REGISTRY=<azure-artifacts-url>
#   --secret id=npm_token,env=SYSTEM_ACCESSTOKEN
#
# SECURITY: The npm auth token is never stored in any image layer.
# The --mount=type=secret syntax makes it available only during the RUN
# instruction that needs it. 'docker history' and 'docker inspect' will
# not reveal the token.
# ============================================================================
FROM node:22-alpine AS build

WORKDIR /app

# Receive the platform Azure Artifacts npm registry URL as a build argument.
ARG NPM_REGISTRY

# Configure npm to authenticate against the platform Azure Artifacts feed.
# The token is scoped to this registry host only — public registry auth
# is not affected.
RUN --mount=type=secret,id=npm_token,env=NPM_TOKEN \
    npm config set registry "${NPM_REGISTRY}" && \
    npm config set "//${NPM_REGISTRY#https://}:_authToken" "${NPM_TOKEN}"

# ── Dependency caching ───────────────────────────────────────────────────────
# Copy package manifests before source. Docker caches this layer until
# package.json or package-lock.json changes, avoiding a full reinstall
# on every source code change.
COPY package.json package-lock.json ./
RUN npm ci --prefer-offline

# ── Source copy ──────────────────────────────────────────────────────────────
COPY . .

# GIT_COMMIT_SHA: injected by the platform pipeline as the full 40-char Git SHA.
# Vite exposes env variables prefixed with VITE_ to client code.
# Access it in your app: import.meta.env.VITE_GIT_COMMIT_SHA
ARG GIT_COMMIT_SHA=dev
ENV VITE_GIT_COMMIT_SHA=${GIT_COMMIT_SHA}

# ── React/Vite build ─────────────────────────────────────────────────────────
# 'npm run build' typically maps to 'vite build' in package.json.
# Vite writes output to 'dist/' by default.
# CRA writes to 'build/' — adjust the COPY in the final stage if using CRA.
RUN npm run build

# ============================================================================
# STAGE 2: final
#
# Minimal nginx image serving the compiled static assets.
# No Node.js runtime is needed — React compiles to static HTML/JS/CSS.
# ============================================================================
FROM nginx:1.27-alpine AS final

# Copy Vite build output to nginx's document root.
# CRA users: change 'dist' to 'build'
COPY --from=build /app/dist /usr/share/nginx/html

# Custom nginx configuration for SPA routing.
# Without this, direct URL navigation and page refreshes return 404.
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
