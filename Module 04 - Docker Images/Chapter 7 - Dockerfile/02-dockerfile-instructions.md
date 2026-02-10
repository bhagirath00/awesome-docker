# Dockerfile Instructions

## Introduction

Every Dockerfile instruction has specific behavior, edge cases, and best practices. Using instructions incorrectly leads to large images, slow builds, security vulnerabilities, and containers that behave unexpectedly.

This file covers every important Dockerfile instruction in depth — not just the syntax, but exactly how each one works under the hood and the correct patterns for using them in production.

---

## FROM — Choosing Your Starting Point

`FROM` is the first instruction in every Dockerfile. It sets the base image — the foundation all your subsequent instructions build on top of.

```dockerfile
FROM python:3.11-slim
```

### FROM with a Tag

Always specify a tag. Without one, Docker uses `latest`, which changes over time:

```dockerfile
# Bad — which version is this? Changes over time
FROM python:latest

# Good — pinned to a specific version
FROM python:3.11.7-slim
```

### FROM with a Digest

For absolute reproducibility in production:

```dockerfile
FROM python@sha256:7d9f9f8a0e5e2c7f3b1a2d4e6f8a0b2d4e6f8a0b2d4e6f8a0b2d4e6f8a0b2d4e
```

### FROM scratch

`scratch` is an empty base image — used for statically compiled binaries like Go programs:

```dockerfile
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o app .

FROM scratch
COPY --from=builder /app/app /app
CMD ["/app"]
```

The resulting image contains only your binary. Nothing else. Extremely small and secure.

### Multiple FROM — Multi-Stage Builds

A Dockerfile can have multiple `FROM` instructions. Each one starts a new build stage:

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY . .
RUN npm ci && npm run build

FROM nginx:1.25-alpine AS production
COPY --from=builder /app/dist /usr/share/nginx/html
```

We cover this in depth in the multi-stage builds file.

---

## WORKDIR — Setting the Working Directory

`WORKDIR` sets the directory that subsequent instructions (`RUN`, `COPY`, `ADD`, `CMD`, `ENTRYPOINT`) execute from inside the container.

```dockerfile
WORKDIR /app
```

If the directory does not exist, Docker creates it automatically.

### Always Use WORKDIR Instead of RUN mkdir + cd

```dockerfile
# Bad — verbose and unnecessary
RUN mkdir -p /app
RUN cd /app

# Good — one instruction does both
WORKDIR /app
```

### Multiple WORKDIR Instructions

You can use `WORKDIR` multiple times. Each one is relative to the previous:

```dockerfile
WORKDIR /app
WORKDIR src        # now at /app/src
WORKDIR utils      # now at /app/src/utils
```

Stick to a single `WORKDIR` in most Dockerfiles to avoid confusion.

### WORKDIR Affects CMD and ENTRYPOINT

The working directory set by `WORKDIR` is also the directory that `CMD` and `ENTRYPOINT` commands run from:

```dockerfile
WORKDIR /app
CMD ["python", "app.py"]   # runs python /app/app.py effectively
```

---

## COPY — Adding Files to the Image

`COPY` copies files and directories from the build context (your local machine) into the image.

```dockerfile
# Copy a single file
COPY requirements.txt .

# Copy a single file to a specific path
COPY requirements.txt /app/requirements.txt

# Copy a directory
COPY src/ /app/src/

# Copy everything in the current directory
COPY . .

# Copy multiple sources to a destination
COPY file1.txt file2.txt /app/
```

### COPY vs ADD

Prefer `COPY` over `ADD` for simple file copying. `ADD` has extra behaviors that can be surprising:

| Feature | COPY | ADD |
|---|---|---|
| Copy local files | ✅ | ✅ |
| Copy from URLs | ❌ | ✅ |
| Auto-extract tar archives | ❌ | ✅ |

Use `COPY` for clarity. Only use `ADD` when you need auto-extraction.

```dockerfile
# Good — use COPY for regular files
COPY app.py /app/

# Use ADD only when you need extraction
ADD myapp-1.0.tar.gz /app/   # extracts automatically
```

### COPY --chown

Set file ownership in the same instruction:

```dockerfile
# Set ownership to the appuser when copying
COPY --chown=appuser:appgroup . /app/
```

This is more efficient than running `chown` with `RUN` afterward (which would create an extra layer).

### COPY --from (Multi-Stage)

In multi-stage builds, `COPY --from` copies files from a previous stage:

```dockerfile
FROM python:3.11-slim AS builder
RUN pip install --user -r requirements.txt

FROM python:3.11-slim AS production
COPY --from=builder /root/.local /root/.local
```

---

## RUN — Executing Build Commands

`RUN` executes a command during the build. The resulting filesystem changes become a new image layer.

There are two forms:

```dockerfile
# Shell form — runs in /bin/sh -c
RUN apt-get update && apt-get install -y curl

# Exec form — runs directly, no shell processing
RUN ["apt-get", "update"]
```

Use the shell form for most commands — it supports shell features like `&&`, pipes, and variable expansion.

### The Golden Rule: Combine Related Commands

Every `RUN` instruction creates a new layer. Cleanup done in a separate `RUN` does not remove data from the previous layer.

```dockerfile
# Bad — cleanup happens in a separate layer, previous layer still has cache
RUN apt-get update
RUN apt-get install -y curl wget
RUN rm -rf /var/lib/apt/lists/*

# Good — all in one RUN, cleanup in the same layer
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      curl \
      wget && \
    rm -rf /var/lib/apt/lists/*
```

### --no-install-recommends

Always use `--no-install-recommends` with `apt-get install`. It prevents apt from installing suggested packages that are not actual dependencies:

```dockerfile
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      libpq-dev \
      gcc && \
    rm -rf /var/lib/apt/lists/*
```

### RUN for Python

```dockerfile
# Install dependencies without caching pip's download cache
RUN pip install --no-cache-dir -r requirements.txt

# Install a specific package
RUN pip install --no-cache-dir gunicorn==21.2.0

# Install with hash verification
RUN pip install --require-hashes -r requirements.txt
```

### RUN for Node.js

```dockerfile
# Install exact versions from package-lock.json
RUN npm ci

# Install production dependencies only
RUN npm ci --only=production

# Clear npm cache in the same layer
RUN npm ci --only=production && npm cache clean --force
```

---

## ENV — Environment Variables

`ENV` sets environment variables that are available both during the build and when containers run.

```dockerfile
# Set single variable
ENV APP_ENV=production

# Set multiple variables (preferred — single layer)
ENV APP_ENV=production \
    PORT=8080 \
    LOG_LEVEL=info
```

### ENV vs ARG

| Feature | ENV | ARG |
|---|---|---|
| Available at build time | ✅ | ✅ |
| Available at runtime | ✅ | ❌ |
| Can be overridden at runtime | ✅ (`-e` flag) | ❌ |
| Visible in `docker inspect` | ✅ | Only during build |

```dockerfile
# ARG — build-time only
ARG APP_VERSION=1.0
ARG BUILD_DATE

# ENV — build and runtime
ENV APP_ENV=production
ENV APP_VERSION=${APP_VERSION}   # can use ARG value in ENV
```

### Never Put Secrets in ENV

Environment variables set with `ENV` are visible in the image metadata. Anyone who can pull the image can see them.

```dockerfile
# NEVER do this — password visible to anyone with image access
ENV DATABASE_PASSWORD=mysecretpassword

# Instead, pass secrets at runtime
docker run -e DATABASE_PASSWORD=mysecretpassword myapp:1.0

# Or use AWS Secrets Manager — fetch secrets in your app code at startup
```

---

## ARG — Build-Time Variables

`ARG` defines variables that users can pass during `docker build` with `--build-arg`.

```dockerfile
ARG APP_VERSION=1.0
ARG BUILD_DATE
ARG GIT_COMMIT
```

Build with arguments:

```bash
docker build \
  --build-arg APP_VERSION=2.0 \
  --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
  --build-arg GIT_COMMIT=$(git rev-parse HEAD) \
  -t myapp:2.0 .
```

### Using ARG for Conditional Logic

```dockerfile
ARG ENV=production

RUN if [ "$ENV" = "development" ]; then \
      pip install -r requirements-dev.txt; \
    else \
      pip install -r requirements.txt; \
    fi
```

### ARG and Cache Invalidation

Changing an `ARG` value invalidates the build cache for all subsequent layers. Place `ARG` instructions just before the `RUN` that uses them to minimize cache invalidation:

```dockerfile
# All instructions before this are cached regardless of ARG value
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .

# Place ARG just before where it is used
ARG GIT_COMMIT=unknown
LABEL git-commit=${GIT_COMMIT}
```

---

## CMD and ENTRYPOINT — Container Startup

These two instructions define what runs when the container starts. Understanding the difference between them is critical.

### CMD — Default Command

`CMD` sets the default command. It can be completely overridden by the user.

```dockerfile
CMD ["python", "app.py"]
```

```bash
# Uses CMD default
docker run myapp:1.0
# Runs: python app.py

# Overrides CMD entirely
docker run myapp:1.0 python debug.py
# Runs: python debug.py
```

### ENTRYPOINT — Fixed Executable

`ENTRYPOINT` sets the main executable. It cannot be overridden without `--entrypoint`. Arguments after the image name are appended to `ENTRYPOINT`.

```dockerfile
ENTRYPOINT ["python"]
CMD ["app.py"]   # default argument
```

```bash
# Uses ENTRYPOINT + CMD
docker run myapp:1.0
# Runs: python app.py

# Overrides CMD argument only — ENTRYPOINT stays
docker run myapp:1.0 debug.py
# Runs: python debug.py

# Override ENTRYPOINT requires --entrypoint flag
docker run --entrypoint bash myapp:1.0
```

### When to Use Each

| Use | Approach |
|---|---|
| Simple script or server | `CMD ["python", "app.py"]` |
| Wrapper script with sub-commands | `ENTRYPOINT ["./entrypoint.sh"]` |
| CLI tool that takes arguments | `ENTRYPOINT ["terraform"]` + `CMD ["--help"]` |

### Always Use Exec Form

Use the exec form (JSON array) for both `CMD` and `ENTRYPOINT`:

```dockerfile
# Bad — shell form (PID 1 is /bin/sh, not your app)
CMD python app.py

# Good — exec form (PID 1 is your app directly)
CMD ["python", "app.py"]
```

The exec form makes your process PID 1 directly, which correctly receives signals like `SIGTERM` for graceful shutdown.

---

## USER — Running as Non-Root

By default, containers run as root. This is a security risk — a vulnerability in your application could give an attacker root access to the container and potentially the host.

Always create and use a non-root user:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Create a non-root user and group
RUN groupadd --gid 1001 appgroup && \
    useradd --uid 1001 --gid appgroup --no-create-home appuser

# Install dependencies as root (needed to install system packages)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application files with correct ownership
COPY --chown=appuser:appgroup . .

# Switch to non-root user before CMD
USER appuser

CMD ["python", "app.py"]
```

For Alpine-based images, use `addgroup` and `adduser`:

```dockerfile
FROM python:3.11-alpine

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY --chown=appuser:appgroup . .

USER appuser
CMD ["python", "app.py"]
```

---

## EXPOSE — Documenting Ports

`EXPOSE` documents which ports the container uses. It does not actually publish or open the port — it is documentation for humans and tooling.

```dockerfile
EXPOSE 8080        # TCP by default
EXPOSE 8080/tcp    # explicit TCP
EXPOSE 5353/udp    # UDP port
EXPOSE 80 443      # multiple ports
```

To actually make the port accessible, you use `-p` at runtime:

```bash
docker run -p 8080:8080 myapp:1.0
```

`EXPOSE` tells Docker Compose and orchestrators (ECS, Kubernetes) which ports to route traffic to, so always include it.

---

## HEALTHCHECK — Container Health Monitoring

`HEALTHCHECK` defines a command Docker runs periodically to check if the container is working correctly.

```dockerfile
HEALTHCHECK --interval=30s \
            --timeout=10s \
            --start-period=40s \
            --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

Options:
- `--interval` — how often to check (default: 30s)
- `--timeout` — how long to wait for the check (default: 30s)
- `--start-period` — grace period on startup before checks count as failures (default: 0s)
- `--retries` — how many consecutive failures before container is unhealthy (default: 3)

The command must exit with:
- `0` — healthy
- `1` — unhealthy

On AWS ECS, unhealthy containers are automatically replaced. Health checks are essential for zero-downtime deployments.

---

## LABEL — Image Metadata

`LABEL` adds metadata to your image as key-value pairs. Good labeling makes images easier to manage in production.

```dockerfile
LABEL maintainer="platform-team@example.com" \
      version="1.4.2" \
      description="API service for myapp" \
      environment="production" \
      git-commit="a1b2c3d4" \
      build-date="2024-01-15T09:23:41Z"
```

Query images by label:

```bash
docker images --filter label=environment=production
docker inspect --format '{{json .Config.Labels}}' myapp:1.0
```

---

## Summary

- `FROM` sets the base image — always use a specific version tag, never `latest`
- `WORKDIR` sets the working directory — use it instead of `RUN mkdir && cd`
- `COPY` copies files from the build context — prefer it over `ADD` for regular files
- `RUN` executes build commands — combine related commands in one `RUN` and clean up in the same instruction
- `ENV` sets environment variables for build and runtime — never put secrets in `ENV`
- `ARG` sets build-time variables — passed with `--build-arg`, not available at runtime
- `CMD` sets the default command (overridable) — `ENTRYPOINT` sets the fixed executable
- Always use exec form (`["command", "arg"]`) for `CMD` and `ENTRYPOINT` so PID 1 is your app
- `USER` switches to a non-root user — always do this before `CMD` for security
- `EXPOSE` documents ports — does not publish them; `HEALTHCHECK` defines a liveness check

In the next file, we cover the build process in depth — how to build images, use build arguments, leverage the build cache, and troubleshoot build failures.
