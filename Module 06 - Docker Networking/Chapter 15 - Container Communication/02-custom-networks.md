# Custom Networks

## Introduction

As your Docker environment grows from a single container to a complex architecture, simply creating a standard bridge network is not always enough. Advanced scenarios require you to configure how Docker routes traffic, restrict whether containers can reach the internet, and avoid IP collisions with your corporate network or AWS VPCs.

This file explores the advanced options available when defining custom networks via the CLI and Docker Compose, including internal networks, specific subnet routing, and connecting to existing infrastructure.

---

## The Internal Network Flag

By default, containers on a custom bridge network can route traffic to the external internet (via NAT on the host). 

However, some containers should *never* have internet access. For example, a database container should only ever receive connections from the API container, and it should never need to initiate an outbound connection to download data or contact external servers.

You can enforce this by creating an **internal** network.

```bash
docker network create --internal secure-db-net
```

When the `--internal` flag is used, Docker creates the network namespace without the default route that forwards traffic to the host's external interface.

### Example: Isolating a Database

```bash
# Create the networks
docker network create public-api-net
docker network create --internal secure-db-net

# Start the DB on the internal network
docker run -d --name secret-db --network secure-db-net postgres:16

# Start the API, connected to both
docker run -d --name api-server \
  --network public-api-net \
  my-api:1.0

docker network connect secure-db-net api-server
```

In this setup:
- The `api-server` can reach the internet (via `public-api-net`).
- The `api-server` can reach the `secret-db` (via `secure-db-net`).
- The `secret-db` **cannot** reach the internet. If an attacker breaches the database, they cannot download exploit payloads or exfiltrate data via the internet.

### Internal Networks in Docker Compose

```yaml
version: '3.8'
services:
  api:
    image: my-api:1.0
    networks:
      - default
      - db-tier
  db:
    image: postgres:16
    networks:
      - db-tier

networks:
  # The default network has outbound internet access
  default:
  
  # The db-tier network is completely isolated from the internet
  db-tier:
    internal: true
```

---

## Defining Custom Subnets and IP Ranges

When Docker creates a network, it automatically pulls a subnet from the `172.17.0.0/16` to `172.31.0.0/16` range. 

If your host machine is connected to a corporate VPN or an AWS VPC that also uses the `172.x.x.x` ranges, Docker's automatic assignment will create a **routing collision**. Your host will not know whether to route `172.20.x.x` traffic into the Docker container or out to the AWS VPC.

To prevent this, you must explicitly define the subnet for your custom network to a range that you know does not conflict with your infrastructure.

```bash
# Create a network on a specific 10.x.x.x subnet
docker network create \
  --subnet 10.10.0.0/16 \
  --gateway 10.10.0.1 \
  --ip-range 10.10.1.0/24 \
  custom-subnet-net
```

### Static IP Assignment

Once a custom subnet is defined, you have the option to assign a static IP address to a container when you run it.

```bash
docker run -d \
  --name legacy-app \
  --network custom-subnet-net \
  --ip 10.10.1.50 \
  legacy-app:1.0
```

*Note:* Static IP assignment is generally considered an anti-pattern in modern container orchestration because containers are ephemeral. You should rely on DNS resolution instead of static IPs. However, it is occasionally necessary when integrating with legacy software that hardcodes IP addresses.

### Custom Subnets in Docker Compose

You can define the IPAM (IP Address Management) configuration directly in the Compose file:

```yaml
version: '3.8'

services:
  web:
    image: nginx
    networks:
      custom_net:
        ipv4_address: 10.5.0.100

networks:
  custom_net:
    ipam:
      driver: default
      config:
        - subnet: "10.5.0.0/16"
          gateway: "10.5.0.1"
```

---

## External Networks

Often, multiple different Docker Compose projects need to communicate with each other. 

By default, `docker-compose up` creates a network prefixed with the project directory name (e.g., `project_default`). A container in `project_a` cannot resolve or reach a container in `project_b`.

To allow them to communicate, you define an **external network**. You create the network manually via the CLI, and then configure both Compose files to attach to it.

```bash
# Manually create the shared network
docker network create shared-infrastructure
```

**Project A (`docker-compose.yml`)**
```yaml
version: '3.8'
services:
  database:
    image: postgres:16
    networks:
      - shared-infrastructure

networks:
  shared-infrastructure:
    external: true
```

**Project B (`docker-compose.yml`)**
```yaml
version: '3.8'
services:
  api:
    image: my-api:1.0
    networks:
      - shared-infrastructure

networks:
  shared-infrastructure:
    external: true
```

When you start Project B, Docker Compose will not try to create the `shared-infrastructure` network. It knows the network already exists and simply attaches the `api` container to it. The `api` container can now resolve and connect to the `database` container from Project A.

---

## AWS Context: Custom ENI Configuration

If you manage your own EC2 instances and run standard Docker, managing subnets to prevent collisions with your VPC CIDR blocks is critical.

However, if you are using AWS ECS in `awsvpc` mode, all of this complexity disappears. 
- You do not define Docker subnets.
- You do not use the `--internal` flag.
- You do not assign static IPs via Docker.

Instead, the ECS Task uses an AWS Elastic Network Interface (ENI) provisioned directly from your VPC Subnets. If you want a task to be "internal," you simply deploy it into an AWS Private Subnet that lacks a NAT Gateway. If you want static IPs, you use an AWS Network Load Balancer or AWS Cloud Map. The networking is shifted entirely from Docker's software layer to AWS's native software-defined networking layer.

---

## Summary

- The `--internal` flag creates a network without outbound internet access, perfect for securing backend databases from exfiltration attacks.
- Specify `--subnet` and `--gateway` when creating a network to prevent IP collisions with corporate VPNs or Cloud VPCs.
- Avoid using static `--ip` assignments unless supporting legacy applications; rely on DNS instead.
- Use `external: true` in Docker Compose to connect containers across different Compose projects by attaching them to a manually created shared network.
- In modern AWS ECS deployments (`awsvpc`), Docker custom network configurations are bypassed in favor of native VPC subnets and Security Groups.

In the next file, we will explore the mechanism that makes container communication possible: Docker's embedded DNS server.
