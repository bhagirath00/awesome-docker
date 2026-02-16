# Bridge Networks

## Introduction

User-defined bridge networks are the standard way to connect containers on a single host. Unlike the default bridge network, user-defined bridges provide automatic DNS resolution, better isolation, and granular control over network configuration.

Whether you are spinning up a local development environment or running standalone Docker in production, understanding how to create and manage custom bridge networks is a required skill. 

This file covers the mechanics of custom bridge networks and how Docker Compose uses them under the hood.

---

## Creating a Custom Bridge Network

Creating a custom network is simple:

```bash
docker network create my-app-net
```

By default, this command creates a network using the `bridge` driver. You can verify it:

```bash
docker network ls

# Output:
# NETWORK ID     NAME         DRIVER    SCOPE
# 8c2d5e6f7a8b   my-app-net   bridge    local
```

### Advanced Creation: Specifying Subnets

When Docker creates a bridge network, it automatically assigns a subnet (e.g., `172.18.0.0/16`, `172.19.0.0/16`). In complex environments, especially when connecting Docker hosts via VPNs or AWS VPC peering, these automatic subnets might conflict with existing infrastructure.

You can explicitly define the subnet and gateway when creating the network:

```bash
docker network create \
  --driver bridge \
  --subnet 10.5.0.0/16 \
  --gateway 10.5.0.1 \
  --ip-range 10.5.1.0/24 \
  my-custom-net
```

- `--subnet`: The overall CIDR block for the network.
- `--gateway`: The IP address of the bridge interface on the host (the router for the containers).
- `--ip-range`: Restricts container IP allocation to a specific smaller range within the subnet.

---

## Connecting Containers to Networks

### At Startup

The most common method is to attach the container to the network when running it:

```bash
# Run a database on the custom network
docker run -d --name db --network my-app-net postgres:16

# Run an API on the same network
docker run -d --name api --network my-app-net myapp:1.0
```

Because both containers are on `my-app-net`, the API container can connect to the database simply by using the hostname `db` (e.g., `postgres://user:pass@db:5432/mydb`).

### Connecting a Running Container

You do not have to stop a container to change its network. You can attach a running container to an additional network dynamically:

```bash
# Start a container on the default bridge
docker run -d --name admin-tool ubuntu sleep 3600

# Connect it to the custom network
docker network connect my-app-net admin-tool
```

The `admin-tool` container now has **two** network interfaces (two `eth` devices). It can communicate with containers on the default bridge AND containers on `my-app-net`.

### Disconnecting a Container

Similarly, you can unplug a container from a network on the fly:

```bash
docker network disconnect my-app-net admin-tool
```

---

## Network Isolation Strategy

Custom bridges provide strong isolation. Containers on different custom networks cannot reach each other, even if they are on the same host. This allows you to securely separate environments or microservice boundaries.

```
Host Machine
│
├── frontend-net (Bridge)
│     ├── Nginx Proxy
│     └── Web UI Container
│
└── backend-net (Bridge)
      ├── API Container
      └── Database Container
```

How do the layers communicate? The API container needs to talk to both the Web UI and the Database. 

You solve this by connecting the API container to **both** networks:

```bash
# Create the networks
docker network create frontend-net
docker network create backend-net

# Run the Database (backend only)
docker run -d --name db --network backend-net postgres:16

# Run the API (attach to backend on startup)
docker run -d --name api --network backend-net my-api:1.0

# Connect the API to the frontend network too
docker network connect frontend-net api

# Run the Web UI (frontend only)
docker run -d --name web-ui --network frontend-net my-ui:1.0
```

Result:
- The Web UI can talk to the API.
- The API can talk to the Database.
- The Web UI **cannot** talk to the Database. It is network-isolated.

---

## Docker Compose and Bridge Networks

Docker Compose abstracts away the manual `docker network` commands. When you run `docker compose up`, Compose automatically creates a custom bridge network for the project and attaches all services to it.

```yaml
# docker-compose.yml (in a directory named "myapp")
version: '3.8'

services:
  web:
    image: nginx
  api:
    image: my-api:1.0
```

When you run `docker compose up`, Compose:
1. Creates a bridge network named `myapp_default`.
2. Starts the `web` container on `myapp_default`.
3. Starts the `api` container on `myapp_default`.

This is why services in the same `docker-compose.yml` can automatically resolve each other by service name.

### Defining Multiple Networks in Compose

You can replicate the frontend/backend isolation strategy in Compose easily:

```yaml
version: '3.8'

services:
  web-ui:
    image: my-ui:1.0
    networks:
      - frontend-net
      
  api:
    image: my-api:1.0
    networks:
      - frontend-net
      - backend-net
      
  db:
    image: postgres:16
    networks:
      - backend-net

networks:
  frontend-net:
  backend-net:
```

Compose handles creating both custom bridge networks and plugging the containers into the correct ones.

---

## Inspecting Network Traffic

If you need to troubleshoot network issues, you can inspect the custom bridge network:

```bash
docker network inspect my-app-net
```

This shows the subnet, the gateway, and a list of all connected containers with their current IP addresses.

To see the actual iptables rules Docker creates for isolation and NAT:

```bash
sudo iptables -t nat -L -n
sudo iptables -L DOCKER -n
```

*Note: Never manually edit iptables rules created by Docker. Docker manages these dynamically, and manual changes will be overwritten or cause routing failures.*

---

## Summary

- User-defined bridge networks are the correct way to network containers on a single host.
- They provide automatic, embedded DNS resolution (unlike the default bridge).
- Containers can be dynamically connected to or disconnected from custom networks while running.
- Connecting a container to multiple networks is the standard pattern for controlling traffic flow between application tiers (e.g., keeping databases isolated from frontend networks).
- Docker Compose automatically handles the creation and management of custom bridge networks for you.
- You can specify exact subnets and IP ranges if you need to avoid conflicts with your corporate network or AWS VPCs.

In the next file, we will explore the `host` network driver and when you might want to bypass bridge networking entirely.
