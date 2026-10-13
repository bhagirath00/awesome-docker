# Health Checks in Docker Compose

## Introduction

Health checks are a mechanism for Docker to actively verify that a container is functioning correctly — not just that the container is running, but that the application inside it is responding properly. A container can be running but completely unresponsive: stuck in a deadlock, failed to initialize, out of memory and garbage collecting constantly, or connected to a broken database.

Health checks detect these situations and expose the container's status via Docker's `health_status` field. This integrates with `depends_on` for startup ordering, with Docker's own restart policies, and with AWS ECS for automatic task replacement.

---

## How Health Checks Work

When a health check is defined, Docker runs the specified command inside the container at a regular interval. Based on the exit code:

- **Exit 0** — healthy
- **Exit 1** — unhealthy
- **Exit 2** — reserved (do not use)

Docker tracks a rolling window of results. If the check fails more than `retries` times consecutively, the container is marked `unhealthy`. The `start_period` allows a grace window after container start before failures count.

```bash
# See the health status of all containers
docker ps --format "table {{.Names}}\t{{.Status}}"

# Output:
# myapp_api_1     Up 5 minutes (healthy)
# myapp_db_1      Up 5 minutes (healthy)
# myapp_redis_1   Up 5 minutes (healthy)
# myapp_worker_1  Up 2 minutes (starting)  ← in start_period
```

---

## Defining Health Checks in Docker Compose

```yaml
services:
  api:
    image: myapp:1.0
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s      # how often to run the check
      timeout: 10s       # max time for the command to complete
      retries: 3         # consecutive failures before marking unhealthy
      start_period: 40s  # grace period after startup
```

### The test Field

The `test` field specifies the command to run. It must be one of:

```yaml
# List form (recommended — no shell interpretation issues)
test: ["CMD", "curl", "-f", "http://localhost:8080/health"]

# Shell form (wraps in /bin/sh -c)
test: ["CMD-SHELL", "pg_isready -U apiuser -d myapp"]

# Disable healthcheck from image
test: ["NONE"]
```

Use `CMD` when the binary is directly available and you do not need shell features.
Use `CMD-SHELL` when you need shell features like environment variable expansion, pipes, or redirection.

```yaml
# CMD — no shell needed
test: ["CMD", "redis-cli", "ping"]

# CMD-SHELL — need shell for variable expansion
test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB"]
# Note: $$ escapes to $ in Compose YAML
```

---

## Health Check Parameters

| Parameter | Default | Description |
|---|---|---|
| `interval` | 30s | How often Docker runs the check |
| `timeout` | 30s | Max time for the check command to complete |
| `retries` | 3 | Consecutive failures before `unhealthy` |
| `start_period` | 0s | Grace window after container start |

### Choosing Parameters

**`interval`**: Balance between responsiveness and overhead. 10-30 seconds is typical for most services. For databases and caches, 5-10 seconds gives faster health feedback.

**`timeout`**: Should be less than `interval`. If the check takes longer than `timeout`, it counts as a failure. Set to a value where a slow-but-working service still passes.

**`retries`**: How many consecutive failures before marking unhealthy. 3-5 is typical. Higher values tolerate transient failures better but delay detecting genuine issues.

**`start_period`**: Critical for services that take time to initialize. PostgreSQL, Elasticsearch, and JVM-based services can take 10-60+ seconds to become ready. During `start_period`, failed checks do not count toward `retries` — the container stays `starting` rather than flipping to `unhealthy` while initializing.

```yaml
# Elasticsearch — takes 30-60 seconds to start
services:
  elasticsearch:
    image: elasticsearch:8.12.0
    healthcheck:
      test: ["CMD-SHELL", "curl -s --fail http://localhost:9200/_cluster/health | grep -q '\"status\":\"green\"\\|\"status\":\"yellow\"'"]
      interval: 15s
      timeout: 10s
      retries: 5
      start_period: 60s   # give it a full minute to start
```

---

## Health Checks for Common Services

### PostgreSQL

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: apiuser
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: myapp
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 20s
```

`pg_isready` exits 0 when PostgreSQL is accepting connections on the default socket. It is the official tool for this purpose — available in all PostgreSQL images.

### MySQL/MariaDB

```yaml
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: myapp
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "127.0.0.1", "--silent"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
```

### Redis

```yaml
services:
  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
```

`redis-cli ping` returns `PONG` and exits 0 when Redis is ready. Simple and reliable.

### RabbitMQ

```yaml
services:
  rabbitmq:
    image: rabbitmq:3.12-management-alpine
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "check_port_connectivity"]
      interval: 15s
      timeout: 10s
      retries: 5
      start_period: 30s
```

### Application API (HTTP)

```yaml
services:
  api:
    image: myapp:1.0
    healthcheck:
      test: ["CMD", "curl", "-f", "--max-time", "5", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

The `/health` endpoint in your application should:
- Return HTTP 200 when healthy
- Verify database connectivity
- Verify cache connectivity
- Respond in under 5 seconds
- Never return 200 if critical dependencies are unavailable

### Application API Without curl (Minimal Images)

If your image is based on `alpine` and does not have `curl`:

```yaml
healthcheck:
  test: ["CMD-SHELL", "wget -qO- --tries=1 --timeout=5 http://localhost:8080/health || exit 1"]
```

Or install curl in your Dockerfile:

```dockerfile
FROM python:3.11-slim
RUN apt-get update && apt-get install -y --no-install-recommends curl && rm -rf /var/lib/apt/lists/*
```

---

## Health Checks and depends_on

Health checks become essential when combined with `depends_on condition: service_healthy`:

```yaml
services:
  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U apiuser -d myapp"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s

  api:
    image: myapp:1.0
    depends_on:
      db:
        condition: service_healthy   # requires db healthcheck to pass
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

Compose will not start `api` until `db`'s health check reports healthy.

---

## Overriding Health Checks from Images

Some Docker Hub images define their own health checks. You can override or disable them:

```yaml
services:
  redis:
    image: redis:7-alpine
    healthcheck:
      # Override the image's default healthcheck with our own
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Disable the image's healthcheck entirely
  some-service:
    image: third-party-image:1.0
    healthcheck:
      disable: true
```

---

## Health Checks in AWS ECS

In ECS task definitions, health checks work the same way but with ECS-specific fields:

```json
{
  "containerDefinitions": [
    {
      "name": "api",
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
        "interval": 30,         # seconds (integer, not string)
        "timeout": 10,
        "retries": 3,
        "startPeriod": 60       # seconds
      }
    }
  ]
}
```

In ECS:
- An unhealthy container causes ECS to stop and replace the task
- The ECS service waits for the new task to become healthy before routing traffic to it (when using an ALB)
- ELB health checks (configured on the target group) are separate from container health checks — both run independently

---

## Monitoring Health Status

```bash
# Quick check — all container statuses including health
docker ps

# Detailed health info for a specific container
docker inspect myapp_api_1 --format '{{json .State.Health}}' | python3 -m json.tool

# Output:
{
  "Status": "healthy",
  "FailingStreak": 0,
  "Log": [
    {
      "Start": "2024-01-15T09:23:41Z",
      "End": "2024-01-15T09:23:41.123Z",
      "ExitCode": 0,
      "Output": "OK"
    }
  ]
}

# Watch health changes in real time
docker compose events --filter event=health_status
```

---

## Summary

- Health checks let Docker verify that a container's application is actually functioning, not just that the container process is running
- Define health checks using `test` (the command), `interval` (how often), `timeout` (max execution time), `retries` (failures before unhealthy), and `start_period` (startup grace period)
- Use `CMD` when you do not need shell features; use `CMD-SHELL` when you need environment variable expansion or pipes
- `start_period` is critical for services with slow startup (databases, JVM apps, Elasticsearch) — prevents false unhealthy marks during initialization
- Health checks unlock `depends_on condition: service_healthy` — the correct way to wait for database and cache readiness before starting the application
- In AWS ECS, the same health check concept applies — an unhealthy container is automatically stopped and replaced by ECS, with the ALB waiting for the replacement to be healthy before routing traffic

This completes the Compose Configuration chapter. In the next chapter, we cover advanced Compose patterns — override files, profiles, and multi-environment deployments.
