# The HEALTHCHECK Instruction

## Introduction

The `HEALTHCHECK` instruction in a Dockerfile embeds health monitoring directly into the image. Any container started from the image automatically runs the health check — no configuration required by the operator. Docker uses the result to report the container's health status and, in orchestrated environments, to route traffic only to healthy containers.

Baking health checks into the image is better than relying on operators to configure them at runtime. The application team knows better than anyone what "healthy" means for their service — that knowledge belongs in the image.

---

## Syntax

```dockerfile
HEALTHCHECK [OPTIONS] CMD command
HEALTHCHECK NONE    # Disable health check inherited from base image
```

### Options

| Option | Default | Description |
|---|---|---|
| `--interval=DURATION` | `30s` | How often the check runs |
| `--timeout=DURATION` | `30s` | Maximum time for the check to complete |
| `--start-period=DURATION` | `0s` | Grace period after container start |
| `--retries=N` | `3` | Failures before marking unhealthy |

---

## Basic Usage

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY src/ .

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

CMD ["gunicorn", "--bind", "0.0.0.0:8080", "app:app"]
```

The health check verifies the application is actually responding to HTTP requests, not just that the Python process is running.

---

## The Health Check Command

The command runs inside the container, using whatever tools are available in the container's environment.

### HTTP Endpoint Check

The most common pattern — verify the application's HTTP health endpoint:

```dockerfile
# Using curl (available on debian/ubuntu-based images)
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD curl -f --max-time 5 http://localhost:${PORT:-8080}/health || exit 1

# Using wget (available on alpine images — curl is not always present)
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD wget -qO- --tries=1 --timeout=5 http://localhost:8080/health || exit 1
```

The `-f` flag on `curl` makes it return a non-zero exit code when the HTTP response is an error code (4xx, 5xx) — without it, `curl` exits 0 even on 404.

### Database Check

For a PostgreSQL container:

```dockerfile
FROM postgres:16

HEALTHCHECK --interval=10s --timeout=5s --retries=5 --start-period=20s \
    CMD pg_isready -U ${POSTGRES_USER:-postgres} -d ${POSTGRES_DB:-postgres} || exit 1
```

For a Redis container:

```dockerfile
FROM redis:7-alpine

HEALTHCHECK --interval=5s --timeout=3s --retries=5 \
    CMD redis-cli ping || exit 1
```

### Process Check (Minimal Approach)

For containers where no network port is exposed (background workers):

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD pgrep -f "python worker.py" || exit 1
```

`pgrep` returns exit 0 if a matching process is found and exit 1 if not.

### Check a File (Simple Liveness)

Applications can write a "heartbeat" file to indicate they are alive:

```dockerfile
HEALTHCHECK --interval=60s --timeout=5s --retries=3 \
    CMD test -f /tmp/healthy || exit 1
```

The application writes `touch /tmp/healthy` periodically. If the application hangs, it stops updating the file, and the health check fails.

---

## Choosing Good Health Check Parameters

### --start-period

This is the most important parameter to get right. During `start-period`, failed checks do not count toward `retries`. The container stays in the `starting` state rather than flipping to `unhealthy` while the application initializes.

```dockerfile
# Python Flask app — starts in ~2 seconds
HEALTHCHECK --start-period=10s ...

# Spring Boot app — JVM startup is 20-60 seconds
HEALTHCHECK --start-period=60s ...

# Elasticsearch — startup can take 1-2 minutes
HEALTHCHECK --start-period=120s ...

# Short-lived migration container — no health check needed (use NONE)
HEALTHCHECK NONE
```

Under-estimating `start-period` causes the container to be marked unhealthy during normal startup. The orchestrator (ECS, Swarm) then kills and restarts the container, creating a restart loop.

### --interval

Balance responsiveness with overhead. For most services:

```dockerfile
# Standard web API — check every 30 seconds
HEALTHCHECK --interval=30s ...

# Database — faster checks for startup detection
HEALTHCHECK --interval=10s ...

# Background worker — less critical, check every minute
HEALTHCHECK --interval=60s ...
```

### --timeout

Must be less than `--interval`. If the check takes longer than `timeout`, it is counted as a failure:

```dockerfile
# Good: timeout < interval
HEALTHCHECK --interval=30s --timeout=10s ...

# Bad: timeout = interval (risky)
HEALTHCHECK --interval=10s --timeout=10s ...
```

### --retries

How many consecutive failures before Docker marks the container `unhealthy`:

```dockerfile
# For startup ordering (service_healthy in Compose) — fewer retries, faster detection
HEALTHCHECK --retries=3 ...

# For databases with variable startup time — more tolerance
HEALTHCHECK --retries=5 ...
```

---

## What Health Checks Enable

### 1. docker ps Status

```bash
docker ps
# NAME         STATUS
# myapp_api_1  Up 5 minutes (healthy)
# myapp_db_1   Up 30 seconds (starting)    ← in start_period
# myapp_old_1  Up 10 minutes (unhealthy)   ← failed retries
```

### 2. depends_on condition: service_healthy in Compose

Health checks embedded in the image are automatically used by `depends_on`:

```yaml
services:
  api:
    depends_on:
      db:
        condition: service_healthy   # uses the health check from the db image
```

No explicit `healthcheck:` needed in the Compose file if the image already has one.

### 3. Automatic Task Replacement in ECS

AWS ECS monitors container health. When a container becomes unhealthy, ECS stops the task and starts a new one. The ALB target group removes the unhealthy container from rotation before ECS stops it, ensuring zero-downtime replacement.

---

## What a Good /health Endpoint Returns

The application-level `/health` endpoint is what the health check validates. Design it properly:

```python
# app.py (Flask example)
import os
import psycopg2
from flask import jsonify

@app.route("/health")
def health():
    checks = {}
    status = 200

    # Check database connectivity
    try:
        conn = psycopg2.connect(os.environ["DATABASE_URL"], connect_timeout=3)
        conn.close()
        checks["database"] = "ok"
    except Exception as e:
        checks["database"] = f"error: {str(e)}"
        status = 503  # Service Unavailable

    # Check Redis connectivity
    try:
        redis_client.ping()
        checks["redis"] = "ok"
    except Exception as e:
        checks["redis"] = f"error: {str(e)}"
        status = 503

    return jsonify({
        "status": "healthy" if status == 200 else "unhealthy",
        "checks": checks
    }), status
```

A `/health` endpoint should:
- Return 200 when all critical dependencies are available
- Return 503 (or any non-2xx) when critical dependencies are down
- Complete within `--timeout` seconds
- Be lightweight — no expensive queries or computations
- Include structured output for debugging (as in the example above)

---

## Disabling an Inherited Health Check

If you extend a base image that has a health check, and your use case does not need it:

```dockerfile
FROM myapp:1.0    # has a HEALTHCHECK baked in

# Disable the inherited health check
HEALTHCHECK NONE
```

This is appropriate for one-off task containers (migrations, seed scripts) that run and exit. A health check on a container that is expected to terminate would trigger false unhealthy alerts.

---

## Health Check in the Context of Build Best Practices

```dockerfile
FROM python:3.11-slim

# Install curl for health check (add before other steps so it caches well)
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app

RUN groupadd --system appuser && \
    useradd --system --no-create-home --gid appuser appuser

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY --chown=appuser:appuser src/ .

USER appuser

EXPOSE 8080

# Health check baked into the image
HEALTHCHECK \
    --interval=30s \
    --timeout=10s \
    --start-period=40s \
    --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

CMD ["gunicorn", "--workers", "4", "--bind", "0.0.0.0:8080", "app:app"]
```

---

## Summary

- `HEALTHCHECK` in a Dockerfile embeds health monitoring into the image — every container started from it automatically runs the check without any operator configuration
- The command exits 0 for healthy and 1 for unhealthy — use `curl -f`, `wget`, `pg_isready`, `redis-cli ping`, or `pgrep` depending on what you are checking
- `--start-period` is the most critical parameter — set it generously to cover the application's full startup time; underestimating causes restart loops during normal startup
- `--timeout` must be shorter than `--interval` — a check that takes too long is counted as a failure
- A health check in the image automatically enables `depends_on condition: service_healthy` in Compose and automatic task replacement in AWS ECS
- Design your `/health` endpoint to verify actual dependencies (database, cache) and return 503 when any critical dependency is unavailable
- Use `HEALTHCHECK NONE` to disable an inherited health check for task containers that are expected to run and exit

In the next file, we explore advanced Dockerfile instructions — ONBUILD, STOPSIGNAL, SHELL, and LABEL — and when each is appropriate.
