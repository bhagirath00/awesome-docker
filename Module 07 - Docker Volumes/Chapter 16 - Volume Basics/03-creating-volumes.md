# Creating Volumes

## Introduction

Creating volumes is straightforward, but the approach you choose — pre-creating versus inline creation, naming conventions, initial data seeding — affects how maintainable and reliable your data layer is in production. This file covers every method of creating volumes with the complete options available.

---

## Method 1: docker volume create

The explicit way to create a volume is with `docker volume create`. This creates the volume before any container uses it.

```bash
# Create a named volume
docker volume create pgdata

# Inspect it to confirm creation
docker volume inspect pgdata
```

Output of `docker volume inspect`:

```json
[
  {
    "CreatedAt": "2024-01-15T09:23:41Z",
    "Driver": "local",
    "Labels": {},
    "Mountpoint": "/var/lib/docker/volumes/pgdata/_data",
    "Name": "pgdata",
    "Options": {},
    "Scope": "local"
  }
]
```

Key fields:
- **Driver** — `local` means Docker manages the storage on the host
- **Mountpoint** — the actual path on the host where the volume data lives
- **Scope** — `local` means it is only available on this Docker host

The volume now exists on disk. No container has used it yet, so the `_data` directory is empty.

### Creating with Labels

Labels help organize and identify volumes:

```bash
docker volume create \
  --label app=myapp \
  --label environment=production \
  --label team=backend \
  pgdata
```

Filter volumes by label:

```bash
docker volume ls --filter label=app=myapp
```

---

## Method 2: Inline Creation with docker run

If you mount a named volume that does not exist yet, Docker creates it automatically the first time. This is the most common pattern because it keeps volume creation and container startup in a single command.

```bash
# Volume "pgdata" does not exist yet
docker run -d \
  --name db \
  -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:16
```

Docker creates `pgdata`, mounts it at `/var/lib/postgresql/data`, and starts the container. If `pgdata` already exists from a previous run, Docker mounts the existing volume — its data is fully intact.

This is the key behavior: **if the volume exists, Docker uses it; if it does not, Docker creates it**. This is why the pattern of using named volumes works reliably for database persistence across container recreations.

---

## The -v vs --mount Syntax

Docker supports two syntaxes for volume mounts, just like bind mounts.

### -v (Short Form)

```bash
docker run -v VOLUME_NAME:CONTAINER_PATH[:OPTIONS] image
```

```bash
# Named volume
docker run -v pgdata:/var/lib/postgresql/data postgres:16

# Read-only mount
docker run -v pgdata:/data:ro alpine

# Named volume + seeding from image
docker run -v pgdata:/var/lib/postgresql/data postgres:16
```

### --mount (Explicit Form)

```bash
docker run --mount type=volume,source=VOLUME_NAME,target=CONTAINER_PATH image
```

```bash
# Named volume with --mount
docker run --mount type=volume,source=pgdata,target=/var/lib/postgresql/data postgres:16

# Read-only
docker run --mount type=volume,source=pgdata,target=/data,readonly alpine
```

The `--mount` syntax is more verbose but makes the mount type explicit. In scripts and documentation, `--mount` is easier to understand at a glance. Both produce identical results.

---

## Initial Data Seeding — How Docker Populates New Volumes

When you mount an empty volume into a container path that already has content in the image, Docker copies the image's content into the volume. This is called **volume seeding** or **initial population**, and it only happens once — when the volume is first created.

This is how the PostgreSQL image works:

```bash
# First run — volume pgdata is empty
docker run -d -v pgdata:/var/lib/postgresql/data postgres:16
# Docker sees that pgdata is empty and /var/lib/postgresql/data has content in the image
# Docker copies the image's /var/lib/postgresql/data into pgdata
# PostgreSQL initializes its database files in the volume
```

```bash
# Second run — volume pgdata already has data
docker run -d -v pgdata:/var/lib/postgresql/data postgres:16
# Docker mounts the existing pgdata — does NOT overwrite the data
# PostgreSQL starts with the existing database
```

This seeding behavior makes volumes self-initializing. The first time you run a database container with a fresh volume, the database is automatically initialized. Subsequent runs use the existing data.

**Important:** This seeding only happens with named and anonymous volumes — **not** with bind mounts. A bind mount always shows whatever the host directory contains, even if it is empty.

---

## Volume Options with the local Driver

The built-in `local` volume driver supports additional options for advanced configurations. These are passed using `--opt` (or `-o`) flags with `docker volume create`.

### Mounting a Specific Host Path

You can make a named volume point to a specific host path instead of the default `/var/lib/docker/volumes/` location:

```bash
docker volume create \
  --driver local \
  --opt type=none \
  --opt device=/data/postgres \
  --opt o=bind \
  pgdata
```

This creates a named volume `pgdata` that maps to `/data/postgres` on the host. The container uses it as a regular named volume by name, but the data actually lives at your chosen path.

### tmpfs via Volume

You can even create a named tmpfs volume:

```bash
docker volume create \
  --driver local \
  --opt type=tmpfs \
  --opt device=tmpfs \
  --opt o=size=256m \
  temp-scratch
```

Now containers can mount the tmpfs by the name `temp-scratch` instead of using the `--tmpfs` flag.

### NFS Mounts

To mount an NFS share as a named volume (commonly used in multi-host environments):

```bash
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=nfs-server.example.com,rw \
  --opt device=:/export/data \
  nfs-data
```

---

## Creating Volumes in Docker Compose

In Docker Compose, volumes are defined in two places: in the `volumes:` key at the top level (declaring the volume) and in the service's `volumes:` key (mounting it).

```yaml
version: "3.8"

services:
  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: secret

  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data

volumes:
  pgdata:
    # No configuration needed — Docker creates a local named volume
  redis-data:
    labels:
      app: myapp
      environment: production
```

When `docker compose up` runs:
1. If `pgdata` does not exist, Compose creates it with the name `<project_name>_pgdata` (prefixed with the project name to avoid collisions).
2. If `pgdata` already exists from a previous run, Compose uses it unchanged.

### Compose Volume with Custom Options

```yaml
volumes:
  pgdata:
    driver: local
    driver_opts:
      type: none
      device: /data/postgres
      o: bind
```

---

## Verifying Volume Mount in a Running Container

After starting a container with a volume, confirm the mount is working:

```bash
# See all mounts for the container
docker inspect --format '{{json .Mounts}}' db | python3 -m json.tool

# Output:
[
  {
    "Type": "volume",
    "Name": "pgdata",
    "Source": "/var/lib/docker/volumes/pgdata/_data",
    "Destination": "/var/lib/postgresql/data",
    "Driver": "local",
    "Mode": "z",
    "RW": true,
    "Propagation": ""
  }
]
```

This confirms:
- **Type** is `volume` (not `bind` or `tmpfs`)
- **Name** is `pgdata` — the correct named volume
- **Source** is the host path where data actually lives
- **RW** is `true` — the mount is read-write

---

## Summary

- Create volumes explicitly with `docker volume create` or let Docker create them automatically when first referenced in `docker run -v volumename:/path`
- Use `-v volumename:/container/path` for concise syntax and `--mount type=volume,source=name,target=/path` for explicit, readable syntax — both are equivalent
- When an empty named volume is mounted to a container path that has content in the image, Docker automatically seeds (populates) the volume with that content — this only happens once on first use
- Docker Compose creates volumes with a project-name prefix to avoid collisions — `pgdata` in a `myapp` project becomes `myapp_pgdata`
- The `local` driver supports advanced options: custom host paths, NFS mounts, and tmpfs — all addressable by a clean named volume name
- Use `docker inspect` to verify a container has the correct volume mounted, its host path, and whether it is read-write

In the next file, we compare volumes and bind mounts side by side — understanding the difference is critical for choosing the right storage mechanism for each use case.
