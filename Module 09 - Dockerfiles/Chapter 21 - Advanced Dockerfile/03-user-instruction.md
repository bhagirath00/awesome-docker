# The USER Instruction

## Introduction

By default, processes inside Docker containers run as `root` (UID 0). While container isolation prevents root inside the container from having root on the host directly, running as root inside a container is still a significant security risk. If an attacker exploits a vulnerability in your application and gains arbitrary code execution, they have full root access inside the container — which gives them access to all files, all processes, and many potential host escape vectors.

The `USER` instruction is the primary defense. It tells Docker to run all subsequent `RUN`, `CMD`, and `ENTRYPOINT` instructions as a specific non-root user. This is not optional in production — running as root in production containers is a security policy violation at most serious organizations.

---

## Why Non-Root Matters

Running as root gives the container process:
- Ability to read and modify any file in the container filesystem
- Ability to bind to privileged ports (below 1024)
- Many Linux capabilities by default (e.g., `CAP_NET_BIND_SERVICE`, `CAP_SYS_PTRACE`)
- Higher chance of successful container escape via kernel exploits

Running as a non-root user:
- Limits damage from exploitation — the attacker can only access files the user owns
- Prevents binding to privileged ports (forcing legitimate configs to use 8080 instead of 80)
- Removes capabilities that are rarely needed by applications
- Satisfies security scanners, compliance frameworks (PCI-DSS, SOC 2), and AWS ECR image policies
- Works with Kubernetes PSP/PSA (PodSecurityPolicy/PodSecurityAdmission) that enforces non-root

---

## Creating and Switching to a Non-Root User

The standard pattern: create a dedicated system user and switch to it near the end of the Dockerfile.

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies as root (required to install packages)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY src/ .

# Create a dedicated, non-root system user
# --system: creates a system account (no home directory by default, no shell)
# --group: creates a matching group
# --no-create-home: no home directory
# The user has no password and cannot log in interactively
RUN groupadd --system appuser && \
    useradd --system --no-create-home --gid appuser appuser

# Change ownership of the app directory to the non-root user
RUN chown -R appuser:appuser /app

# Switch to the non-root user
USER appuser

EXPOSE 8080
CMD ["gunicorn", "--workers", "4", "--bind", "0.0.0.0:8080", "app:app"]
```

From the `USER appuser` line onward, all subsequent commands — including the final `CMD` — run as `appuser`.

---

## Specifying User by UID

For maximum portability (especially in Kubernetes and OpenShift environments that may require specific UIDs), specify the user by numeric UID instead of username:

```dockerfile
RUN groupadd --gid 1001 appuser && \
    useradd --uid 1001 --gid 1001 --no-create-home --shell /bin/false appuser

USER 1001
```

Or directly without creating a named user (minimal approach):

```dockerfile
# Switch to UID 1001 — does not require the user to exist in /etc/passwd
USER 1001:1001
```

The benefit of using a UID: it works even in environments that do not have `/etc/passwd` (distroless images) and is more explicit for security policy enforcement.

---

## File Ownership — The Critical Step

Switching to `USER appuser` does not automatically grant access to files. If you copy files as root and then switch users, the application cannot write to them.

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Wrong — files copied as root, then switching user
# Application will get "Permission denied" when trying to write logs or temp files
COPY src/ .
RUN useradd --system --no-create-home appuser
USER appuser
```

The correct pattern — copy files then immediately `chown`:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Create user early so chown works
RUN groupadd --system appuser && \
    useradd --system --no-create-home --gid appuser appuser

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY src/ .

# Set ownership before switching user
RUN chown -R appuser:appuser /app

USER appuser

EXPOSE 8080
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "app:app"]
```

### The COPY --chown Shortcut

Modern Docker supports `--chown` on `COPY` and `ADD` instructions, avoiding the separate `chown` layer:

```dockerfile
FROM python:3.11-slim

RUN groupadd --system appuser && \
    useradd --system --no-create-home --gid appuser appuser

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy with correct ownership in one step (no separate RUN chown needed)
COPY --chown=appuser:appuser src/ .

USER appuser
EXPOSE 8080
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "app:app"]
```

`COPY --chown` is more efficient because it sets ownership during the copy, avoiding an extra layer.

---

## Handling Privileged Ports

Non-root users cannot bind to ports below 1024 (HTTP: 80, HTTPS: 443, SMTP: 25, etc.) on Linux by default. This is a feature, not a bug — it prevents unauthorized processes from hijacking standard ports.

Solutions:

**Option 1: Use a high port inside the container (recommended)**

```dockerfile
# Application runs on 8080, a reverse proxy handles 80/443 outside
EXPOSE 8080
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "app:app"]
```

This is the correct production pattern. An Nginx container or an AWS ALB handles the public-facing port 80/443 and proxies to 8080.

**Option 2: Grant CAP_NET_BIND_SERVICE capability**

```bash
# Allow binding to privileged ports without being root
docker run --cap-add NET_BIND_SERVICE myapp:1.0
```

Or in Docker Compose:

```yaml
services:
  api:
    cap_add:
      - NET_BIND_SERVICE
```

This grants only the specific capability needed, rather than running the entire process as root. Still prefer the high-port approach.

---

## Non-Root Users in Base Images

Some base images already create and switch to a non-root user. If the image you are extending already has a user set, switching back to root for installation steps and then back to the non-root user is required:

```dockerfile
# node:20-alpine runs as root by default, but has a built-in "node" user
FROM node:20-alpine

WORKDIR /app

# Install as root
COPY package.json package-lock.json ./
RUN npm ci --only=production

COPY --chown=node:node . .

# Switch to the built-in node user
USER node

EXPOSE 3000
CMD ["node", "server.js"]
```

```dockerfile
# nginx creates a non-root "nginx" user but the main process still runs as root
# for configuration. Override with a custom user for the worker process:
FROM nginx:1.25-alpine

# The main nginx process (master) must run as root to bind port 80
# But worker processes run as the "nginx" user
# This is a case where rootless nginx requires a different base image
```

For truly rootless Nginx, use `nginxinc/nginx-unprivileged`:

```dockerfile
FROM nginxinc/nginx-unprivileged:1.25-alpine
# Already configured to run on port 8080 as a non-root user
COPY --chown=nginx:nginx ./site-files/ /usr/share/nginx/html/
```

---

## Verifying the User in Running Containers

```bash
# Confirm the container is running as non-root
docker run myapp:1.0 whoami
# appuser

docker run myapp:1.0 id
# uid=1001(appuser) gid=1001(appuser) groups=1001(appuser)

# Check the user of a running container
docker inspect myapp_api_1 --format '{{.Config.User}}'
# appuser  (or 1001:1001)
```

---

## AWS ECS and the Non-Root User

In AWS ECS task definitions, you can enforce non-root at the task level:

```json
{
  "containerDefinitions": [
    {
      "name": "api",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2",
      "user": "1001:1001"
    }
  ]
}
```

ECS also supports `readonlyRootFilesystem` to pair with non-root for maximum security:

```json
{
  "containerDefinitions": [
    {
      "name": "api",
      "user": "1001:1001",
      "readonlyRootFilesystem": true,
      "mountPoints": [
        {
          "containerPath": "/tmp",
          "sourceVolume": "tmp-volume"
        }
      ]
    }
  ]
}
```

---

## Summary

- By default, containers run as root — this is a significant security risk in production
- Use `USER appuser` in your Dockerfile to switch to a non-root user before the final `CMD` or `ENTRYPOINT`
- Create the user with `useradd --system --no-create-home` to create a locked-down system account
- Use `COPY --chown=appuser:appuser` to set correct file ownership during the copy step — avoids a separate `RUN chown` layer
- Specify users by UID (`USER 1001:1001`) for compatibility with distroless images and Kubernetes security policies
- Non-root users cannot bind to ports below 1024 — use high ports (8080) inside the container and let a reverse proxy or ALB handle 80/443
- Verify with `docker run myapp whoami` and `docker inspect` to confirm the container is not running as root

In the next file, we cover the `HEALTHCHECK` instruction in Dockerfiles — baking health monitoring directly into the image.
