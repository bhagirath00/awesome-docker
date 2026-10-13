# Container Lifecycle

## Introduction

Every container goes through a defined lifecycle — from the moment you create it to the moment it is removed. Understanding this lifecycle is not just theoretical knowledge. It directly affects how you write Dockerfiles, how you handle application crashes, how you manage production deployments, and how you debug containers that are not behaving as expected.

This file walks through every state a container can be in, what triggers transitions between states, and what you as a DevOps engineer need to know at each stage.

---

## Container States

A Docker container can be in one of the following states at any given time:

| State | Description |
|---|---|
| **Created** | Container exists but has never been started |
| **Running** | Main process is actively executing |
| **Paused** | Main process is frozen (suspended) |
| **Restarting** | Container is in the process of restarting |
| **Exited** | Main process has stopped (successfully or with an error) |
| **Dead** | Container could not be removed cleanly (rare) |

You can see the current state of any container with:

```bash
docker ps -a --format "table {{.Names}}\t{{.Status}}"
```

Output:

```
NAMES       STATUS
webserver   Up 2 hours
api         Exited (1) 5 minutes ago
worker      Restarting (1) 30 seconds ago
cache       Paused
```

---

## The Full Lifecycle — Step by Step

### Stage 1: Created

A container is in the `created` state when it has been created but not yet started. This happens when you use `docker create` instead of `docker run`.

```bash
# Create a container without starting it
docker create --name webserver -p 8080:80 nginx

# Check — it exists but is not running
docker ps -a | grep webserver
# webserver   Created   2 seconds ago
```

`docker run` is the combination of `docker create` + `docker start`. In most cases you use `docker run` directly. The `created` state is mainly relevant when you need to prepare a container (copy files into it, inspect it) before starting it.

```bash
# Copy a config file into a created (not running) container
docker cp ./nginx.conf webserver:/etc/nginx/nginx.conf

# Now start it
docker start webserver
```

---

### Stage 2: Running

The `running` state is when the container's main process (PID 1) is actively executing.

```bash
# Start the created container
docker start webserver

# Or create and start in one step
docker run -d --name webserver -p 8080:80 nginx

# Confirm it is running
docker ps
# NAMES       IMAGE   STATUS
# webserver   nginx   Up 3 seconds
```

While running, the container:
- Accepts and responds to network connections (if ports are configured)
- Writes logs to stdout/stderr (accessible via `docker logs`)
- Can be interacted with via `docker exec`
- Consumes CPU, memory, and other resources within its configured limits

```bash
# Check resource usage of a running container
docker stats webserver --no-stream

# Output:
CONTAINER   CPU %   MEM USAGE / LIMIT   MEM %   NET I/O     BLOCK I/O
webserver   0.01%   8.5MiB / 512MiB     1.66%   1.2kB/648B  0B/0B
```

---

### Stage 3: Paused

Pausing a container freezes all its processes. The container still exists and consumes memory, but its processes stop executing and it uses no CPU.

```bash
# Pause a container
docker pause webserver

# Check — status shows "Paused"
docker ps | grep webserver
# webserver   Up 5 minutes (Paused)
```

Pausing is rarely used in production but has specific use cases:

- Temporarily freeing CPU on a shared host without stopping the container
- Taking a consistent snapshot of a container's state before backup
- Debugging race conditions by freezing one container while examining another

```bash
# Resume a paused container
docker unpause webserver

# Container continues exactly where it left off
```

---

### Stage 4: Restarting

The `restarting` state appears when a container has stopped and its restart policy is triggering a restart. You will see this when:

- A container crashes repeatedly
- The application inside the container exits with an error

```bash
# Run a container that will crash immediately
docker run -d --restart on-failure --name crasher alpine exit 1

# Check — you will see it restarting
docker ps | grep crasher
# crasher   Restarting (1) 2 seconds ago
```

Docker applies exponential backoff between restarts to prevent a crashing container from hammering the system:

- First restart: immediately
- Second restart: after 100ms
- Third restart: after 200ms
- Fourth restart: after 400ms
- Maximum backoff: 1 minute

This is why a container stuck in a crash loop shows "Restarting (N) X seconds ago" — the number in parentheses is the exit code, and the time indicates when the last restart attempt was.

---

### Stage 5: Exited

A container enters the `exited` state when its main process stops — either successfully or with an error.

```bash
# Container that exits successfully (exit code 0)
docker run --name success alpine echo "done"

# Container that exits with an error (non-zero exit code)
docker run --name failure alpine sh -c "exit 1"

# Check both
docker ps -a | grep -E "success|failure"
# success   Exited (0) 5 seconds ago
# failure   Exited (1) 3 seconds ago
```

**Exit code 0** means the process completed successfully.
**Non-zero exit code** means the process exited with an error.

The exit code tells you what happened:

| Exit Code | Meaning |
|---|---|
| 0 | Success |
| 1 | General error |
| 125 | Docker daemon error |
| 126 | Command cannot be invoked |
| 127 | Command not found |
| 130 | Process terminated by Ctrl+C (SIGINT) |
| 137 | Process killed by SIGKILL (often OOM) |
| 143 | Process terminated by SIGTERM |

Exit code `137` is especially important — it usually means your container was killed because it exceeded its memory limit. If you see this in production, increase the memory limit or fix a memory leak.

```bash
# Check exit code of a stopped container
docker inspect --format '{{.State.ExitCode}}' failure
# 1
```

### Debugging an Exited Container

When a container exits unexpectedly, your first tool is `docker logs`:

```bash
# View logs of an exited container
docker logs failure

# If the logs do not show the error, override the entrypoint and inspect manually
docker run -it --entrypoint bash myapp:1.0
```

---

### Stage 6: Removing

When you remove a container, Docker cleans up:

- The container's writable filesystem layer (all data written inside the container is deleted)
- The container's network configuration
- The container's metadata

```bash
# Remove a stopped container
docker rm webserver

# Force remove a running container (sends SIGKILL first)
docker rm -f webserver

# Remove all stopped containers at once
docker container prune -f
```

After removal, the container is completely gone. The image it was based on still exists locally.

---

## Restart Policies

Restart policies control what happens when a container's main process exits. They are crucial for production deployments.

### Available Restart Policies

```bash
# no — Never restart (default)
docker run --restart no nginx

# always — Always restart, regardless of exit code
# Also starts on Docker daemon startup
docker run --restart always nginx

# unless-stopped — Restart unless manually stopped
# Does NOT restart if you explicitly ran docker stop
docker run --restart unless-stopped nginx

# on-failure — Restart only when exit code is non-zero
# Optionally limit the number of restarts
docker run --restart on-failure:5 nginx
```

### Which Policy to Use?

| Policy | When to use |
|---|---|
| `no` | Short-lived task containers (build, test, migration) |
| `always` | Critical services that must always be up (database, proxy) |
| `unless-stopped` | Application services — restart automatically, but respect manual stops |
| `on-failure` | Services that should restart on crash but stop on clean exit |

For most production application containers, use `unless-stopped`:

```bash
docker run -d \
  --name api \
  --restart unless-stopped \
  -p 8080:8080 \
  -e APP_ENV=production \
  myapp:1.0
```

---

## Stopping Containers — SIGTERM vs SIGKILL

Understanding how Docker stops containers matters for graceful shutdowns — especially critical when your application needs to finish in-flight requests or flush data before stopping.

### docker stop (Graceful)

`docker stop` sends `SIGTERM` to PID 1. This gives the process a chance to shut down cleanly. After a grace period (10 seconds by default), if the process is still running, Docker sends `SIGKILL`.

```bash
# Stop with default 10-second grace period
docker stop webserver

# Stop with 30-second grace period (give app more time to finish requests)
docker stop --time 30 webserver
```

Your application should handle `SIGTERM` by:
1. Stopping accepting new requests
2. Finishing in-flight requests
3. Flushing buffers and closing database connections
4. Exiting cleanly with code 0

### docker kill (Immediate)

`docker kill` sends `SIGKILL` directly — the process is terminated immediately with no chance to clean up.

```bash
# Immediate termination — no cleanup
docker kill webserver
```

Use `docker kill` only when a container is frozen and not responding to `docker stop`.

---

## Automatic Container Removal

For containers that run a task and exit, use `--rm` to remove them automatically on exit:

```bash
# Remove container automatically when it exits
docker run --rm \
  -v $(pwd):/app \
  python:3.11 \
  python /app/run_migration.py
```

This is useful for:
- Database migrations
- Build steps
- Test runners
- One-time scripts

Without `--rm`, every exited container accumulates on the host, consuming disk space and cluttering `docker ps -a`.

---

## Health Checks

Docker can actively monitor whether your container is healthy — not just whether it is running, but whether it is working correctly.

```bash
# Run a container with a health check
docker run -d \
  --name webserver \
  --health-cmd "curl -f http://localhost/health || exit 1" \
  --health-interval 30s \
  --health-timeout 10s \
  --health-retries 3 \
  --health-start-period 40s \
  nginx
```

Health check states:

- `starting` — container is starting, health check not yet run
- `healthy` — health check passed
- `unhealthy` — health check failed the configured number of times

```bash
# Check health status
docker inspect --format '{{.State.Health.Status}}' webserver
# healthy

# View health check history
docker inspect --format '{{json .State.Health}}' webserver | python3 -m json.tool
```

On AWS ECS and Kubernetes, health checks are used to determine if a container should receive traffic and when to restart or replace it. A container that is `unhealthy` can be automatically replaced without manual intervention.

---

## Summary

- Containers move through states: Created → Running → Paused → Restarting → Exited → Removed
- The container's lifecycle is entirely controlled by its main process (PID 1) — when PID 1 exits, the container exits
- Exit codes tell you exactly what happened: 0 is success, 137 usually means OOM killed, 143 means SIGTERM
- Restart policies control automatic recovery: use `unless-stopped` for production services and `no` for task containers
- `docker stop` sends SIGTERM (graceful), `docker kill` sends SIGKILL (immediate) — always prefer `docker stop`
- Use `--rm` for task containers that should clean themselves up after completion
- Health checks let Docker and orchestrators like ECS know whether a container is actually working, not just running

In the next file, we go hands-on with running containers — all the patterns and options you need for real DevOps work.
