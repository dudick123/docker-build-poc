# Example: Python

A complete, annotated example of the platform container build pipeline for a Python application (FastAPI/uvicorn). All files in this directory can be copied into an application repository root and customized.

---

## Files in This Example

| File | Purpose |
|---|---|
| `azure-pipelines.yml` | ADO pipeline definition |
| `Dockerfile` | Multi-stage Dockerfile using pip + requirements.txt |
| `README.md` | This document |

---

## How It Works

### Pipeline stages

```
Setup → Build → Sign & Attest → Publish → Notify
```

The Python runtime validator (`steps/runtime/python.yml`) runs after the Docker build as the last step in Stage 2. It performs two advisory checks and emits the version string:

1. Checks for a recognized dependency file (`requirements.txt`, `pyproject.toml`, `poetry.lock`)
2. Extracts `version` from `pyproject.toml` or `setup.cfg`

Neither check is a hard failure — the build continues regardless.

---

## Prerequisites

1. **Dockerfile** at the repository root (or at `buildContext` if overridden). Stage 1 validates the path.
2. **`requirements.txt`** (or `pyproject.toml` / `poetry.lock`) in the build context — recommended to avoid the advisory warning.
3. **Platform engineering** has provisioned your tenant service connection and variable group access.

---

## Version Tagging

The pipeline extracts the version from project metadata files. Add a static `version` to your `pyproject.toml`:

```toml
[project]
name = "ingest-api"
version = "3.1.0"
```

Or for Poetry:

```toml
[tool.poetry]
name = "ingest-api"
version = "3.1.0"
```

Or in `setup.cfg`:

```ini
[metadata]
version = 3.1.0
```

| Branch | Tags pushed to ACR |
|---|---|
| Any branch, no static version | `<40-char-sha>`, `<branch>-<12-char-sha>` |
| Feature branch with version | `<40-char-sha>`, `<branch>-<12-char-sha>`, `3.1.0-<12-char-sha>` |
| `main`, tag `3.1.0` is new | `<40-char-sha>`, `main-<12-char-sha>`, `3.1.0` |
| `main`, tag `3.1.0` already exists | **Pipeline fails** — bump the version before merging |

**Dynamic versions are not supported.** Fields like `version = {attr: mypackage.__version__}` are not matched. Use a static string or omit the version (tagging will be skipped).

---

## Customizing the Dockerfile

### Switching to Poetry

Replace the pip-based build stage with a Poetry-based one:

```dockerfile
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
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Switching to Django / Gunicorn

Change the `CMD`:

```dockerfile
CMD ["python", "-m", "gunicorn", "myproject.wsgi:application", \
     "--bind", "0.0.0.0:8000", "--workers", "2"]
```

### Applications that need write access at runtime

If your app writes to disk (uploads, temporary files, SQLite), the `nobody` user will encounter permission errors on paths it doesn't own. Create a dedicated user instead:

```dockerfile
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
RUN chown -R appuser:appgroup /app
USER appuser
```

### No native extensions

If `requirements.txt` contains only pure-Python packages, omit `build-essential` and `libpq-dev` from the build stage and `libpq5` from the final stage:

```dockerfile
FROM python:3.12-slim AS build
WORKDIR /app
COPY requirements.txt ./
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt
COPY . .

FROM python:3.12-slim AS final
ARG GIT_COMMIT_SHA=dev
WORKDIR /app
COPY --from=build /install /usr/local
COPY --from=build /app .
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1 GIT_COMMIT_SHA=${GIT_COMMIT_SHA}
EXPOSE 8000
USER nobody
CMD ["python", "-m", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

This produces a smaller image and has a smaller attack surface.

### Accessing `GIT_COMMIT_SHA` at runtime

The build arg is exposed as an environment variable in the final image. Read it in your application:

```python
import os

VERSION = os.environ.get("GIT_COMMIT_SHA", "dev")

@app.get("/healthz")
def health():
    return {"status": "ok", "version": VERSION}
```

---

## Hadolint and Python

Common Hadolint rules relevant to Python Dockerfiles:

| Rule | Issue | Fix |
|---|---|---|
| `DL3013` | `pip install` without `--no-cache-dir` | Add `--no-cache-dir` to all `pip install` commands |
| `DL3042` | `pip install` runs as root | Use `--user` or switch to a virtualenv approach |
| `DL3008` | `apt-get install` without version pinning | Pin versions or add to `.hadolint.yaml` ignore list |

To suppress a project-wide rule, create `.hadolint.yaml` in the build context:

```yaml
ignore:
  - DL3008
```

---

## What the Pipeline Produces

For a successful non-dry-run run on `main` with `version = "3.1.0"` in `pyproject.toml`:

### ACR tags (at `<platform-acr>/data-platform/ingest-api`)

| Tag | Use |
|---|---|
| `<40-char-sha>` | Kustomize manifests (immutable) |
| `main-<12-char-sha>` | Human navigation |
| `3.1.0` | Release version tag |

### Pipeline artifacts

| Artifact | Contents |
|---|---|
| `sbom-data-platform-ingest-api` | CycloneDX JSON SBOM |
| `provenance-data-platform-ingest-api` | JSON provenance record |

---

## Troubleshooting

**`No recognized Python dependency file found` warning**
The pipeline did not find `requirements.txt`, `pyproject.toml`, or `poetry.lock`. This is advisory only — the build continues. Add one of these files to suppress the warning.

**`pip install` fails for a native package inside Docker**
Ensure `build-essential` and any required C library dev packages are installed in the build stage. Common ones: `libpq-dev` (psycopg2), `libssl-dev` (cryptography), `gcc` (general C extension packages).

**`Permission denied` at runtime in AKS**
The `nobody` user does not own any files in `/app`. If the application creates files at startup, switch to a dedicated user with `chown -R` before `USER`. Ports above 1024 do not require root.

**`ModuleNotFoundError` at runtime**
The `COPY --from=build /install /usr/local` step copies pip-installed packages. If a package installs into a Python version-specific path (e.g., `/install/lib/python3.12/site-packages/`), verify the final stage uses the same Python version as the build stage.

**Version not extracted despite `pyproject.toml` present**
The pipeline extracts static version strings only. Dynamic fields (`version = {attr: ...}`, `version = {file: VERSION}`) are not matched. Use a literal string like `version = "3.1.0"`.
