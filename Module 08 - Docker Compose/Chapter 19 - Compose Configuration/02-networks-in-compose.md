# Networks in Docker Compose

## Introduction

Docker Compose automatically creates a default network for your application and attaches all services to it. For simple applications, you never need to think about networking — services can reach each other by name immediately.

For production applications with security requirements, you need explicit control: databases should not be on the same network as web-facing services, monitoring tools should have limited access, and some services should be completely isolated from the internet.

This file covers the complete Compose networking model — from the automatic default network to multi-tier, security-conscious architectures.

---

## The Default Network

When you run `docker compose up`, Compose creates a bridge network named `<project_name>_default`. Every service is automatically attached to this network unless you specify otherwise.

```yaml
# No networks: defined — all services on the default network
version: "3.8"

services:
  api:
    image: myapp:1.0
    ports:
      - "8080:8080"

  db:
    image: postgres:16

  redis:
    image: redis:7-alpine
```

In this Compose file, all three services are on `myapp_default`. The API can reach the database at hostname `db:5432` and Redis at `redis:6379`.

### Verifying the Default Network

```bash
docker compose up -d

docker network ls | grep myapp
# NETWORK ID   NAME            DRIVER  SCOPE
# 3f4a5b6c     myapp_default   bridge  local

docker network inspect myapp_default
# Shows all three containers connected to the network
```

---

## Defining Custom Networks

Explicit network definitions give you control over the network topology, isolation, and configuration.

```yaml
version: "3.8"

services:
  api:
    image: myapp:1.0
    networks:
      - default

  db:
    image: postgres:16
    networks:
      - backend   # only on backend, not default

networks:
  default:        # The auto-created default, explicitly named for clarity
  backend:
    driver: bridge
    internal: true   # no outbound internet access
```

When you specify `networks:` on a service, it is attached **only** to those networks. It is no longer automatically added to the default network unless you explicitly include `default` in its network list.

---

## Multi-Tier Network Architecture

The standard three-tier architecture separates traffic between public, API, and data layers:

```yaml
version: "3.8"

services:
  nginx:
    image: nginx:1.25
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    networks:
      - frontend     # can talk to api

  api:
    image: myapp:1.4.2
    # No ports — nginx proxies to it, not exposed directly
    networks:
      - frontend     # can talk to nginx
      - backend      # can talk to db and redis

  db:
    image: postgres:16
    # No ports published — only accessible via backend network
    networks:
      - backend      # can only talk to api

  redis:
    image: redis:7-alpine
    networks:
      - backend

networks:
  frontend:
    driver: bridge

  backend:
    driver: bridge
    internal: true   # no direct internet access for database tier
```

With this configuration:
- `nginx` and `api` share `frontend` — nginx proxies requests to the API
- `api`, `db`, and `redis` share `backend` — API queries the database and cache
- `nginx` **cannot** reach `db` or `redis` — no shared network
- `db` and `redis` have no public ports and `internal: true` — completely isolated from the internet

---

## Network Configuration Options

### driver

```yaml
networks:
  mynet:
    driver: bridge      # default for single-host (most common)
    driver: overlay     # multi-host (Docker Swarm)
    driver: host        # use host network (no isolation)
    driver: none        # no network
```

### internal

```yaml
networks:
  backend:
    internal: true   # containers cannot reach the internet via this network
```

### Custom Subnet and Gateway

```yaml
networks:
  mynet:
    ipam:
      driver: default
      config:
        - subnet: "10.5.0.0/16"
          gateway: "10.5.0.1"
          ip_range: "10.5.1.0/24"
```

Specify subnets when running Docker on hosts connected to networks (VPNs, AWS VPCs) that use overlapping address ranges.

### DNS Options

```yaml
networks:
  mynet:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.name: myapp-bridge   # custom bridge name
```

### Labels

```yaml
networks:
  backend:
    labels:
      app: myapp
      tier: backend
      environment: production
```

---

## Service-Level Network Configuration

You can configure how a specific service attaches to a network:

```yaml
services:
  api:
    networks:
      frontend:              # attach to frontend with default settings
      backend:
        aliases:
          - api-service      # other containers can reach api as "api" or "api-service"
          - internal-api
        ipv4_address: 10.5.0.10   # static IP (requires subnet defined)
```

### Network Aliases

Aliases give a service additional DNS names on a specific network:

```yaml
services:
  legacy-api:
    image: myapp:0.9
    networks:
      backend:
        aliases:
          - api        # new services use "api", legacy stuff uses "legacy-api"
          - api-v1
```

This is useful during migration: the new version of a service can be introduced while still being reachable by both the old and new names.

---

## External Networks

An external network is one created outside of Compose that Compose should attach services to without managing it:

```bash
# Create the shared network manually
docker network create shared-services
```

```yaml
# Project A
version: "3.8"

services:
  database:
    image: postgres:16
    networks:
      - shared-services

networks:
  shared-services:
    external: true   # Compose uses existing network, does not create/delete it
```

```yaml
# Project B
version: "3.8"

services:
  api:
    image: myapp:1.0
    networks:
      - shared-services

networks:
  shared-services:
    external: true
```

Now the `api` in Project B can reach `database` in Project A by hostname — they share the externally managed network.

### External Network with a Different Name

```yaml
networks:
  db-network:
    external:
      name: myapp-production-backend   # the actual network name
```

---

## Default Network Configuration

You can configure the automatically created default network using the reserved name `default`:

```yaml
version: "3.8"

services:
  api:
    image: myapp:1.0
    # No networks: — automatically on the default network

networks:
  default:
    driver: bridge
    ipam:
      config:
        - subnet: "10.10.0.0/16"
    internal: false
    labels:
      app: myapp
```

---

## Connecting to Networks Started by Another Compose Project

A common development pattern: a shared infrastructure stack (databases, queues) runs permanently, and individual service stacks connect to it.

```bash
# Start the shared infrastructure
docker compose -p infrastructure -f infrastructure/docker-compose.yml up -d
# Creates network: infrastructure_default
```

```yaml
# services/docker-compose.yml
version: "3.8"

services:
  api:
    image: myapp:1.0
    networks:
      - default
      - infrastructure_default   # connect to infrastructure stack's network

networks:
  infrastructure_default:
    external: true
```

The API can now resolve `db` (running in the infrastructure stack) by name.

---

## Troubleshooting Compose Networking

```bash
# List all networks created by a Compose project
docker compose config --networks

# Inspect a specific network
docker network inspect myapp_backend

# Verify which containers are on a network
docker network inspect myapp_backend --format '{{range .Containers}}{{.Name}} {{end}}'

# Test connectivity from inside a service
docker compose exec api ping db
docker compose exec api nc -vz db 5432

# Check DNS resolution
docker compose exec api nslookup db
# or
docker compose exec api cat /etc/resolv.conf
```

---

## Summary

- Compose automatically creates a `<project>_default` bridge network and attaches all services to it — no configuration required for simple applications
- Specify a `networks:` list per service to attach it only to specific networks — services are not automatically on the default network when you do this
- `internal: true` on a network blocks outbound internet access — use it for database and cache tiers that should never initiate external connections
- Network aliases give a service additional DNS names on a specific network — useful for backward compatibility during migrations
- Use `external: true` on a network to connect services to a pre-existing network managed outside Compose — enables cross-project service communication
- Specify subnets in `ipam.config` when running on infrastructure with overlapping address ranges (corporate VPNs, AWS VPCs)

In the next file, we cover `depends_on` — the full options for controlling service startup order and health-based dependency waiting.
