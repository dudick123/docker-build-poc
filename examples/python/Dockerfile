# syntax=docker/dockerfile:1

# ============================================================================
# STAGE 1: build
#
# Installs all dependencies (including build-time tools like gcc) and
# packages them for copying into the final image. Keeping build tools
# out of the final image reduces attack surface and image size.
#
# This example uses pip + requirements.txt. See the README for a Poetry
# variant.
# ============================================================================
FROM python:3.12-slim AS build

WORKDIR /app

# Install C build tools needed by some Python packages (e.g., cryptography,
# numpy, psycopg2). These are only needed at build time — they will NOT be
# in the final image.
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# ── Dependency caching ───────────────────────────────────────────────────────
# Install dependencies before copying source. The '--prefix=/install' flag
# places all packages in /install instead of the system site-packages,
# making it easy to COPY this directory to the final image.
# '--no-cache-dir' avoids storing the pip download cache in the layer.
COPY requirements.txt ./
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# ── Source copy ──────────────────────────────────────────────────────────────
COPY . .

# ============================================================================
# STAGE 2: final
#
# Minimal runtime image. Copies the installed packages and application
# source from the build stage. Does not include gcc, libpq-dev, or pip.
#
# Using the same python:3.12-slim base ensures ABI compatibility between
# the build and runtime environments.
# ============================================================================
FROM python:3.12-slim AS final

# Only install runtime C libraries needed by installed Python packages
# (e.g., libpq for psycopg2). Build tools are not included.
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# GIT_COMMIT_SHA: injected by the platform pipeline as the full 40-char Git SHA.
# The default 'dev' allows local builds without specifying --build-arg.
ARG GIT_COMMIT_SHA=dev

# Copy the installed packages from the build stage into the standard
# Python library path.
COPY --from=build /install /usr/local

# Copy the application source code.
COPY --from=build /app .

# ── Python runtime optimizations ─────────────────────────────────────────────
# PYTHONDONTWRITEBYTECODE: prevents writing .pyc files (not needed in containers)
# PYTHONUNBUFFERED: ensures stdout/stderr are not buffered — required for ADO
#   log streaming and for Kubernetes log collection to work correctly
# GIT_COMMIT_SHA: available at runtime via os.environ for version reporting
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    GIT_COMMIT_SHA=${GIT_COMMIT_SHA}

EXPOSE 8000

# Run as a non-root user. 'nobody' is a universally available unprivileged
# user in Debian-based images. For write access to specific directories,
# create a dedicated user and chown before this directive.
USER nobody

# Start the application using uvicorn (FastAPI/Starlette).
# Adjust the module path to match your application's entry point.
# For Django: python -m gunicorn myapp.wsgi:application --bind 0.0.0.0:8000
CMD ["python", "-m", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
