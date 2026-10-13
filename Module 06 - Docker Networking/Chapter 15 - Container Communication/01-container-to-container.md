# Container-to-Container Communication

## Introduction

The primary reason for networking containers together is to allow them to communicate. A microservices architecture relies entirely on container-to-container communication — the frontend talks to the API, the API talks to the database, the worker talks to the message queue.

This file covers the practical mechanics of how containers talk to each other, how to structure networks for security, and how Docker Compose simplifies multi-container environments.

---

## The Golden Rule: Use Custom Networks

As established in the Network Basics chapter, **containers on the default bridge network cannot resolve each other by name.** They can only communicate via IP addresses, which are dynamic and change every time a container restarts.

To enable reliable communication, you must place the containers on a **user-defined (custom) network**.

```bash
# 1. Create the network
docker network create app-net

# 2. Start the database
docker run -d --name database --network app-net postgres:16

# 3. Start the API
docker run -d --name api --network app-net my-api:1.0
```

---

## How Communication Works (Embedded DNS)

Once containers are on the same custom network, Docker's embedded DNS server takes over. 

When the `api` container attempts to connect to `database`, the following happens:
1. The API application makes a DNS request for the hostname `database`.
2. The DNS request goes to Docker's embedded DNS server (always located at `127.0.0.11` inside custom networks).
3. The DNS server looks up the current IP address of the container named `database`.
4. It returns the IP (e.g., `172.18.0.2`), and the connection is established.

### Aliases

You are not restricted to using the container's exact name. You can assign network aliases. This allows multiple containers to respond to the same DNS name (acting as a simple round-robin load balancer), or allows a container to be known by a different name on a specific network.

```bash
docker run -d \
  --name user-db-v2 \
  --network app-net \
  --network-alias db-primary \
  postgres:16
```

Now, the API container can connect to `db-primary:5432`, and the traffic will route to `user-db-v2`.

---

## Security via Network Segmentation

By default, a container can talk to *any* other container on the same custom network. In production, you should limit the blast radius of a security breach by isolating containers.

Imagine a classic 3-tier architecture:
1. Web Frontend (Nginx)
2. Backend API (Node.js)
3. Database (PostgreSQL)

The Web Frontend needs to talk to the API. The API needs to talk to the Database. **The Web Frontend should never be able to talk directly to the Database.**

We achieve this using multiple networks:

```bash
# Create two isolated networks
docker network create frontend-tier
docker network create backend-tier

# 1. Start the Database on the backend tier only
docker run -d --name db --network backend-tier postgres:16

# 2. Start the API. Connect it to the backend tier initially.
docker run -d --name api --network backend-tier my-api:1.0

# 3. Connect the API to the frontend tier as well
docker network connect frontend-tier api

# 4. Start the Web Frontend on the frontend tier only
docker run -d --name web --network frontend-tier nginx
```

Result:
- `web` and `api` share `frontend-tier`. They can communicate.
- `api` and `db` share `backend-tier`. They can communicate.
- `web` and `db` share no networks. They cannot communicate. The database is shielded from the public-facing web tier.

---

## Communication in Docker Compose

Docker Compose abstracts away the manual `docker network create` and `docker network connect` commands. 

By default, Compose creates a single custom network for your entire `docker-compose.yml` file and attaches all services to it. 

### Basic Compose Networking

```yaml
version: '3.8'

services:
  web:
    image: nginx
  api:
    image: my-api:1.0
```
In this simple setup, `web` can ping `api` immediately.

### Segmented Compose Networking

To implement the secure 3-tier architecture in Compose, you explicitly define the networks:

```yaml
version: '3.8'

services:
  web:
    image: nginx
    networks:
      - frontend

  api:
    image: my-api:1.0
    networks:
      - frontend
      - backend

  db:
    image: postgres:16
    networks:
      - backend

networks:
  frontend:
  backend:
```

This YAML file exactly replicates the secure, multi-network architecture we built via the CLI above. It is declarative, reproducible, and self-documenting.

---

## Troubleshooting Container Communication

When containers cannot communicate, follow these troubleshooting steps:

### 1. Verify they are on the same network
```bash
docker inspect api | grep Networks -A 10
docker inspect db | grep Networks -A 10
```
Ensure they share at least one custom network name.

### 2. Test DNS Resolution
Exec into the source container and try to ping the destination by name.

```bash
docker exec -it api bash
ping db
```
If it says "bad address" or "Name or service not known", the DNS lookup failed. Verify you are using a custom network and that you spelled the container name correctly.

### 3. Test Port Connectivity
Even if DNS works, the port might be closed or blocked. Use `curl` or `nc` (netcat) from inside the container.

```bash
# Test if the database port is accepting connections
docker exec -it api nc -vz db 5432
# Output: Connection to db 5432 port [tcp/postgresql] succeeded!
```
If this fails but `ping` works, the network is fine, but the application inside the `db` container is not listening correctly (e.g., PostgreSQL might be configured to only listen on `localhost` instead of `0.0.0.0`).

---

## AWS Context: Container Communication in ECS

In AWS ECS using `awsvpc` mode, Docker's internal networks are bypassed. Every container (task) gets a real VPC IP address.

To allow ECS tasks to communicate with each other securely:

1. **Service Discovery:** AWS Cloud Map is integrated with ECS to provide DNS resolution. When the `api` task starts, it registers its VPC IP address in AWS Route 53 with a name like `api.local`. The `web` task queries `api.local` to find it.
2. **Security Groups:** Network isolation is enforced using AWS Security Groups. You would create a Security Group for the Database that allows inbound traffic on port 5432 *only* if the traffic originates from the API's Security Group. 

While the underlying technology changes from Docker Bridges to AWS VPC routing, the architectural principles of DNS resolution and tiered network isolation remain identical.

---

## Summary

- Containers communicate via Docker's embedded DNS server, which automatically resolves container names to IP addresses.
- **Crucial:** Automatic DNS only works on user-defined (custom) networks, never on the default bridge.
- Network segmentation (using multiple networks) is a core security practice to isolate public-facing containers from sensitive backend databases.
- A container can be connected to multiple networks simultaneously to act as a bridge between tiers.
- Docker Compose handles complex multi-network architectures declaratively.
- When troubleshooting, first verify shared networks, then test DNS with `ping`, then test port connectivity with `nc` or `curl`.

In the next file, we will cover how to expose these networked containers to the outside world using port mapping.
