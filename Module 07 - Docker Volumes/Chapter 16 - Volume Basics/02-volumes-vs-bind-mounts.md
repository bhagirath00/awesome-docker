# Volumes vs Bind Mounts

## Introduction

Docker provides two primary mechanisms for mounting host storage into containers: **volumes** and **bind mounts**. Both make data available inside a container from outside the container's ephemeral writable layer, but they work differently, behave differently, and are appropriate for different use cases.

Choosing the wrong one causes real problems — using bind mounts in production for database data leads to permission issues, performance degradation, and data loss during deployments. Using volumes for development leads to stale code that does not reflect your edits. Understanding this distinction is not academic — it is a practical decision you make every time you run a container.

---

## The Fundamental Difference

| | **Volume** | **Bind Mount** |
|---|---|---|
| **What is mounted** | A Docker-managed directory | A specific host filesystem path |
| **Who controls the path** | Docker | You (the operator) |
| **Host path** | `/var/lib/docker/volumes/name/_data` | Any absolute path you specify |
| **Pre-existing host data** | Empty on creation (Docker seeds from image) | Shows existing host files immediately |
| **Created if missing** | Yes, automatically | No, container fails if path does not exist |
| **Portable** | Yes | No (depends on specific host paths) |

```bash
# Volume — Docker manages the path
docker run -v pgdata:/var/lib/postgresql/data postgres:16

# Bind mount — you control the path
docker run -v /home/user/myapp:/app myapp:1.0
```

---

## Behavior Comparison: Initial Content

This is where volumes and bind mounts differ most critically.

### Volume: Seeds from Image

When an empty volume is mounted to a path that has content in the image, Docker copies the image content into the volume. The container starts with the image's data.

```bash
# Create a fresh empty volume
docker volume create fresh-vol

# Mount it to /data, which has content in the alpine image
docker run -v fresh-vol:/data alpine ls /data
# → alpine's default /data content is copied in, appears in the volume
```

For a database like PostgreSQL:
1. First run with a new volume → PostgreSQL initializes its data directory into the volume
2. Subsequent runs → PostgreSQL finds its already-initialized data directory

### Bind Mount: Shows Host Content (Even if Empty)

A bind mount always shows exactly what the host directory contains — even if it is completely empty, which hides any content the image had at that path.

```bash
mkdir /tmp/empty-dir

# Mount the empty host directory to /usr/share/nginx/html
docker run -v /tmp/empty-dir:/usr/share/nginx/html nginx

# Result: nginx has no HTML files. The empty host directory hides
# the nginx default content that was in the image.
```

This is why the Node.js development pattern uses a separate anonymous volume for `node_modules`:

```bash
docker run -d \
  -v $(pwd):/app \           # bind mount source code
  -v /app/node_modules \     # anonymous volume shields node_modules
  node:20-alpine \
  npm start
```

Without `-v /app/node_modules`, the bind mount of `$(pwd)` would cover `/app` entirely — hiding the `node_modules` installed inside the image with whatever (possibly empty) `node_modules` exists on your host.

---

## Performance Comparison

### Linux Hosts

On a Linux host (including Linux EC2 instances), both volumes and bind mounts use the native filesystem directly. Performance is comparable, though volumes have a slight edge because they bypass the OverlayFS layer.

For database workloads on Linux, the difference is measurable. PostgreSQL on a named volume is consistently faster than PostgreSQL writing to the container's writable layer (which uses OverlayFS).

### macOS and Windows Hosts

On macOS and Windows, Docker runs inside a Linux VM. Every read and write to a bind mount must cross the VM boundary — from the host's macOS/Windows filesystem into the Linux VM.

This VM filesystem bridge is **significantly slower**. A Node.js application with its `node_modules` on a bind mount on macOS can run noticeably slower than on a native Linux host.

Named volumes are stored inside the Linux VM and do not cross this boundary. They perform at native Linux speeds.

**Practical implication:** In Docker Desktop on macOS, mount your source code with a bind mount (you need host-side edits to appear in the container) but ensure any high-I/O directories like `node_modules`, build outputs, or caches are on named volumes.

---

## Side-by-Side Use Cases

### Development — Use Bind Mounts for Code

During development, you need edits on your host machine to appear immediately in the container. This requires a bind mount.

```bash
# Development: mount source code from host
docker run -d \
  --name api-dev \
  -p 5000:5000 \
  -v $(pwd):/app \              # bind mount — source code edits appear immediately
  -v /app/__pycache__ \         # volume — keep compiled files inside container
  -e FLASK_DEBUG=1 \
  python:3.11-slim \
  python app.py
```

The bind mount lets you edit `app.py` in VS Code on your Mac and see Flask reload immediately.

### Production — Use Volumes for Data

In production, containers are immutable. You never edit code inside a running container. The application code is baked into the image via `COPY` in the Dockerfile. But the data the application generates must persist.

```bash
# Production: named volumes for persistent data
docker run -d \
  --name api-prod \
  --restart unless-stopped \
  -p 8080:8080 \
  -v api-uploads:/app/uploads \     # volume — user uploads persist across deployments
  -e APP_ENV=production \
  myapp:1.4.2                       # code is in the image, not mounted
```

When you deploy `myapp:1.4.3`, you create a new container with the same volume:

```bash
docker stop api-prod && docker rm api-prod
docker run -d --name api-prod \
  -v api-uploads:/app/uploads \
  myapp:1.4.3                       # new image version
```

The `api-uploads` volume carries all user uploads from the previous container into the new one.

### Configuration Files — Use Bind Mounts (Read-Only)

For injecting environment-specific configuration files, bind mounts are appropriate. The config lives on the host (or is generated by a configuration management system), and the container reads it.

```bash
docker run -d \
  --name nginx \
  -p 80:80 \
  -v /etc/myapp/nginx.conf:/etc/nginx/nginx.conf:ro \   # bind mount, read-only
  -v /etc/ssl/certs:/etc/ssl/certs:ro \                  # bind mount, read-only
  nginx:1.25
```

The `:ro` suffix prevents the container from modifying these files.

---

## When to Choose What — Decision Guide

```
Is this data generated at runtime that must persist across container restarts?
├── YES → Use a named volume
│     (databases, user uploads, application state)
└── NO → Continue...

Is this code or configuration you need to edit on the host machine?
├── YES → Use a bind mount
│     (source code in development, config files)
└── NO → Continue...

Is this sensitive temporary data that should never touch disk?
├── YES → Use tmpfs
└── NO → Let the container's writable layer handle it
```

---

## Feature Comparison Table

| Feature | Named Volume | Bind Mount | Anonymous Volume |
|---|---|---|---|
| **Persists after `docker rm`** | Yes | Yes (host files remain) | No (Docker may clean up) |
| **Host path control** | No | Yes | No |
| **Works without host path existing** | Yes (Docker creates) | No (container fails) | Yes |
| **Seeds from image on first use** | Yes | No | Yes |
| **Performance on macOS** | Native (in VM) | Slow (crosses VM boundary) | Native (in VM) |
| **Portable between hosts** | Yes | No (path-specific) | No |
| **Best for production data** | ✅ | ❌ | ❌ |
| **Best for dev code mounting** | ❌ | ✅ | ❌ |
| **Best for read-only config** | — | ✅ | — |

---

## Summary

- Volumes are managed by Docker; bind mounts use a specific host path you control — this is the core distinction
- Empty volumes seeded to a container path that has content in the image will be populated automatically; bind mounts always show the raw host directory content, even if empty
- On macOS and Windows, bind mounts cross a VM filesystem boundary and are significantly slower for high-I/O operations like npm install or heavy database writes; named volumes run at native speed
- Use named volumes for production persistent data: databases, user-generated uploads, application state
- Use bind mounts in development for source code (so host edits appear immediately in the container) and for injecting read-only config files
- The most common development pattern combines both: a bind mount for source code and an anonymous volume to shield directories like `node_modules` that must stay container-internal

In the next file, we cover the complete set of volume management commands — listing, inspecting, pruning, and understanding what is on disk.
