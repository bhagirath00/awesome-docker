# Port Mapping

## Introduction

Containers run in their own isolated network namespace. By default, no network traffic from the outside world reaches a container — not even from the host machine. Port mapping is how you bridge that isolation and make container services accessible.

Understanding port mapping correctly is essential. It affects how your application is accessed locally, how services communicate with each other in Docker Compose, and how traffic flows in production on AWS ECS with load balancers.

---

## How Container Networking Works by Default

When Docker creates a container, it gets:
- Its own virtual network interface (`eth0` inside the container)
- Its own IP address (typically in the `172.17.0.0/16` range)
- A route to a virtual bridge network (`docker0`) that connects to the host

The container can reach the internet (outbound traffic works). But by default, nothing outside the container can reach it — no port on the host forwards traffic into the container.

```bash
# Run nginx — it listens on port 80 inside the container
docker run -d nginx

# Try to access it from the host — it fails
curl http://localhost:80
# curl: (7) Failed to connect to localhost port 80: Connection refused

# The container has an IP, but only accessible within Docker's internal network
docker inspect --format '{{.NetworkSettings.IPAddress}}' container_name
# 172.17.0.2

curl http://172.17.0.2:80
# This works — but only from the host, not from outside the host
```

Port mapping creates a rule that forwards traffic from a host port to a container port.

---

## The -p Flag — Port Mapping

### Basic Syntax

```bash
docker run -p HOST_PORT:CONTAINER_PORT image
```

```bash
# Map host port 8080 to container port 80
docker run -d -p 8080:80 nginx

# Now accessible from the host
curl http://localhost:8080
# Returns nginx default page
```

### Multiple Port Mappings

```bash
# Map HTTP and HTTPS
docker run -d \
  -p 80:80 \
  -p 443:443 \
  nginx

# Map multiple application ports
docker run -d \
  -p 8080:8080 \    # API
  -p 9090:9090 \    # metrics endpoint
  myapp:1.0
```

### Map to a Specific Host Interface

By default, Docker binds to all network interfaces (`0.0.0.0`). This means the port is accessible from any network interface on the host — including public ones if the host is internet-facing.

To restrict access:

```bash
# Only accessible from localhost (127.0.0.1) — not from other machines
docker run -p 127.0.0.1:8080:80 nginx

# Only accessible from a specific network interface
docker run -p 192.168.1.100:8080:80 nginx

# Accessible from all interfaces (default)
docker run -p 0.0.0.0:8080:80 nginx
# equivalent to:
docker run -p 8080:80 nginx
```

Binding to `127.0.0.1` is a security best practice when:
- The service is only for internal use (database, admin panel)
- You have a reverse proxy (nginx, Traefik) that handles external traffic
- You do not want the service directly accessible from the network

### Let Docker Choose the Host Port

You can tell Docker to pick any available host port:

```bash
# Docker assigns a random available host port
docker run -d -p 80 nginx

# Find out which port was assigned
docker port container_name 80
# 0.0.0.0:32768

# Or with docker ps
docker ps
# PORTS: 0.0.0.0:32768->80/tcp
```

Random ports are useful when running multiple containers from the same image that all listen on the same port.

---

## UDP Port Mapping

TCP is the default protocol. Specify UDP explicitly when needed:

```bash
# Map UDP port 53 (DNS)
docker run -p 53:53/udp dns-server:1.0

# Map both TCP and UDP for the same port
docker run \
  -p 53:53/tcp \
  -p 53:53/udp \
  dns-server:1.0
```

---

## Checking Port Mappings

```bash
# List all port mappings for a container
docker port webserver

# Output:
80/tcp -> 0.0.0.0:8080
443/tcp -> 0.0.0.0:8443

# Check a specific container port
docker port webserver 80
# 0.0.0.0:8080

# See ports in docker ps
docker ps --format "table {{.Names}}\t{{.Ports}}"
# NAMES       PORTS
# webserver   0.0.0.0:8080->80/tcp, 0.0.0.0:8443->443/tcp
```

---

## EXPOSE vs Port Mapping

These two things are often confused. They are completely different.

| | `EXPOSE` in Dockerfile | `-p` in `docker run` |
|---|---|---|
| **What it does** | Documents the port — no effect on networking | Actually maps and opens the port |
| **Who uses it** | Other engineers reading your Dockerfile | Docker at container startup |
| **Does it open the port?** | No | Yes |
| **Is it required for -p?** | No | N/A |

```dockerfile
# EXPOSE tells people "this container uses port 8080"
# It does NOT open port 8080 — just documentation
EXPOSE 8080
```

```bash
# -p actually opens the port and makes it accessible
docker run -p 8080:8080 myapp:1.0
```

You can use `-p` without `EXPOSE` in the Dockerfile — it still works. `EXPOSE` is metadata that helps Docker Compose, tooling, and engineers understand which ports the container uses.

---

## Port Mapping in Docker Compose

In Docker Compose, port mapping is defined under the `ports` key:

```yaml
version: "3.8"

services:
  api:
    image: myapp:1.0
    ports:
      - "8080:8080"       # host:container
      - "9090:9090"       # metrics

  nginx:
    image: nginx:1.25
    ports:
      - "80:80"
      - "443:443"
      - "127.0.0.1:8081:80"  # admin only on localhost

  postgres:
    image: postgres:16
    ports:
      - "127.0.0.1:5432:5432"  # only accessible from localhost for security
    environment:
      - POSTGRES_PASSWORD=pass
```

### Container-to-Container Communication in Compose

Containers in the same Docker Compose project can communicate with each other using service names as hostnames — **without port mapping**. Port mapping is only needed for access from outside Docker.

```yaml
version: "3.8"

services:
  api:
    image: myapp:1.0
    environment:
      # Talk to postgres by service name — no port mapping needed
      - DATABASE_URL=postgresql://user:pass@postgres:5432/myapp
      # Talk to redis by service name
      - REDIS_URL=redis://redis:6379

  postgres:
    image: postgres:16
    # No ports: here — api reaches it via Docker's internal network
    environment:
      - POSTGRES_PASSWORD=pass

  redis:
    image: redis:7-alpine
    # No ports: here — api reaches it via service name "redis"
```

Exposing database ports externally (on the host) is a security risk in production. Containers that need each other should communicate via Docker's internal network, not via the host.

---

## Port Conflicts

If the host port is already in use, Docker fails with an error:

```bash
docker run -p 8080:80 nginx

# Error:
# docker: Error response from daemon: driver failed programming external connectivity
# on endpoint webserver: Bind for 0.0.0.0:8080 failed: port is already allocated.
```

Diagnose and resolve:

```bash
# Find what is using port 8080 on Linux
sudo ss -tlnp | grep 8080
# or
sudo lsof -i :8080

# Find on macOS
lsof -i :8080

# Kill the conflicting process or choose a different host port
docker run -p 8081:80 nginx
```

---

## Port Mapping in AWS ECS

On AWS ECS, you do not run `docker run` commands directly. Port mapping is defined in the task definition and handled by ECS.

### Port Mapping in ECS Task Definition

```json
{
  "containerDefinitions": [
    {
      "name": "api",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2",
      "portMappings": [
        {
          "containerPort": 8080,
          "hostPort": 8080,
          "protocol": "tcp"
        }
      ]
    }
  ]
}
```

### awsvpc Mode — No Host Port Needed

When using `awsvpc` networking mode (required for Fargate, recommended for ECS on EC2), each task gets its own network interface and IP. No host port mapping is needed:

```json
{
  "networkMode": "awsvpc",
  "containerDefinitions": [
    {
      "name": "api",
      "portMappings": [
        {
          "containerPort": 8080,
          "protocol": "tcp"
          // No hostPort needed in awsvpc mode
        }
      ]
    }
  ]
}
```

Traffic reaches your container through an Application Load Balancer (ALB) that routes to the container's IP directly:

```
Internet
    ↓
Application Load Balancer
    ↓
Container IP:8080 (via awsvpc)
```

---

## Production Pattern — Reverse Proxy

In production, you typically do not expose application ports directly. Instead, all traffic goes through a reverse proxy (nginx or AWS ALB) that routes to multiple backend containers.

```bash
# Application containers — no external port mapping
# They are only accessible within Docker's internal network
docker run -d --name api1 --network appnet myapp:1.0
docker run -d --name api2 --network appnet myapp:1.0

# Nginx reverse proxy — only this has external port mapping
docker run -d \
  --name nginx \
  --network appnet \
  -p 80:80 \
  -p 443:443 \
  -v ./nginx.conf:/etc/nginx/conf.d/default.conf \
  nginx:1.25
```

This pattern:
- Only exposes one point of entry to the outside world
- Application containers are not directly accessible from the internet
- The proxy handles SSL termination, load balancing, and request routing

---

## Summary

- By default, no traffic reaches a container from outside Docker — port mapping creates explicit access
- `-p HOST_PORT:CONTAINER_PORT` maps a host port to a container port
- Bind to `127.0.0.1:HOST_PORT` to restrict access to localhost only — use this for databases and admin services
- `EXPOSE` in Dockerfiles is documentation only — it does not open any ports; `-p` at runtime actually opens them
- In Docker Compose, containers in the same project communicate by service name over Docker's internal network — no port mapping needed between services
- Databases should never have external port mappings in production — only the application containers that need to reach them should access them via Docker's internal network
- On AWS ECS with `awsvpc` mode, containers get their own IPs — the ALB routes directly to container IPs without host port mapping

In the next file, we cover container naming and restart policies — two simple but critical configuration decisions for production deployments.
