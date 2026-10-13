# ARG vs ENV

## Introduction

`ARG` and `ENV` both define variables in a Dockerfile, but they operate in completely different scopes and have very different security implications. Using the wrong one for secrets is one of the most common security mistakes in container images — and it can expose credentials permanently in your image layer history.

This file explains precisely what each instruction does, when to use each, and how they interact during the build process.

---

## The Core Distinction

| | `ARG` | `ENV` |
|---|---|---|
| **Available during** | Build time only | Build time AND runtime |
| **Persists in the image** | No (not in layers) | Yes (embedded in image metadata) |
| **Available to containers** | No | Yes |
| **Overridable at runtime** | No | Yes (via `docker run -e`) |
| **Overridable at build time** | Yes (`--build-arg`) | Yes (but in-Dockerfile only) |
| **Visible in `docker inspect`** | No (after build) | Yes |
| **Safe for secrets** | Not quite (see below) | No |

---

## ARG — Build-Time Variables

`ARG` defines a variable that exists only during the build process. It is used to parameterize builds — allowing you to pass different values for things like version numbers, registry URLs, or feature flags without modifying the Dockerfile.

```dockerfile
FROM alpine

# Declare the ARG
ARG APP_VERSION=1.0.0

# Use it during build
RUN echo "Building version ${APP_VERSION}"

# ARG is NOT available after the image is built
# Running containers cannot access APP_VERSION
```

### Passing ARG Values at Build Time

```bash
# Use the default value (1.0.0)
docker build -t myapp .

# Override the value
docker build --build-arg APP_VERSION=2.3.1 -t myapp:2.3.1 .

# Multiple build args
docker build \
  --build-arg APP_VERSION=2.3.1 \
  --build-arg GIT_COMMIT=a1b2c3d4 \
  --build-arg BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
  -t myapp:2.3.1 .
```

### Default Values

```dockerfile
# With a default — build succeeds even without --build-arg
ARG APP_VERSION=latest
ARG PYTHON_VERSION=3.11

# Required (no default) — build FAILS if not provided
ARG ECR_REGISTRY
```

---

## ENV — Runtime Environment Variables

`ENV` sets an environment variable that is baked into the image metadata and is available both during the build and at container runtime.

```dockerfile
FROM python:3.11-slim

# Set runtime configuration
ENV APP_ENV=production \
    PORT=8080 \
    LOG_FORMAT=json \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

COPY . /app
WORKDIR /app
```

When a container starts from this image, `APP_ENV`, `PORT`, `LOG_FORMAT`, `PYTHONUNBUFFERED`, and `PYTHONDONTWRITEBYTECODE` are all set automatically.

```bash
# Inspect the image to see baked-in ENV values
docker inspect myapp:1.0 --format '{{json .Config.Env}}'
# ["APP_ENV=production","PORT=8080","LOG_FORMAT=json","PYTHONUNBUFFERED=1","PYTHONDONTWRITEBYTECODE=1"]

# These can be overridden at runtime
docker run -e APP_ENV=staging -e PORT=9090 myapp:1.0
```

---

## What to Use for What

### Use ARG for: Build Configuration

Version numbers, registry URLs, build targets, feature flags, compiler options — anything that parameterizes the build but is not needed at runtime.

```dockerfile
ARG APP_VERSION=1.0.0
ARG GIT_COMMIT=unknown
ARG ECR_REGISTRY

# Use build args to tag the image or embed version metadata
LABEL org.opencontainers.image.version="${APP_VERSION}"
LABEL org.opencontainers.image.revision="${GIT_COMMIT}"

# Pull from a registry using the ARG
FROM ${ECR_REGISTRY}/base-image:latest AS base
```

### Use ENV for: Runtime Configuration Defaults

Port numbers, language settings, feature flags with sensible defaults — anything the running container needs.

```dockerfile
# Python best practices
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

# Node.js best practices
ENV NODE_ENV=production \
    NODE_OPTIONS="--max-old-space-size=512"

# Application defaults (overridable via docker run -e)
ENV PORT=8080 \
    LOG_LEVEL=info \
    METRICS_ENABLED=true
```

### Use Neither for: Secrets

This is the critical rule. **Do not use `ENV` or `ARG` for passwords, API keys, or credentials.**

```dockerfile
# WRONG — password baked into the image permanently
ENV DATABASE_PASSWORD=supersecret

# WRONG — password visible in docker history
ARG DATABASE_PASSWORD
ENV DATABASE_PASSWORD=${DATABASE_PASSWORD}

# WRONG — password in a layer, visible in docker history
RUN export DATABASE_PASSWORD=supersecret && pip install -r requirements.txt
```

Secrets belong in runtime injection (environment variables from Secrets Manager) or BuildKit secrets. Never in the Dockerfile itself.

---

## ARG and Layer Cache Interactions

`ARG` values affect the build cache. If an `ARG` value changes, the cache is invalidated from that point forward.

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Installing dependencies first (before the ARG that changes often)
COPY requirements.txt .
RUN pip install -r requirements.txt   # cached as long as requirements.txt doesn't change

# Declare the frequently changing ARG AFTER the expensive cached steps
ARG APP_VERSION=latest
ARG GIT_COMMIT=unknown

# Apply version metadata
LABEL org.opencontainers.image.version="${APP_VERSION}"
LABEL org.opencontainers.image.revision="${GIT_COMMIT}"

COPY . .
```

If `APP_VERSION` were declared before the `pip install`, every build with a new version tag would invalidate the pip cache and re-download all dependencies. Keep `ARG` declarations as late as possible to maximize cache reuse.

---

## Converting ARG to ENV

You can promote a build-time `ARG` to a runtime `ENV` — making it available in both phases. This is the pattern for embedding version information in an image so that the running container knows its own version:

```dockerfile
ARG APP_VERSION=1.0.0
ARG GIT_COMMIT=unknown

# Promote to ENV — now available in running containers
ENV APP_VERSION=${APP_VERSION} \
    GIT_COMMIT=${GIT_COMMIT}
```

```bash
docker build --build-arg APP_VERSION=2.3.1 --build-arg GIT_COMMIT=a1b2c3d4 -t myapp .

docker run myapp env | grep VERSION
# APP_VERSION=2.3.1
# GIT_COMMIT=a1b2c3d4
```

This is useful for building version endpoints into your application:

```python
import os

@app.route("/version")
def version():
    return {
        "version": os.getenv("APP_VERSION", "unknown"),
        "commit": os.getenv("GIT_COMMIT", "unknown")
    }
```

---

## ARG Before and After FROM

`ARG` placed before `FROM` is special — it only exists during the `FROM` instruction. It cannot be used in the rest of the Dockerfile without re-declaring it.

```dockerfile
# This ARG is available only during the FROM instruction
ARG BASE_IMAGE=python:3.11-slim

FROM ${BASE_IMAGE}

# Must re-declare ARG after FROM to use it in the rest of the build
ARG APP_VERSION=1.0.0

RUN echo "Building version ${APP_VERSION} on ${BASE_IMAGE}"
```

This lets you parameterize the base image itself — useful for testing your image against different base image versions.

```bash
docker build --build-arg BASE_IMAGE=python:3.12-slim -t myapp:py312 .
docker build --build-arg BASE_IMAGE=python:3.11-slim -t myapp:py311 .
```

---

## ARG and ENV in Docker Compose

```yaml
# docker-compose.yml
services:
  api:
    build:
      context: .
      args:
        APP_VERSION: ${APP_VERSION:-latest}   # passed to Dockerfile as ARG
        GIT_COMMIT: ${GIT_SHA:-unknown}
    environment:
      PORT: "8080"                             # set as ENV at runtime
      LOG_LEVEL: ${LOG_LEVEL:-info}
```

The `build.args` values are passed to the Dockerfile as `--build-arg` during `docker compose build`. The `environment` values are passed as `-e` when the container starts.

---

## Visibility and Security

```bash
# ARG values are NOT visible in docker inspect after the build completes
docker inspect myapp:1.0 --format '{{json .Config}}'
# You will NOT see ARG values here

# ENV values ARE visible in docker inspect
docker inspect myapp:1.0 --format '{{json .Config.Env}}'
# Shows all ENV values baked into the image

# docker history shows the commands used to build each layer
docker history myapp:1.0
# If you used ENV with a secret: "ENV DATABASE_PASSWORD=secret" is visible here
# If you used ARG → ENV with a secret: the ARG value is visible in history
```

This is why even `ARG` is not safe for secrets. The `ARG` value is passed via `--build-arg` and is captured in `docker history`. On ECR and Docker Hub, image history is often accessible to anyone who can pull the image.

BuildKit secrets (`RUN --mount=type=secret`) are the correct solution for build-time credentials — they are never stored in any layer.

---

## Practical Example: Full Parameterized Dockerfile

```dockerfile
# Arguments for parameterizing the build
ARG BASE_IMAGE=python:3.11-slim
FROM ${BASE_IMAGE}

# Re-declare after FROM to use in the rest of the build
ARG APP_VERSION=1.0.0
ARG GIT_COMMIT=unknown
ARG BUILD_DATE

# Runtime environment defaults
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PORT=8080 \
    LOG_LEVEL=info

# Promote build info to runtime ENV (optional — for /version endpoints)
ENV APP_VERSION=${APP_VERSION} \
    GIT_COMMIT=${GIT_COMMIT}

# Build metadata labels (uses both ARG and the build date)
LABEL org.opencontainers.image.version="${APP_VERSION}" \
      org.opencontainers.image.revision="${GIT_COMMIT}" \
      org.opencontainers.image.created="${BUILD_DATE}" \
      org.opencontainers.image.source="https://github.com/myorg/myapp"

WORKDIR /app

# Dependencies first — cached unless requirements.txt changes
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY src/ .

USER 1001
EXPOSE 8080
CMD ["gunicorn", "--workers", "4", "--bind", "0.0.0.0:8080", "app:app"]
```

---

## Summary

- `ARG` exists only at build time — not available in running containers, not baked into image metadata
- `ENV` exists at build time and runtime — baked into image metadata, available in every container started from the image
- Use `ARG` for: base image selection, version tags, registry URLs, build configuration — things the build process needs but containers do not
- Use `ENV` for: runtime defaults (port, log level, language flags) — values containers need but operators may want to override with `-e`
- Place `ARG` declarations as late as possible in the Dockerfile — an `ARG` change invalidates the layer cache from that point, so expensive steps before the ARG remain cached
- Never put secrets in `ARG` or `ENV` — `docker history` reveals `ARG` values and `docker inspect` reveals `ENV` values; use BuildKit secrets or runtime injection from Secrets Manager

In the next file, we cover the `USER` instruction — running containers as non-root for security hardening.
