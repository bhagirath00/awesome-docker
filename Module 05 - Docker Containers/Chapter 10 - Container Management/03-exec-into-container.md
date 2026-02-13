# Exec into Container

## Introduction

`docker exec` lets you run commands inside a container that is already running. It is your most powerful tool for debugging, inspecting state, running maintenance tasks, and understanding what is happening inside a container without stopping it.

Knowing how to use `exec` effectively is the difference between spending 5 minutes diagnosing a problem and spending an hour guessing.

---

## The Basic Syntax

```bash
docker exec [OPTIONS] CONTAINER COMMAND [ARGS]
```

The container must be running — `exec` cannot be used on stopped containers. To debug a stopped container, use `docker start` first, or inspect its logs and filesystem with `docker logs` and `docker cp`.

---

## Opening an Interactive Shell

The most common use of `exec` is getting a shell inside a running container:

```bash
# Open bash in a running container
docker exec -it webserver bash

# Use sh if bash is not available (Alpine images, distroless variants)
docker exec -it alpine-container sh

# You are now inside the container
# Prompt changes to show you are in the container:
root@a1b2c3d4ef56:/app#
```

The flags:
- `-i` — interactive (keeps stdin open so you can type)
- `-t` — allocate a pseudo-TTY (gives you a proper terminal with line editing)

Both flags together (`-it`) give you a full interactive terminal experience.

### What You Can Do Inside the Shell

Once inside the container:

```bash
# Check which user you are running as
whoami
# root

# Check running processes
ps aux

# Check environment variables
env | sort

# Check network configuration
ip addr
netstat -tlnp

# Check disk usage
df -h

# Check which files exist
ls -la /app/

# Check file contents
cat /app/config.yml

# Test network connectivity to other services
curl http://database:5432
ping database

# Check listening ports
ss -tlnp
```

### Exit the Shell

```bash
# Type exit to leave the container shell
exit

# Or press Ctrl+D
# The container continues running — you only exited the exec session
```

---

## Running Specific Commands

You do not need to open a full shell. You can run any specific command directly:

```bash
# List files in the container
docker exec webserver ls -la /etc/nginx/

# Read a config file
docker exec webserver cat /etc/nginx/nginx.conf

# Check running processes without entering the container
docker exec api ps aux

# Check which user the main process runs as
docker exec api whoami

# Test database connectivity
docker exec api python -c "
import psycopg2
conn = psycopg2.connect('postgresql://user:pass@db:5432/myapp')
print('Connected successfully')
conn.close()
"

# Check the Python version in the container
docker exec api python --version

# Run a Django management command
docker exec api python manage.py check

# Show environment variables
docker exec api env | sort

# Check free memory inside the container
docker exec api free -m

# Verify a file exists
docker exec api test -f /app/config.yml && echo "exists" || echo "missing"
```

---

## Running as a Specific User

By default, `exec` runs commands as the user set by the `USER` instruction in the Dockerfile. Sometimes you need to run as root for maintenance tasks:

```bash
# Run as root even though the container normally runs as appuser
docker exec -it --user root api bash

# Run as a specific UID
docker exec -it --user 1001 api bash

# Run as a specific user:group
docker exec -it --user appuser:appgroup api bash
```

**Security note:** In production, avoid using `--user root` unless absolutely necessary. It bypasses the security boundary you set up with the `USER` instruction.

---

## Setting Environment Variables for exec Sessions

You can inject environment variables that only exist in the exec session:

```bash
# Set DEBUG=true just for this exec session
docker exec -it -e DEBUG=true api bash

# Useful for enabling verbose logging temporarily
docker exec -it -e LOG_LEVEL=debug api python manage.py shell
```

These variables only exist in the exec session — they do not affect the running container or persist after the session ends.

---

## Setting the Working Directory

```bash
# Run a command from a specific directory inside the container
docker exec -w /app/data api ls -la

# Open a shell in a specific directory
docker exec -it -w /var/log/nginx webserver bash
```

---

## Practical Debugging Workflows

### Debugging: Application Not Responding

```bash
# Step 1: Confirm the container is running
docker ps | grep api

# Step 2: Check recent logs
docker logs --tail 50 api

# Step 3: Get inside and check
docker exec -it api bash

# Inside the container:
# Is the process running?
ps aux

# Is it listening on the expected port?
ss -tlnp | grep 8080

# Can it reach the database?
curl http://db:5432

# Check environment variables — is DATABASE_URL set?
env | grep DATABASE

# Exit
exit
```

### Debugging: Database Connection Issues

```bash
# Test the connection string manually
docker exec api python -c "
import os
import psycopg2

db_url = os.environ.get('DATABASE_URL')
print(f'Connecting to: {db_url}')

try:
    conn = psycopg2.connect(db_url)
    cursor = conn.cursor()
    cursor.execute('SELECT version()')
    print('Connected:', cursor.fetchone())
    conn.close()
except Exception as e:
    print(f'Connection failed: {e}')
"
```

### Debugging: File Permission Issues

```bash
# Check file ownership and permissions
docker exec api ls -la /app/uploads/

# Check what user the process runs as
docker exec api whoami
docker exec api id

# Check if the app can write to a directory
docker exec api touch /app/uploads/test.txt && echo "writable" || echo "permission denied"
```

### Debugging: High Memory Usage

```bash
# Check memory usage of processes inside the container
docker exec api ps aux --sort=-%mem | head -20

# Check total memory from inside the container
docker exec api free -m

# For Python apps — check object counts
docker exec -it api python -c "
import gc
gc.collect()
objects = {}
for obj in gc.get_objects():
    name = type(obj).__name__
    objects[name] = objects.get(name, 0) + 1
for name, count in sorted(objects.items(), key=lambda x: -x[1])[:20]:
    print(f'{count:10d}  {name}')
"
```

### Running Database Migrations

```bash
# Django
docker exec api python manage.py migrate

# Flask with Flask-Migrate
docker exec api flask db upgrade

# Alembic
docker exec api alembic upgrade head

# Node.js with Sequelize
docker exec api npx sequelize db:migrate

# With interactive output
docker exec -it api python manage.py migrate
```

### Reloading Configuration Without Restarting

Some applications can reload configuration by receiving a signal:

```bash
# Send SIGHUP to nginx to reload config (reads nginx.conf without restart)
docker exec webserver nginx -s reload

# Or send the signal directly to PID 1
docker exec webserver kill -HUP 1
```

---

## exec vs docker run for Debugging

Sometimes you cannot `exec` into a container because:
- The container exited
- The container has no shell (distroless images)
- The container is too broken to exec into

### Debugging Exited Containers

```bash
# Container exited — cannot exec into it
docker exec crashed-app bash
# Error: container is not running

# Read the logs instead
docker logs crashed-app

# Copy files out of the stopped container
docker cp crashed-app:/app/logs ./logs-backup/

# Or start a new container from the same image with an overridden entrypoint
docker run -it --entrypoint bash myapp:1.0
# Now you are in a fresh container from the same image
# Useful for reproducing the failure
```

### Debugging Distroless Images (No Shell)

Distroless images have no shell at all. `exec` does not give you a terminal:

```bash
docker exec -it distroless-app sh
# OCI runtime exec failed: exec failed: unable to start container process:
# exec: "sh": executable file not found in $PATH
```

Solutions:

```bash
# Option 1: Add a debug build target with a shell in your Dockerfile
FROM gcr.io/distroless/python3 AS production
...

FROM python:3.11-slim AS debug  # for debugging only
...

# Build debug image when needed
docker build --target debug -t myapp:debug .
docker exec -it debug-container bash

# Option 2: Use kubectl debug (for Kubernetes) or ECS Exec (for ECS)
# ECS Exec:
aws ecs execute-command \
  --cluster production \
  --task a1b2c3d4-e5f6-7890-abcd-ef1234567890 \
  --container api \
  --interactive \
  --command "/bin/sh"
```

---

## exec in Production — ECS Exec

On AWS ECS, you cannot use `docker exec` directly (ECS manages the containers, not you). Use **ECS Exec** instead:

```bash
# Enable ECS Exec on a service (requires SSM Agent in the container)
aws ecs update-service \
  --cluster production \
  --service myapp \
  --enable-execute-command

# Execute a command in a running ECS task
aws ecs execute-command \
  --cluster production \
  --task a1b2c3d4-e5f6-7890-abcd-ef1234567890 \
  --container api \
  --interactive \
  --command "/bin/bash"
```

ECS Exec uses AWS SSM Session Manager — all sessions are logged to CloudWatch and S3 for auditing. No SSH required, no open ports.

---

## Summary

- `docker exec -it CONTAINER bash` opens an interactive shell inside a running container — your primary debugging tool
- Use `sh` instead of `bash` for Alpine and other minimal images that do not have bash
- `docker exec -it --user root CONTAINER bash` runs the shell as root — useful for maintenance tasks, but avoid in production
- You can inject environment variables into an exec session with `-e` — they do not affect the running container
- `docker exec` only works on running containers — for stopped containers, read logs with `docker logs` or start the container first
- For distroless images without a shell, build a debug variant with a shell for troubleshooting
- On AWS ECS, use `aws ecs execute-command` instead of `docker exec` — it works through AWS SSM and logs all sessions for auditing

In the next chapter, we cover container configuration — how to pass configuration, name containers effectively, map ports, and set restart policies.
