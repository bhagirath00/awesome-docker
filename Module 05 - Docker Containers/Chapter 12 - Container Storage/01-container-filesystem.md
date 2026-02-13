# Container Filesystem

## Introduction

Every container has its own filesystem. This filesystem is assembled from the image's read-only layers with a thin writable layer added on top. Understanding how this filesystem works — what is persistent, what is temporary, and where data actually lives — is essential for designing containerized applications that behave correctly.

This file explains the container filesystem in depth, how the writable layer works, what happens to data when containers are removed, and what this means for your application design.

---

## The Container Filesystem Stack

When Docker creates a container from an image, it assembles a filesystem from multiple layers:

```
┌──────────────────────────────────────────┐
│  Container Writable Layer (thin, empty)  │  ← writes go here
├──────────────────────────────────────────┤
│  Image Layer 4: COPY . /app              │  ← read-only
├──────────────────────────────────────────┤
│  Image Layer 3: RUN pip install          │  ← read-only
├──────────────────────────────────────────┤
│  Image Layer 2: COPY requirements.txt    │  ← read-only
├──────────────────────────────────────────┤
│  Image Layer 1: FROM python:3.11-slim    │  ← read-only
└──────────────────────────────────────────┘
```

All the image layers are **read-only** and **shared** between containers. The writable layer belongs to that specific container and starts completely empty.

Docker uses **OverlayFS** (overlay filesystem) to present all these layers as a single unified filesystem to the container. From inside the container, you see one filesystem — you cannot tell which files come from which layer.

---

## The Writable Layer — Copy-on-Write

When a container needs to modify a file that exists in a read-only image layer, Docker uses **copy-on-write (CoW)**:

1. Docker finds the file in the image layers
2. Docker copies the file to the container's writable layer
3. Docker modifies the copy in the writable layer
4. The original in the image layer is untouched

This means:
- Reading a file that has not been modified is fast — read directly from the image layer
- First modification of a file is slower — involves a copy operation
- After the first write, subsequent writes are fast — the file is in the writable layer

### Seeing Copy-on-Write in Action

```bash
# Start a container
docker run -it --name test ubuntu:22.04 bash

# Inside the container — modify a file that exists in the image
echo "modified" >> /etc/hosts

# Exit without stopping
# Ctrl+P Ctrl+Q

# Check what files the container changed
docker diff test

# Output:
C /etc
C /etc/hosts    ← Changed — was in image, now in writable layer
```

---

## What Data Lives in the Writable Layer

The writable layer accumulates all changes the container makes to the filesystem:

- Files created by the running application (uploads, caches, temp files)
- Log files written inside the container
- Configuration files modified at runtime
- Compiled `.pyc` files, `__pycache__`, etc.
- Database files if a database container writes to a non-volume path

### The Critical Problem: Writable Layer is Ephemeral

When a container is removed, its writable layer is deleted. All data in the writable layer is gone.

```bash
# Start a container and write a file
docker run -it --name test ubuntu bash

# Inside container:
echo "important data" > /tmp/myfile.txt
cat /tmp/myfile.txt
# important data
exit

# The container is stopped but still exists
docker ps -a | grep test
# test   Exited (0) 5 seconds ago

# Start it again — data is still there (container was stopped, not removed)
docker start -ai test
cat /tmp/myfile.txt
# important data
exit

# Remove the container — now the data is gone
docker rm test

# Start a fresh container — the file is gone
docker run -it --name test ubuntu bash
cat /tmp/myfile.txt
# cat: /tmp/myfile.txt: No such file or directory
```

**Key distinction:**
- `docker stop` → container paused, writable layer preserved
- `docker rm` → container deleted, writable layer deleted

---

## The Size of the Writable Layer

The writable layer grows as the container writes data. You can see the size:

```bash
# Show container disk usage
docker ps -s

# Output:
CONTAINER   SIZE               VIRTUAL SIZE
test        1.23MB (writable)  72.8MB (image + writable)
```

- **SIZE** — the size of the writable layer (what the container added)
- **VIRTUAL SIZE** — total size of all layers (image + writable)

Containers that write a lot to their writable layer — like database containers without volumes — can grow very large. This is another reason to use volumes for data that grows.

---

## Inspecting the Container Filesystem

### docker diff — See What Changed

```bash
docker diff container_name

# Prefixes:
# A = Added
# C = Changed
# D = Deleted
```

```bash
# Example: what did a Python app change in the filesystem?
docker run -d --name api myapp:1.0
sleep 5  # let it start

docker diff api

# Output:
C /tmp
A /tmp/gunicorn.pid         ← gunicorn created a PID file
C /app
A /app/logs
A /app/logs/access.log      ← log files created
A /app/logs/error.log
```

### docker exec — Browse the Filesystem

```bash
# See what files exist inside the container
docker exec container_name ls -la /app/

# Find recently created files
docker exec container_name find /tmp -newer /app -type f

# Check disk usage inside the container
docker exec container_name df -h

# Check size of specific directories
docker exec container_name du -sh /app/ /tmp/ /var/log/
```

### docker cp — Copy Files Between Host and Container

```bash
# Copy a file from inside the container to the host
docker cp container_name:/app/logs/error.log ./error.log

# Copy a directory out of the container
docker cp container_name:/app/uploads ./uploads-backup/

# Copy a file from the host into the container
docker cp ./updated-config.yml container_name:/app/config.yml
```

`docker cp` works on both running and stopped containers, which makes it invaluable for recovering data from a crashed container before removing it.

---

## What Should NOT Live in the Writable Layer

### Data that must persist across container restarts

If you store application data in the writable layer:
- It survives `docker stop` / `docker start` (container preserved)
- It is lost on `docker rm` (container removed)
- It is lost on redeployment (new container is created)

Any data that must outlive the container must be stored outside the writable layer — in a volume or bind mount.

```bash
# Database data should NEVER be in the writable layer
# BAD — postgres data lost when container is removed
docker run -d postgres:16

# GOOD — postgres data stored in a named volume
docker run -d -v pgdata:/var/lib/postgresql/data postgres:16
```

### Shared data between containers

Two containers cannot share the same writable layer. If Container A and Container B both need access to the same files, those files must be in a shared volume, not in either container's writable layer.

---

## Designing for Ephemeral Containers

The best containerized applications treat containers as **ephemeral** — temporary and replaceable. This design principle brings significant benefits:

- Deployments are clean replacements, not updates to running containers
- Rollbacks are instant — just run the old image
- Scaling is simple — start more containers, they start clean every time
- No "snowflake" containers that have been modified from their original state

### The Stateless Application Pattern

Design your application so all important state lives outside the container:

```
┌─────────────────────────────────────────────────────┐
│ Container (ephemeral — can be removed anytime)       │
│   - Application code                                  │
│   - Runtime (Python, Node.js, etc.)                  │
│   - In-memory computation                            │
│                                                       │
│   Does NOT store:                                     │
│   - User uploads (→ S3)                              │
│   - Database records (→ RDS, DynamoDB)               │
│   - Session data (→ ElastiCache Redis)               │
│   - Application logs (→ CloudWatch via log driver)   │
└─────────────────────────────────────────────────────┘
```

A truly stateless application container can be stopped, removed, and replaced with a new container at any time — with zero data loss — because all persistent state is in external services.

---

## tmpfs — In-Memory Filesystem

For temporary data that must be fast and must not be persisted to disk — scratch space, session data, API keys loaded at startup — you can mount a **tmpfs**. This is a filesystem that lives in the host's RAM, not on disk:

```bash
# Mount a tmpfs at /tmp inside the container
docker run -d \
  --tmpfs /tmp:rw,size=512m \
  --name api \
  myapp:1.0

# Everything written to /tmp lives in RAM — not on disk
# It is cleared when the container stops
```

tmpfs is covered in detail in the next file. For now, understand that it gives you a fast, temporary, in-memory filesystem for sensitive or high-speed temporary data.

---

## Filesystem Summary — Three Types of Storage

| Storage Type | Where it lives | Persists after rm? | Shared between containers? |
|---|---|---|---|
| **Writable layer** | Host disk, managed by Docker | No | No |
| **Bind mount** | Host filesystem path | Yes | Yes (if same path) |
| **Named volume** | Host disk, managed by Docker | Yes | Yes (if same volume) |
| **tmpfs** | Host RAM | No | No |

Choose based on your needs:
- **Writable layer** — automatically used for all container filesystem changes you do not route elsewhere
- **Bind mount** — development (mount source code into container), config files
- **Named volume** — production persistent data (databases, uploads)
- **tmpfs** — sensitive temporary data, high-speed scratch space

---

## Summary

- The container filesystem is assembled from the image's read-only layers plus a thin writable layer that starts empty
- OverlayFS presents all layers as one unified filesystem from inside the container
- Copy-on-write means Docker copies a file to the writable layer the first time the container modifies it — the image layers are never changed
- The writable layer is ephemeral — `docker stop` preserves it, but `docker rm` deletes it permanently
- Use `docker diff` to see what files a container changed compared to its image
- Use `docker cp` to recover files from stopped containers before removing them
- Any data that must outlive the container must live in a volume, bind mount, or external service (S3, RDS, Redis) — not in the writable layer
- Design applications to be stateless — treat containers as ephemeral and store all persistent state externally

In the next file, we cover bind mounts — mounting host directories directly into containers, the primary storage mechanism for development workflows.
