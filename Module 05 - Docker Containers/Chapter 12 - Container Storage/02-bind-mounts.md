# Bind Mounts

## Introduction

A bind mount connects a specific path on the host machine to a path inside a container. Files in that directory are shared directly — the container and the host see the same files in real time.

Bind mounts are the foundation of Docker development workflows. They let you edit code on your host machine in your preferred editor and see those changes reflected immediately inside the running container — without rebuilding the image. They are also used to inject configuration files and to persist data in specific host locations.

---

## How Bind Mounts Work

A bind mount bypasses the container filesystem stack entirely. When you bind mount `/home/user/myapp` to `/app` inside a container:
- The container's `/app` is not in the writable layer
- It is a direct reference to `/home/user/myapp` on the host
- Reads and writes to `/app` inside the container happen directly on the host filesystem
- Changes are visible immediately in both directions — host changes appear in the container, container changes appear on the host

```bash
# Mount current directory into /app inside the container
docker run -v $(pwd):/app python:3.11-slim python /app/app.py

# Edit app.py on the host — the container sees the change immediately
# No image rebuild required
```

---

## Bind Mount Syntax

Docker supports two syntaxes for bind mounts:

### Syntax 1: -v (Short Form)

```bash
docker run -v HOST_PATH:CONTAINER_PATH[:OPTIONS] image
```

```bash
# Mount current directory to /app
docker run -v $(pwd):/app python:3.11-slim

# Mount a specific host path
docker run -v /home/user/myapp:/app python:3.11-slim

# Mount with read-only access
docker run -v $(pwd)/config:/app/config:ro myapp:1.0

# Mount a Windows path (PowerShell)
docker run -v C:\Users\user\myapp:/app python:3.11-slim
```

### Syntax 2: --mount (Explicit Form)

```bash
docker run --mount type=bind,source=HOST_PATH,target=CONTAINER_PATH image
```

```bash
# Same mount as above, explicit syntax
docker run --mount type=bind,source=$(pwd),target=/app python:3.11-slim

# Read-only mount
docker run --mount type=bind,source=$(pwd)/config,target=/app/config,readonly myapp:1.0
```

The `--mount` syntax is more verbose but more explicit and harder to misuse. Both work identically.

---

## Development Workflow — Live Code Reloading

The most common use of bind mounts is mounting source code into a container so changes take effect without rebuilding the image.

### Python Flask Development

```bash
# Run flask in development mode with the source code mounted
docker run -d \
  --name flask-dev \
  -p 5000:5000 \
  -v $(pwd):/app \
  -w /app \
  -e FLASK_ENV=development \
  -e FLASK_DEBUG=1 \
  python:3.11-slim \
  python app.py
```

When Flask is in debug mode, it watches for file changes and reloads automatically. Because the source code is bind-mounted from the host, editing a file on the host triggers the Flask reload — no container restart needed.

### Node.js Development with Nodemon

```bash
# Run Node.js with nodemon for auto-reload
docker run -d \
  --name node-dev \
  -p 3000:3000 \
  -v $(pwd):/app \
  -v /app/node_modules \
  -w /app \
  node:20-alpine \
  npx nodemon src/index.js
```

Note the second `-v /app/node_modules` — this is an anonymous volume that prevents the bind mount from overwriting the `node_modules` installed inside the container with the (possibly non-existent or platform-incompatible) `node_modules` on the host.

### Using Docker Compose for Development

Docker Compose is the standard way to define development environments with bind mounts:

```yaml
# docker-compose.dev.yml
version: "3.8"

services:
  api:
    image: python:3.11-slim
    working_dir: /app
    command: python app.py
    ports:
      - "5000:5000"
    volumes:
      - .:/app           # mount current directory into /app
    environment:
      - FLASK_ENV=development
      - FLASK_DEBUG=1
      - DATABASE_URL=postgresql://user:pass@db:5432/myapp

  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data   # named volume for data persistence
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=myapp

volumes:
  pgdata:
```

```bash
# Start development environment
docker compose -f docker-compose.dev.yml up

# Edit code on the host — changes appear immediately in the container
```

---

## Injecting Configuration Files

Bind mounts are also used to inject configuration files into containers without baking them into the image:

```bash
# Nginx with custom configuration
docker run -d \
  --name webserver \
  -p 80:80 \
  -p 443:443 \
  -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \
  -v $(pwd)/ssl-certs:/etc/ssl/certs:ro \
  nginx:1.25.3
```

The `:ro` suffix makes the bind mount read-only inside the container — the nginx process can read the config but cannot modify it.

```bash
# Prometheus with custom configuration
docker run -d \
  --name prometheus \
  -p 9090:9090 \
  -v $(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml:ro \
  prom/prometheus:v2.48.0

# Grafana with provisioned dashboards
docker run -d \
  --name grafana \
  -p 3000:3000 \
  -v $(pwd)/grafana/dashboards:/etc/grafana/provisioning/dashboards:ro \
  -v $(pwd)/grafana/datasources:/etc/grafana/provisioning/datasources:ro \
  grafana/grafana:10.2.0
```

---

## Persisting Data with Bind Mounts

Bind mounts can persist container-generated data to a specific host location:

```bash
# Persist nginx logs to a host directory
docker run -d \
  --name webserver \
  -p 80:80 \
  -v /var/log/nginx:/var/log/nginx \
  nginx:1.25.3

# Now logs are at /var/log/nginx on the host
# A log rotation daemon on the host can manage them
```

```bash
# Persist application data to a specific host path
docker run -d \
  --name api \
  -v /data/myapp/uploads:/app/uploads \
  myapp:1.0
```

For production data persistence, **named volumes** are generally preferred over bind mounts because Docker manages the storage location. Use bind mounts for persistent data only when you specifically need the data at a known host path — for log aggregation, for backup scripts that expect a specific path, or when integrating with host-based tools.

---

## Read-Only Bind Mounts

Make a bind mount read-only to prevent the container from modifying host files:

```bash
# Container can read config but not modify it
docker run -v $(pwd)/config:/app/config:ro myapp:1.0

# Using --mount syntax
docker run --mount type=bind,source=$(pwd)/config,target=/app/config,readonly myapp:1.0
```

Read-only mounts are a security best practice for configuration files, SSL certificates, and secrets that the container should only read, not modify.

---

## File Permission Issues with Bind Mounts

Bind mounts inherit the permissions of the host filesystem. This can cause permission problems when the container runs as a non-root user.

```bash
# Host file owned by root (UID 0)
ls -la $(pwd)/app.py
# -rw-r--r-- 1 root root 1234 Jan 15 09:23 app.py

# Container runs as UID 1001
docker run -v $(pwd):/app --user 1001 python:3.11-slim python /app/app.py
# Permission denied — UID 1001 cannot write to files owned by root
```

Solutions:

```bash
# Option 1: Change host file ownership to match container user
sudo chown -R 1001:1001 $(pwd)

# Option 2: Make files world-writable (less secure)
chmod -R o+w $(pwd)

# Option 3: Run the container as the host user
docker run -v $(pwd):/app --user $(id -u):$(id -g) python:3.11-slim python /app/app.py
```

---

## Bind Mounts vs Named Volumes

| Feature | Bind Mount | Named Volume |
|---|---|---|
| Host path control | Yes — you specify exact path | No — Docker chooses the path |
| Visible to host | Yes | Requires `docker volume inspect` |
| Works on all OS | Linux, macOS (via VM), Windows (via WSL2) | Yes |
| Performance on macOS | Slower (VM filesystem bridge) | Faster |
| Data persistence after `docker rm` | Yes — host files remain | Yes — volume remains |
| Best for | Development, config injection | Production data, databases |

For **development**: use bind mounts to mount source code — you need host-side edits to reflect immediately.

For **production data** (databases, uploaded files): use named volumes — Docker manages storage, performance is better, and there is no dependency on a specific host path.

---

## Bind Mounts in Production

Bind mounts are primarily a development and configuration tool. In production, they are used for:

1. **Configuration files** — inject environment-specific config without baking it into the image
2. **SSL certificates** — mount certificates managed by certbot or ACM
3. **Log output** — route container logs to a host path for aggregation
4. **Shared secrets** — in environments where Secrets Manager is not available

In AWS ECS/Fargate, bind mounts work differently because there is no persistent host filesystem. ECS supports:
- **EFS volumes** — mount AWS Elastic File System into containers for shared, persistent storage
- **Ephemeral storage** — temporary storage allocated per task, not persistent

```json
{
  "volumes": [
    {
      "name": "app-config",
      "efsVolumeConfiguration": {
        "fileSystemId": "fs-12345678",
        "rootDirectory": "/config/myapp"
      }
    }
  ],
  "containerDefinitions": [
    {
      "name": "api",
      "mountPoints": [
        {
          "sourceVolume": "app-config",
          "containerPath": "/app/config",
          "readOnly": true
        }
      ]
    }
  ]
}
```

---

## Summary

- A bind mount directly maps a host path into a container — reads and writes are immediate in both directions
- The primary use case is development: mount source code into a container so code changes take effect without rebuilding the image
- Use `-v HOST_PATH:CONTAINER_PATH` or `--mount type=bind,source=...,target=...` syntax
- Add `:ro` or `readonly` to make the mount read-only inside the container — important for configuration files and secrets
- File permission mismatches happen when the host files are owned by a different UID than the container runs as — use `--user $(id -u):$(id -g)` to run the container as your host user
- For production persistent data (databases, uploads), prefer named volumes — Docker manages storage and performance is better, especially on macOS
- On AWS ECS/Fargate, use EFS volumes for shared persistent storage instead of bind mounts

In the next file, we cover tmpfs mounts — in-memory filesystems for fast, secure temporary storage inside containers.
