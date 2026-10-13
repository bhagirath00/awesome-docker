# Restart Policies

## Introduction

When a container's main process exits — whether cleanly or due to a crash — what should Docker do? Leave it stopped? Restart it automatically? Restart it only if it crashed?

The restart policy answers these questions. Choosing the right policy for each container is a fundamental part of running Docker in production. A wrong policy can mean your critical service stays down after a crash, or a broken container restarts endlessly and burns resources.

---

## Why Restart Policies Matter

Consider two scenarios:

**Scenario 1:** You run your API server with no restart policy. The application crashes due to an unhandled exception at 3am. Docker leaves the container stopped. Your users get 502 errors. Your on-call engineer wakes up at 4am to restart it manually.

**Scenario 2:** You run the same API server with `--restart unless-stopped`. The application crashes at 3am. Docker restarts it in milliseconds. The brief outage is limited to the time it takes to restart. Your users see a few failed requests and then the service is back. Your on-call engineer sleeps through the night.

Restart policies are one of the simplest things you can configure and one of the most impactful.

---

## The Four Restart Policies

### no (Default)

Docker never restarts the container automatically. When the main process exits for any reason, the container stays stopped until you manually start it again.

```bash
docker run --restart no nginx
# or just omit --restart (no is the default)
docker run nginx
```

**When to use:** Task containers that should run once and complete. Database migrations, report generation scripts, one-time data imports. You do not want these to restart if they fail — you want to know they failed and fix the problem.

```bash
# Run a migration task — do not restart if it fails
docker run --rm --restart no \
  myapp:1.0 \
  python manage.py migrate
```

### always

Docker always restarts the container regardless of why it stopped — successful exit, crash, `docker stop`. If the container keeps crashing, Docker applies exponential backoff between restarts.

Containers with `--restart always` also start automatically when the Docker daemon starts (e.g., after a server reboot).

```bash
docker run -d --restart always nginx
```

**When to use:** Infrastructure services that must always be running — your primary web server, load balancer, monitoring agent. Use when you never want the container to stop, even after a clean exit.

**Caution:** With `always`, even `docker stop` will restart the container when the daemon restarts. To permanently stop an `always` container, you must remove it:

```bash
# docker stop alone does not prevent always policy from restarting on daemon restart
docker stop webserver
docker start webserver   # docker start restarts it right away

# To permanently stop — remove the container
docker rm -f webserver
```

### unless-stopped

Docker restarts the container automatically unless it was manually stopped with `docker stop`. If the container exits due to a crash or the daemon restarts, Docker brings it back. But if you explicitly stopped it, Docker respects that and does not restart it.

```bash
docker run -d --restart unless-stopped myapp:1.0
```

**When to use:** Most production services — APIs, workers, background jobs. This is the recommended policy for application containers. It gives you automatic crash recovery while still respecting intentional stops during deployments and maintenance.

```bash
# Crash recovery: container crashes → Docker restarts it automatically
# Intentional stop: docker stop myapp → Docker stops it, does not restart
# Daemon restart (e.g., server reboot): → Docker restarts the container

# This makes unless-stopped the ideal choice for most services
docker run -d \
  --name api \
  --restart unless-stopped \
  myapp:1.0
```

### on-failure[:max-retries]

Docker restarts the container only when it exits with a non-zero exit code (indicating a failure). If the container exits cleanly (exit code 0), Docker does not restart it.

Optionally, you can set a maximum number of restart attempts.

```bash
# Restart on failure, unlimited retries
docker run -d --restart on-failure myapp:1.0

# Restart on failure, but give up after 5 attempts
docker run -d --restart on-failure:5 myapp:1.0
```

**When to use:** Containers that should complete successfully and not restart if they do, but should be retried if they fail. Good for batch jobs and data processing pipelines where a transient error (network timeout, temporary unavailability of a dependency) might cause failure, but you want to retry automatically.

```bash
# Retry a batch job up to 3 times on failure
docker run \
  --restart on-failure:3 \
  myapp:1.0 \
  python process_batch.py --date 2024-01-15
```

---

## Restart Backoff

Docker does not restart a crashing container in a tight loop — that would consume all CPU and prevent other work. It applies **exponential backoff** between restart attempts:

| Restart attempt | Wait time |
|---|---|
| 1st restart | immediately |
| 2nd restart | 100ms |
| 3rd restart | 200ms |
| 4th restart | 400ms |
| 5th restart | 800ms |
| Subsequent | capped at 1 minute |

If the container stays running for more than 10 seconds after a restart, Docker resets the backoff counter.

You can see this in action:

```bash
# Container with a crash loop
docker run -d --name crasher --restart on-failure alpine sh -c "exit 1"

docker ps
# NAMES     STATUS
# crasher   Restarting (1) 3 seconds ago

# After a few cycles:
# crasher   Restarting (1) 45 seconds ago
```

---

## Checking and Changing Restart Policy

### Check Current Policy

```bash
# View restart policy of a running container
docker inspect --format '{{.HostConfig.RestartPolicy.Name}}' api
# unless-stopped

# View with max retry count
docker inspect --format '{{.HostConfig.RestartPolicy}}' api
# {unless-stopped 0}
```

### Change Restart Policy

You cannot change the restart policy of a running container using `docker run` — that creates a new container. To change the policy of an existing container without recreating it:

```bash
# Update restart policy of an existing container
docker update --restart always webserver
docker update --restart unless-stopped api
docker update --restart on-failure:5 worker
docker update --restart no task-container
```

`docker update` applies the change immediately without stopping the container.

---

## Restart Policy vs Health Checks

Restart policies react to container exit. Health checks react to container health status without requiring an exit.

| Mechanism | Triggers on | What it does |
|---|---|---|
| Restart policy | Container process exits | Restarts the container |
| Health check | Health check command fails | Marks container as `unhealthy` |

A container can be running (process active) but unhealthy (health check failing). For example:
- Your web server process is running but stuck in an infinite loop not handling requests
- Your database is running but not accepting connections
- Your application started but failed to initialize correctly

Health checks detect these situations. On AWS ECS, an unhealthy container is replaced automatically by ECS — even though its process never exited.

```dockerfile
# Combined restart policy AND health check
# Restart policy handles crashes
# Health check handles "running but broken" situations
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

```bash
docker run -d \
  --name api \
  --restart unless-stopped \
  --health-cmd "curl -f http://localhost:8080/health || exit 1" \
  --health-interval 30s \
  --health-retries 3 \
  myapp:1.0
```

---

## Restart Policies in Docker Compose

```yaml
version: "3.8"

services:
  # Production API — restart on crash, respect manual stops
  api:
    image: myapp:1.0
    restart: unless-stopped
    ports:
      - "8080:8080"

  # Infrastructure — always running, survives reboots
  nginx:
    image: nginx:1.25
    restart: always
    ports:
      - "80:80"
      - "443:443"

  # Background worker — retry on failure, stop cleanly when done
  worker:
    image: myapp:1.0
    restart: on-failure
    command: python worker.py

  # One-off task — never restart
  migration:
    image: myapp:1.0
    restart: "no"
    command: python manage.py migrate
```

---

## Restart Policies on AWS ECS

On ECS, restart policies are defined differently. ECS manages container health and restart behavior through:

1. **Essential containers** — if an essential container stops (for any reason), ECS stops the entire task and reschedules it

```json
{
  "containerDefinitions": [
    {
      "name": "api",
      "essential": true,   // if this container stops, the task is replaced
      "image": "...",
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
        "interval": 30,
        "timeout": 10,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ]
}
```

2. **ECS Service** — the service definition specifies the desired number of running tasks. ECS continuously ensures that number is met by replacing failed tasks.

```json
{
  "serviceName": "myapp-api",
  "desiredCount": 3,
  "deploymentConfiguration": {
    "minimumHealthyPercent": 50,
    "maximumPercent": 200
  }
}
```

The ECS service is the equivalent of `--restart unless-stopped` at the orchestration level — ECS always maintains the desired number of healthy tasks.

---

## Policy Decision Guide

| Container type | Recommended policy | Reasoning |
|---|---|---|
| Web server / API | `unless-stopped` | Auto-recover from crashes, respect intentional deploys |
| Background worker | `unless-stopped` | Same as API |
| Database (standalone) | `always` | Critical infrastructure, must survive reboots |
| Nginx / proxy | `always` | Infrastructure, always needed |
| Batch job / migration | `no` | Must complete cleanly, investigate failures |
| CI runner / build job | `no` | One-time task, should not retry |
| Sidecar / monitoring agent | `always` | Must always be running |

---

## Summary

- `no` — never restarts; use for task containers and one-off jobs
- `always` — restarts in all cases, including after daemon restart; use for critical infrastructure that must survive server reboots
- `unless-stopped` — restarts after crashes and daemon restarts, but not after `docker stop`; use for most production application services
- `on-failure[:N]` — restarts only on non-zero exit codes; optionally with a maximum retry count; use for jobs that should be retried on transient failure
- Docker applies exponential backoff (up to 1 minute) between restart attempts to prevent a crash loop from consuming all resources
- Change the policy of an existing container without recreating it using `docker update --restart POLICY container_name`
- Combine restart policies with health checks — restart policy handles crashes, health checks handle "running but broken" scenarios
- On AWS ECS, the service's `desiredCount` replaces restart policies — ECS reschedules tasks automatically when they fail

In the next chapter, we cover container storage — how data persists beyond the container lifecycle and the different options for managing storage in Docker.
