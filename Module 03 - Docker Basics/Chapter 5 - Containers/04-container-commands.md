# Container Commands

## Introduction

Knowing how to start a container is only the beginning. The real skill in working with Docker is managing containers once they are running — reading their logs, executing commands inside them, monitoring their resources, and inspecting their configuration.

These are the commands you use daily in a DevOps role. When something is wrong in production, these are the tools you reach for first.

---

## docker logs — Reading Container Output

All output that your container's processes write to stdout and stderr is captured by Docker and available through `docker logs`.

This is your first tool when debugging a container that is not behaving correctly.

```bash
# View all logs from a container
docker logs webserver

# Follow logs in real time (like tail -f)
docker logs -f webserver

# Show only the last N lines
docker logs --tail 50 webserver

# Show logs with timestamps
docker logs -t webserver

# Combine: last 100 lines with timestamps, followed live
docker logs --tail 100 -t -f webserver

# Show logs since a specific time
docker logs --since 2024-01-15T09:00:00 webserver

# Show logs since a relative time
docker logs --since 30m webserver

# Show logs up to a specific time
docker logs --until 2024-01-15T10:00:00 webserver
```

### Important: Logs from Exited Containers

You can still read logs from a container that has already stopped:

```bash
# Start a container that will fail
docker run --name crasher myapp:1.0

# Container exited — read its logs to find out why
docker logs crasher
```

This is critical for post-mortem analysis of crashed containers.

### Log Drivers

By default, Docker stores logs as JSON files on the host. In production, you often want to send logs to a centralized system instead.

```bash
# Send logs to AWS CloudWatch Logs
docker run -d \
  --name api \
  --log-driver awslogs \
  --log-opt awslogs-region=us-east-1 \
  --log-opt awslogs-group=/myapp/production/api \
  --log-opt awslogs-stream=api-server-1 \
  myapp:1.0
```

On AWS ECS, you configure CloudWatch logging in the task definition. Every container's logs automatically go to CloudWatch — no log agent needed.

```json
{
  "logConfiguration": {
    "logDriver": "awslogs",
    "options": {
      "awslogs-group": "/ecs/myapp-production",
      "awslogs-region": "us-east-1",
      "awslogs-stream-prefix": "api"
    }
  }
}
```

---

## docker exec — Running Commands in Running Containers

`docker exec` lets you run a command inside a container that is already running. This is your go-to tool for debugging, inspecting state, and running maintenance tasks.

```bash
# Run a single command in a running container
docker exec webserver ls -la /etc/nginx

# Run a command and see the output
docker exec webserver cat /etc/nginx/nginx.conf

# Open an interactive shell inside a running container
docker exec -it webserver bash

# Use sh if bash is not available (common in Alpine images)
docker exec -it webserver sh

# Set environment variables for the exec session
docker exec -it -e DEBUG=true api bash

# Run as a specific user
docker exec -it --user root webserver bash
docker exec -it --user nginx webserver bash

# Set the working directory
docker exec -it -w /app api bash
```

### Common Debugging Patterns with exec

```bash
# Check what processes are running inside a container
docker exec webserver ps aux

# Check network connectivity from inside the container
docker exec api curl http://database:5432

# Check environment variables
docker exec api env | sort

# Check disk usage inside the container
docker exec api df -h

# Test database connectivity
docker exec api python -c "import psycopg2; conn = psycopg2.connect('postgresql://user:pass@db/myapp'); print('Connected')"

# Check what files changed in the container filesystem
docker exec api find /app -newer /app/requirements.txt -type f
```

### exec vs Running a New Container

`docker exec` runs inside an already-running container. It shares the container's:
- Filesystem
- Network
- Environment variables
- Running processes

This is different from `docker run`, which creates a brand new container. Use `exec` when you want to inspect a running service. Use `run` when you want a fresh environment.

---

## docker inspect — Getting Full Container Details

`docker inspect` returns a complete JSON document with everything Docker knows about a container or image.

```bash
# Get full JSON details
docker inspect webserver

# The output is large — pipe to less for easier reading
docker inspect webserver | less

# Or pretty-print it
docker inspect webserver | python3 -m json.tool
```

### Extracting Specific Information with Templates

The real power of `inspect` is the `--format` flag with Go templates, which lets you extract exactly the field you need:

```bash
# Get the container's IP address
docker inspect --format '{{.NetworkSettings.IPAddress}}' webserver
# 172.17.0.2

# Get the container's state
docker inspect --format '{{.State.Status}}' webserver
# running

# Check if container is running
docker inspect --format '{{.State.Running}}' webserver
# true

# Get the exit code
docker inspect --format '{{.State.ExitCode}}' webserver
# 0

# Check if OOM killed
docker inspect --format '{{.State.OOMKilled}}' webserver
# false

# Get the image used
docker inspect --format '{{.Config.Image}}' webserver
# nginx:1.25.3

# Get all environment variables
docker inspect --format '{{range .Config.Env}}{{println .}}{{end}}' webserver

# Get the mounted volumes
docker inspect --format '{{json .Mounts}}' webserver | python3 -m json.tool

# Get labels
docker inspect --format '{{json .Config.Labels}}' webserver | python3 -m json.tool

# Get restart count (useful for detecting crash loops)
docker inspect --format '{{.RestartCount}}' webserver
# 3
```

---

## docker stats — Live Resource Monitoring

`docker stats` shows a live stream of resource usage for containers:

```bash
# Monitor all running containers
docker stats

# Monitor specific containers
docker stats webserver api postgres

# Show stats once without live updating
docker stats --no-stream

# Custom output format
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}"
```

Output:

```
CONTAINER   CPU %   MEM USAGE / LIMIT   MEM %   NET I/O           BLOCK I/O
webserver   0.01%   8.5MiB / 512MiB     1.66%   1.2kB / 648B      0B / 0B
api         1.23%   256MiB / 1GiB       25.0%   45.2MB / 12.1MB   2.1MB / 0B
postgres    0.45%   89MiB / 512MiB      17.4%   1.1MB / 980kB     5.2MB / 8.4MB
```

Use `docker stats` when you suspect a container is consuming too many resources or when you want to set appropriate resource limits.

---

## docker top — Processes Inside a Container

`docker top` shows the processes running inside a container without needing to exec into it:

```bash
docker top webserver
```

Output:

```
UID    PID    PPID   C   STIME  TTY  TIME      CMD
root   15234  15210  0   09:30  ?    00:00:00  nginx: master process nginx -g daemon off;
www-data 15235 15234 0  09:30  ?    00:00:00  nginx: worker process
```

---

## docker cp — Copying Files To and From Containers

`docker cp` copies files between your host machine and a container. It works on both running and stopped containers.

```bash
# Copy a file from container to host
docker cp webserver:/etc/nginx/nginx.conf ./nginx-backup.conf

# Copy a directory from container to host
docker cp webserver:/var/log/nginx ./nginx-logs/

# Copy a file from host to container
docker cp ./my-nginx.conf webserver:/etc/nginx/nginx.conf

# Copy a directory from host to container
docker cp ./ssl-certs/ webserver:/etc/ssl/

# Copy from a stopped container (works the same way)
docker cp stopped-container:/app/config.yml ./config-backup.yml
```

Common uses:
- Recovering logs or data from a crashed container before removing it
- Temporarily injecting a config file to test a fix without rebuilding
- Extracting build artifacts from a build container

---

## docker diff — What Changed in the Container Filesystem

`docker diff` shows you what files were added, modified, or deleted in the container's writable layer compared to the original image:

```bash
docker diff webserver

# Output:
C /var     ← Changed
C /var/log
C /var/log/nginx
A /var/log/nginx/access.log   ← Added
A /var/log/nginx/error.log    ← Added
C /etc/nginx/nginx.conf       ← Changed
```

Prefixes:
- `A` — file was Added
- `C` — file was Changed
- `D` — file was Deleted

This is useful for debugging what a container is writing to disk or verifying that a config change took effect.

---

## docker rename — Renaming Containers

You can rename a running or stopped container without recreating it:

```bash
# Rename a container
docker rename old-name new-name

# Rename to fix a naming mistake
docker rename webserver-v1 webserver-production
```

---

## Combining Commands — Real Debugging Workflows

Here are complete workflows for common situations you will face on the job.

### Debugging a Container That Keeps Restarting

```bash
# Step 1: See restart count and last exit code
docker inspect --format 'Restarts: {{.RestartCount}} | Exit Code: {{.State.ExitCode}} | OOM Killed: {{.State.OOMKilled}}' myapp

# Step 2: Read recent logs
docker logs --tail 100 myapp

# Step 3: If OOM killed, check memory usage pattern
docker stats myapp --no-stream

# Step 4: Try running the container interactively to reproduce the error
docker run -it --entrypoint bash myapp:1.0
```

### Debugging a Container That Started But Is Not Responding

```bash
# Step 1: Confirm it is running
docker ps | grep myapp

# Step 2: Check if the process is actually listening on the right port
docker exec myapp netstat -tlnp

# Step 3: Try reaching the service from inside the container
docker exec myapp curl -s http://localhost:8080/health

# Step 4: Check logs for errors
docker logs --tail 50 -t myapp

# Step 5: Check if there are any environment variable issues
docker exec myapp env | grep -i database
```

### Recovering Data from a Crashed Container

```bash
# Step 1: Find the stopped container
docker ps -a | grep myapp

# Step 2: Copy out the data before removing
docker cp crashed-myapp:/app/uploads ./uploads-backup/
docker cp crashed-myapp:/var/log/myapp ./log-backup/

# Step 3: Now it is safe to remove
docker rm crashed-myapp
```

---

## Summary

- `docker logs` is your first tool when debugging — use `-f` to follow live, `--tail` for recent lines, and `--since` for time-based filtering
- `docker exec -it CONTAINER bash` opens an interactive shell inside a running container — this is your window into the container's world
- `docker inspect --format` extracts specific fields like IP address, exit code, OOM kill status, and restart count
- `docker stats` shows live CPU, memory, and network usage per container — use it to find resource hogs and set appropriate limits
- `docker top` shows processes inside a container without needing to exec into it
- `docker cp` copies files between host and container — invaluable for recovering data from crashed containers
- `docker diff` shows what files the container has written or modified compared to its base image

In the next chapter, we move from containers to images — understanding what Docker images are, how they are built, and how they work under the hood.
