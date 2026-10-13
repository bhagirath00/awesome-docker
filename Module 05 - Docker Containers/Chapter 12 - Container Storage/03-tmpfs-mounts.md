# tmpfs Mounts

## Introduction

A tmpfs mount creates an in-memory filesystem inside a container. Data written to a tmpfs mount lives in the host's RAM, not on disk. When the container stops, the data is gone — by design.

tmpfs mounts exist to solve two specific problems: performance and security for temporary data. Understanding when to use them — and when not to — helps you build faster and more secure containers.

---

## What is tmpfs?

`tmpfs` is a Linux filesystem type that stores data entirely in memory (RAM) rather than on a disk-backed filesystem. It is not unique to Docker — Linux systems have used tmpfs for `/tmp` and shared memory (`/dev/shm`) for decades.

Docker exposes tmpfs mounts as a way to give containers a fast, secure, temporary scratch space.

Key characteristics:
- **In-memory** — reads and writes happen at RAM speed, much faster than disk
- **Ephemeral** — data is lost when the container stops or the tmpfs is unmounted
- **Not shared** — tmpfs data is not persisted to disk, cannot be shared between containers, and is not visible on the host filesystem
- **Size-limited** — you define a maximum size; the kernel enforces it

---

## When to Use tmpfs

### Use Case 1: Sensitive Temporary Data

If your application decrypts secrets, generates temporary tokens, or processes sensitive data, you do not want that data written to disk — even temporarily. On disk, files can be recovered from disk caches, swap files, or storage snapshots. Data in tmpfs exists only in RAM and is discarded cleanly when the container stops.

```bash
# Store decrypted secrets in memory, not disk
docker run -d \
  --name api \
  --tmpfs /run/secrets:rw,noexec,nosuid,size=64m \
  myapp:1.0
```

Inside the application:
```python
# Write decrypted secrets to tmpfs — never touches disk
import os

secrets_path = "/run/secrets"
os.makedirs(secrets_path, exist_ok=True)

with open(f"{secrets_path}/db_password", "w") as f:
    f.write(decrypted_password)

# When container stops, /run/secrets is gone from memory
```

### Use Case 2: High-Speed Scratch Space

Some workloads require fast intermediate storage — image processing, video encoding, data transformation pipelines. Disk I/O is a bottleneck for these tasks. A tmpfs mount removes the disk bottleneck entirely.

```bash
# Image processing with fast scratch space
docker run -d \
  --name image-processor \
  --tmpfs /tmp:rw,size=2g \
  image-processor:1.0
```

The processor writes intermediate files to `/tmp` in memory. For large batches of small images, this can be significantly faster than writing to an SSD.

### Use Case 3: Applications with /tmp Writes

Many applications write temporary files to `/tmp`. If your container runs in a read-only root filesystem (a security best practice), `/tmp` must be a writable tmpfs:

```bash
# Read-only root filesystem with writable tmpfs for /tmp
docker run -d \
  --name api \
  --read-only \
  --tmpfs /tmp:rw,size=256m \
  --tmpfs /run:rw,size=64m \
  myapp:1.0
```

This combination gives you a security-hardened container where the application code filesystem is read-only, but the application still has writable scratch space.

---

## tmpfs Syntax

### Using --tmpfs

```bash
# Simple tmpfs mount at /tmp
docker run --tmpfs /tmp myapp:1.0

# With options: size, noexec, nosuid
docker run --tmpfs /tmp:rw,size=256m,noexec,nosuid myapp:1.0

# Multiple tmpfs mounts
docker run \
  --tmpfs /tmp:rw,size=512m \
  --tmpfs /run:rw,size=64m \
  --tmpfs /run/secrets:rw,size=32m,noexec,nosuid \
  myapp:1.0
```

### Using --mount

```bash
# tmpfs with --mount syntax
docker run --mount type=tmpfs,target=/tmp,tmpfs-size=256m myapp:1.0

# With additional options
docker run \
  --mount type=tmpfs,target=/tmp,tmpfs-size=512m,tmpfs-mode=1777 \
  myapp:1.0
```

---

## tmpfs Options

All options are comma-separated after the colon in `--tmpfs /path:OPTIONS`.

| Option | Description |
|---|---|
| `rw` | Read-write (default) |
| `ro` | Read-only (rarely useful for tmpfs) |
| `size=N` | Maximum size in bytes. Supports `k`, `m`, `g` suffixes |
| `noexec` | Prevents execution of binaries from this mount |
| `nosuid` | Prevents setuid/setgid bits from taking effect |
| `mode=OCTAL` | Permissions of the root directory (e.g., `mode=1777`) |

### Security-Hardened tmpfs

For sensitive paths, always add `noexec` and `nosuid`:

```bash
docker run \
  --tmpfs /run/secrets:rw,noexec,nosuid,size=32m \
  myapp:1.0
```

- `noexec` — attackers cannot upload a malicious binary to tmpfs and execute it
- `nosuid` — prevents privilege escalation through setuid binaries placed in tmpfs

### Sizing tmpfs

Set the size to match what the application needs. Too small causes write failures. Too large wastes RAM.

```bash
# For application /tmp (scratch files, small temporaries)
--tmpfs /tmp:rw,size=256m

# For large file processing
--tmpfs /tmp:rw,size=2g

# For secrets/tokens (tiny files)
--tmpfs /run/secrets:rw,noexec,nosuid,size=32m

# For application run directory (PID files, sockets)
--tmpfs /run:rw,size=64m
```

The kernel enforces the size limit. If the application tries to write more than the limit, it gets a "No space left on device" error.

---

## Read-Only Root Filesystem + tmpfs

The most secure container configuration uses a read-only root filesystem. The application cannot modify its own files — which limits the damage an attacker can do even if they exploit a vulnerability.

```bash
# Read-only root filesystem
docker run -d \
  --name api \
  --read-only \
  --tmpfs /tmp:rw,size=256m,noexec,nosuid \
  --tmpfs /run:rw,size=64m \
  --tmpfs /var/log:rw,size=128m \
  -p 8080:8080 \
  myapp:1.0
```

Test which paths your application needs to write:

```bash
# Run with --read-only and watch for permission errors
docker run --read-only myapp:1.0
# open /tmp/gunicorn.pid: read-only file system
# → add --tmpfs /tmp

# open /var/run/app.sock: read-only file system
# → add --tmpfs /run

# Keep adding tmpfs until application starts cleanly
```

In a Dockerfile, declare tmpfs paths your application uses:

```dockerfile
# Document what tmpfs mounts the container needs
# (This does not create tmpfs — just documents it)
VOLUME ["/tmp", "/run"]
```

---

## tmpfs in Docker Compose

```yaml
version: "3.8"

services:
  api:
    image: myapp:1.0
    read_only: true       # read-only root filesystem
    tmpfs:
      - /tmp:size=256m,mode=1777
      - /run:size=64m
    ports:
      - "8080:8080"

  image-processor:
    image: processor:1.0
    tmpfs:
      - /tmp:size=2g      # large scratch space for image processing

  secrets-handler:
    image: myapp:1.0
    tmpfs:
      - /run/secrets:size=32m,noexec,nosuid
```

---

## Checking tmpfs Usage

```bash
# Inside a container — verify tmpfs is mounted
docker exec api df -h /tmp
# Filesystem      Size  Used Avail Use% Mounted on
# tmpfs           256M  4.0K  256M   1% /tmp

# Check mount type
docker exec api mount | grep tmpfs
# tmpfs on /tmp type tmpfs (rw,nosuid,nodev,noexec,relatime,size=262144k)

# Check how much is used
docker exec api df -h | grep tmpfs
```

---

## Comparison: tmpfs vs Writable Layer vs Bind Mount

| Feature | Writable Layer | Bind Mount | tmpfs |
|---|---|---|---|
| Location | Disk (overlay) | Host filesystem | RAM |
| Speed | Fast | Fast (Linux), slow (macOS) | Fastest |
| Persists after stop | No | Yes | No |
| Survives rm | No | Yes (host files stay) | No |
| Shared with host | No | Yes | No |
| Shared between containers | No | Yes (same path) | No |
| Sensitive data safe | No (on disk) | No (on disk) | Yes (RAM only) |
| Size limit | Host disk | Host disk | Configurable, max is RAM |

---

## When NOT to Use tmpfs

### Data that must persist

tmpfs is completely cleared when the container stops. Never use it for:
- Database files
- User uploads
- Application logs (unless you have a log driver routing them out)
- Any data that should survive a container restart

### Data that must be shared between containers

tmpfs is local to one container. If Container A and Container B both need access to the same files, use a named volume or bind mount — not tmpfs.

### Very large data

tmpfs uses RAM. If you mount a 10GB tmpfs, you have 10GB less RAM available to other processes. The OS will not always enforce this limit strictly (it may use swap), but as a general rule, keep tmpfs mounts appropriately sized.

---

## Summary

- tmpfs mounts create an in-memory filesystem inside a container — data exists only in RAM, never touches disk, and is cleared when the container stops
- Use tmpfs for sensitive temporary data (decrypted secrets, tokens) — RAM-only storage prevents data from persisting in disk caches or snapshots
- Use tmpfs for high-speed scratch space in I/O-heavy workloads — RAM access is orders of magnitude faster than disk
- Use tmpfs with `--read-only` root filesystem to create security-hardened containers that cannot modify their own code
- Always add `noexec` and `nosuid` to tmpfs mounts for sensitive paths — prevents execution of binaries uploaded to tmpfs
- Size tmpfs appropriately — too small causes write failures, too large wastes RAM
- Never use tmpfs for data that must persist across container restarts — use named volumes or bind mounts for that

In the next module, we move into Docker networking — how containers communicate with each other, with the host, and with the outside world.
