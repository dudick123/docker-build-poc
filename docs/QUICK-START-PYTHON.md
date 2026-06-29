# Quick Start: Python

This guide covers what the pipeline requires from a Python application repository. Read [QUICK-START.md](QUICK-START.md) first for prerequisites and the base `azure-pipelines.yml` structure.

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
    runtimeType: python
```

---

## Build Pattern

Python uses a self-contained multi-stage Dockerfile. The pipeline agent runs only `docker build` — no Python toolchain is installed on the agent. All dependency installation, compilation, and packaging happens inside the Dockerfile.

---

## Repository Requirements

### Dependency file (recommended)

The pipeline performs an advisory check for a recognized Python dependency file in the build context. If none is found, a warning is written to the build log but the build is not blocked.

Recognized files (any one is sufficient):

| File | Tool |
|---|---|
| `requirements.txt` | pip |
| `pyproject.toml` | Poetry, PDM, Hatch, setuptools |
| `poetry.lock` | Poetry |

This is a soft check — it tells you that the pipeline recognizes your project layout, not that dependencies will install correctly.

### Version extraction

The pipeline extracts a version string for optional ACR version tags. It checks in this order:

1. `pyproject.toml` — looks for `version = "x.y.z"` under `[project]` or `[tool.poetry]`
2. `setup.cfg` — looks for `version = x.y.z` under `[metadata]`

If neither file exists or neither contains a static version string, the version tag is skipped — the image is still published with the full SHA and alias tags. Dynamic version schemes (e.g., `version = {attr: mypackage.__version__}`) are not supported; set a static version string or omit the version entirely.

---

## Reference Dockerfile

### Standard (pip + requirements.txt)

```dockerfile
# syntax=docker/dockerfile:1

# ── build stage ──────────────────────────────────────────────────────────────
FROM python:3.12-slim AS build

WORKDIR /app

# Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt ./
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

COPY . .

# ── final stage ──────────────────────────────────────────────────────────────
FROM python:3.12-slim AS final

ARG GIT_COMMIT_SHA=dev

WORKDIR /app

COPY --from=build /install /usr/local
COPY --from=build /app .

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    GIT_COMMIT_SHA=${GIT_COMMIT_SHA}

EXPOSE 8000

USER nobody

CMD ["python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Poetry

```dockerfile
# syntax=docker/dockerfile:1

FROM python:3.12-slim AS build

ENV POETRY_VERSION=1.8.3 \
    POETRY_VIRTUALENVS_IN_PROJECT=true \
    POETRY_NO_INTERACTION=1

RUN pip install --no-cache-dir poetry==$POETRY_VERSION

WORKDIR /app

COPY pyproject.toml poetry.lock ./
RUN poetry install --only=main --no-root

COPY . .
RUN poetry install --only=main

# ── final stage ──────────────────────────────────────────────────────────────
FROM python:3.12-slim AS final

ARG GIT_COMMIT_SHA=dev

WORKDIR /app

COPY --from=build /app/.venv ./.venv
COPY --from=build /app .

ENV PATH="/app/.venv/bin:$PATH" \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    GIT_COMMIT_SHA=${GIT_COMMIT_SHA}

EXPOSE 8000

USER nobody

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Key points

- **`GIT_COMMIT_SHA` build arg** — injected by the pipeline as the full 40-character Git SHA. The `ARG` default (`dev`) keeps local builds working without the pipeline.
- **Non-root user** — use `USER nobody` or create a dedicated user. Running as root in the final image is a Hadolint warning.
- **`PYTHONDONTWRITEBYTECODE=1`** — prevents `.pyc` files from being written, reducing image size.
- **`PYTHONUNBUFFERED=1`** — ensures stdout/stderr is not buffered, which is required for ADO log streaming.
- **Build stage separation** — keep build tools (`build-essential`, Poetry, pip) in the build stage only. The final stage should contain only runtime dependencies.

---

## What the Pipeline Validates

The Python runtime step (`steps/runtime/python.yml`) runs after the Docker build and performs:

1. Checks for the presence of `requirements.txt`, `pyproject.toml`, or `poetry.lock` — advisory warning if none found
2. Extracts a version string from `pyproject.toml` or `setup.cfg` — emits as `PYTHON_VERSION` for the Publish stage tag logic
3. Logs a notice if no static version is found (version tag will be skipped)

This step does not run `pip`, `poetry`, or any Python command on the agent.

---

## Version Tagging Summary

| Scenario | Tags pushed to ACR |
|---|---|
| No version in `pyproject.toml` or `setup.cfg` | `<40-char-sha>`, `<branch>-<12-char-sha>` |
| Version found, non-main branch | `<40-char-sha>`, `<branch>-<12-char-sha>`, `<version>-<12-char-sha>` |
| Version found, `main` branch, tag does not exist | `<40-char-sha>`, `main-<12-char-sha>`, `<version>` |
| Version found, `main` branch, tag already exists | Pipeline fails — bump the version before merging |

---

## Common Issues

**Version not extracted despite `pyproject.toml` being present**
The pipeline uses a regex for static version strings (`version = "x.y.z"`). Dynamic version fields (e.g., `version = {attr: mypackage.__version__}` or `version = {file: VERSION}`) are not matched. Use a static string or let the version tag be skipped.

**`No recognized Python dependency file found` warning**
This is advisory only — the build continues. Add a `requirements.txt` or `pyproject.toml` to make the warning go away.

**Package install fails inside Docker build**
If your packages require native compilation (`gcc`, `libpq-dev`, etc.), ensure the build stage base image includes those dependencies (`apt-get install build-essential`). The final stage does not need them.

**`USER nobody` causes permission errors at runtime**
If your application writes to disk or binds to a privileged port, adjust the `USER` directive. Ports above 1024 do not require root. For write access, use `chown` in the Dockerfile before switching users.
