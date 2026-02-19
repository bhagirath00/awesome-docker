# Volume Commands

## Introduction

Docker volumes are first-class objects with their own complete set of management commands under the `docker volume` subcommand. Knowing how to list, inspect, and clean up volumes is essential for day-to-day operations — volumes accumulate over time and unchecked volume growth can fill your disk without warning.

This file covers every volume management command, what each flag does, and the operational patterns for keeping your Docker host clean.

---

## docker volume ls — List Volumes

```bash
# List all volumes on the host
docker volume ls

# Output:
DRIVER    VOLUME NAME
local     pgdata
local     redis-data
local     api-uploads
local     3f4a5b6c7d8e9f0a1b2c3d4e5f6a7b8c   ← anonymous volume
```

The `DRIVER` column shows how the volume is managed — `local` means Docker manages it on the host filesystem. Third-party drivers (for NFS, S3, EFS) show their driver name here.

### Filtering

```bash
# Show only volumes with a specific label
docker volume ls --filter label=app=myapp

# Show only dangling volumes (not connected to any container)
docker volume ls --filter dangling=true

# Show volumes by driver
docker volume ls --filter driver=local

# Custom output format
docker volume ls --format "table {{.Name}}\t{{.Driver}}\t{{.Mountpoint}}"
```

Dangling volumes are volumes not mounted by any container (running or stopped). They are safe candidates for cleanup, though verify they are not needed before removing them.

---

## docker volume inspect — Detailed Volume Information

`docker volume inspect` shows the full metadata for one or more volumes.

```bash
# Inspect a single volume
docker volume inspect pgdata

# Output:
[
  {
    "CreatedAt": "2024-01-15T09:23:41Z",
    "Driver": "local",
    "Labels": {
      "app": "myapp",
      "environment": "production"
    },
    "Mountpoint": "/var/lib/docker/volumes/pgdata/_data",
    "Name": "pgdata",
    "Options": {},
    "Scope": "local"
  }
]

# Inspect multiple volumes at once
docker volume inspect pgdata redis-data api-uploads

# Extract a specific field using Go template format
docker volume inspect --format '{{.Mountpoint}}' pgdata
# /var/lib/docker/volumes/pgdata/_data

# Extract the creation timestamp
docker volume inspect --format '{{.CreatedAt}}' pgdata
# 2024-01-15T09:23:41Z
```

The **Mountpoint** field gives you the actual host path. You can use this path directly with standard Linux tools to inspect or manipulate volume contents from the host:

```bash
# Get the host path
MOUNT=$(docker volume inspect --format '{{.Mountpoint}}' pgdata)

# List the files in the volume from the host
sudo ls -la $MOUNT

# Check disk usage of the volume from the host
sudo du -sh $MOUNT
```

---

## docker volume rm — Remove Volumes

```bash
# Remove a specific volume
docker volume rm pgdata

# Remove multiple volumes
docker volume rm pgdata redis-data temp-cache

# Force remove (skips confirmation for dangling volumes)
docker volume rm -f old-volume
```

**Safety check:** Docker will refuse to remove a volume that is currently mounted by a running container:

```bash
docker volume rm pgdata
# Error response from daemon: remove pgdata: volume is in use - [a1b2c3d4...]
```

To remove it, stop and remove the container first:

```bash
docker stop db && docker rm db
docker volume rm pgdata
```

---

## docker volume prune — Clean Up Unused Volumes

`docker volume prune` removes all **dangling** (unused) volumes — volumes not currently mounted by any container, running or stopped.

```bash
# Interactive prune (asks for confirmation)
docker volume prune

# Output:
# WARNING! This will remove all local volumes not used by at least one container.
# Are you sure you want to continue? [y/N] y
# Deleted Volumes:
# 3f4a5b6c7d8e9f0a...
# a1b2c3d4e5f6g7h8...
# Total reclaimed space: 2.3GB

# Skip confirmation
docker volume prune -f

# Prune with a label filter — only prune volumes matching the label
docker volume prune --filter label=environment=staging
```

**Warning:** Be very careful with `docker volume prune` on production hosts. It deletes all unused volumes — if a database container was recently stopped for maintenance, its volume is "dangling" and will be deleted. Always verify which volumes are dangling before pruning:

```bash
# Inspect dangling volumes first
docker volume ls --filter dangling=true

# Review the list, then prune safely
docker volume prune
```

---

## docker volume create — Create a Volume

```bash
# Create a simple named volume
docker volume create pgdata

# Create with labels
docker volume create \
  --label app=myapp \
  --label environment=production \
  pgdata

# Create with driver options (NFS example)
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw \
  --opt device=:/exports/data \
  nfs-volume

# Create with size limit (tmpfs)
docker volume create \
  --driver local \
  --opt type=tmpfs \
  --opt device=tmpfs \
  --opt o=size=512m \
  temp-scratch
```

---

## Checking Which Containers Use a Volume

Docker does not provide a direct command for "which containers use this volume," but you can get this from `docker inspect`:

```bash
# Find which containers mount the pgdata volume
docker ps -a --format '{{.Names}}' | xargs -I{} docker inspect {} \
  --format '{{.Name}} {{range .Mounts}}{{.Name}}{{end}}' | grep pgdata

# Alternative: inspect the container directly if you know its name
docker inspect db --format '{{range .Mounts}}{{.Name}}{{end}}'
# pgdata
```

---

## Disk Usage — Understanding Volume Storage

Volumes can grow unexpectedly. A database volume that started at 100MB may reach 50GB after months of usage. Monitor regularly.

```bash
# Docker's built-in disk usage overview
docker system df

# Output:
TYPE            TOTAL   ACTIVE  SIZE    RECLAIMABLE
Images          15      8       12.5GB  4.1GB (32%)
Containers      23      5       2.1GB   1.8GB (85%)
Local Volumes   12      6       45.3GB  3.2GB (7%)
Build Cache     0       0       0B      0B

# Detailed view including volume sizes
docker system df -v

# Output includes:
VOLUME NAME     LINKS   SIZE
pgdata          1       15.2GB
redis-data      1       128MB
old-testdb      0       2.1GB   ← dangling, safe to prune
```

The `LINKS` column tells you how many containers currently have the volume mounted. `0` means the volume is dangling.

---

## Full System Cleanup

When you want to reclaim all disk space on a Docker host (useful when cleaning up a CI runner or development machine):

```bash
# Remove everything: stopped containers, unused networks, dangling images, build cache
docker system prune

# Add -a to also remove all images not used by a running container
docker system prune -a

# Add --volumes to also remove all dangling volumes
docker system prune --volumes

# Nuclear option: remove everything
docker system prune -a --volumes -f
```

**Never run `docker system prune --volumes` on a production host without first checking which volumes contain live data.** Use `docker volume ls --filter dangling=true` to review dangling volumes first.

---

## Volume Naming Conventions

Consistent volume naming saves operational headaches. In teams, establish and enforce naming conventions:

```bash
# Pattern: appname-component-environment
docker volume create myapp-postgres-production
docker volume create myapp-postgres-staging
docker volume create myapp-uploads-production
docker volume create myapp-redis-production

# Pattern: component-environment (shorter, used inside a team's dedicated infrastructure)
docker volume create postgres-prod
docker volume create redis-prod
docker volume create uploads-prod
```

In Docker Compose, Compose automatically prefixes volumes with the project name:

```yaml
# In a project named "myapp", this creates volume "myapp_postgres_data"
volumes:
  postgres_data:
```

---

## Summary

- `docker volume ls` lists all volumes; use `--filter dangling=true` to find volumes not mounted by any container
- `docker volume inspect VOLUME` shows the Mountpoint (actual host path), creation date, labels, and driver — use the Mountpoint to inspect volume contents from the host
- `docker volume rm` removes specific volumes — Docker refuses to remove a volume mounted by a running container
- `docker volume prune` removes all dangling volumes — review with `docker volume ls --filter dangling=true` before running on production hosts
- `docker system df -v` shows disk usage by volume — essential for tracking storage growth over time
- Establish consistent naming conventions for volumes — `appname-component-environment` makes it immediately clear what a volume belongs to when listed on the host

In the next chapter, we cover named volumes in depth — the primary production storage pattern — along with sharing volumes between containers, backup/restore strategies, and volume drivers for cloud storage.
