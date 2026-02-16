# Exposing Ports

## Introduction

Container isolation is great for security, but eventually, users need to interact with your application. A web server locked inside a private container network is useless to the outside world.

Exposing ports (also known as port publishing or port mapping) is the mechanism Docker uses to punch a controlled hole through the network isolation, allowing traffic from the host machine—or the public internet—to reach a specific port inside a container.

This file covers the syntax of port mapping, binding to specific interfaces, the difference between `EXPOSE` and `-p`, and production patterns.

---

## The -p Flag: Publishing Ports

To publish a port, use the `-p` (or `--publish`) flag with `docker run`. The syntax maps a port on the Docker host to a port inside the container.

```bash
docker run -p [HOST_IP]:[HOST_PORT]:[CONTAINER_PORT] image
```

### 1. The Standard Map (Host to Container)
This is the most common use case. It maps a specific port on the host to a specific port in the container.

```bash
# Map host port 8080 to container port 80
docker run -d -p 8080:80 nginx
```
If the host's IP is `192.168.1.50`, you can access the Nginx container by navigating to `http://192.168.1.50:8080` or `http://localhost:8080` from the host itself.

### 2. Binding to a Specific Host Interface (Security Best Practice)
By default, `-p 8080:80` binds the port to `0.0.0.0` on the host, meaning the port is exposed on *every* network interface the host has (public IP, private LAN IP, localhost).

If you are running a database on a public EC2 instance, mapping `-p 5432:5432` exposes your database directly to the internet. This is highly dangerous.

To restrict access, bind the port to a specific interface, such as `localhost` (`127.0.0.1`):

```bash
# Bind ONLY to localhost. The container is reachable from the host machine,
# but completely inaccessible from the outside network.
docker run -d -p 127.0.0.1:5432:5432 postgres:16
```

### 3. Ephemeral (Random) Ports
If you want to run 5 instances of an API, you cannot map them all to host port 8080. You can let Docker choose random, available high ports on the host.

```bash
# Map container port 80 to a random available host port
docker run -d -p 80 nginx
```
To find out which port Docker assigned, use `docker port`:

```bash
docker port <container_name>
# Output: 80/tcp -> 0.0.0.0:32768
```
Now the container is accessible at port `32768`.

### 4. UDP Ports
Docker assumes TCP by default. If your application requires UDP (e.g., a DNS server or a game server), you must specify it explicitly:

```bash
docker run -d -p 53:53/udp bind9
```

---

## EXPOSE vs -p (Publish)

A common point of confusion for Docker beginners is the difference between the `EXPOSE` instruction in a `Dockerfile` and the `-p` flag in `docker run`.

### EXPOSE (Dockerfile)
```dockerfile
# Inside a Dockerfile
EXPOSE 8080
```
**`EXPOSE` does absolutely nothing to the network.** It does not open ports, it does not route traffic, and it does not make the container accessible to the host. 

It is purely **documentation**. It tells the person reading the Dockerfile, "The application inside this container is configured to listen on port 8080." It serves as a hint for the engineer running the container to know which port they should map.

### -p (Runtime)
```bash
docker run -p 80:8080 my-api
```
The `-p` flag actually creates the iptables NAT rules in the Linux kernel that physically route traffic from the host's port 80 into the container's port 8080.

---

## The -P (Capital P) Flag

If a Dockerfile contains `EXPOSE` instructions, you can use the uppercase `-P` flag (Publish All) to tell Docker to automatically map *all* exposed ports to random high ports on the host.

```bash
# If the image has EXPOSE 80 and EXPOSE 443
docker run -d -P nginx

docker ps
# PORTS
# 0.0.0.0:32769->80/tcp, 0.0.0.0:32768->443/tcp
```
This is useful in local development or automated testing where you need to spin up multiple containers without manually tracking port assignments.

---

## Port Mapping in Docker Compose

Docker Compose makes defining ports straightforward and declarative.

```yaml
version: '3.8'

services:
  web:
    image: nginx
    ports:
      - "8080:80"                  # Standard mapping
      - "127.0.0.1:5432:5432"      # Bound to localhost only
      - "53:53/udp"                # UDP port
      - "80"                       # Map to random host port
```

**Crucial Compose Concept:** You do *not* need to publish ports for containers to communicate with each other. 

If `web` needs to talk to `api` in the same Compose file, it uses Docker's internal DNS over the internal network. You only use the `ports:` block when you need the service to be accessible from *outside* the Docker host (e.g., from your laptop browser to the Compose environment).

---

## AWS Context: Application Load Balancers

In a production AWS environment using ECS on EC2, port mapping is handled automatically and dynamically to support scaling.

1. You configure the ECS Task Definition to map a container port (e.g., 80) to a host port of `0` (which tells ECS to pick a random ephemeral port).
2. ECS starts the container on EC2 instance A, mapping it to port `32768`.
3. ECS starts a second container on the same EC2 instance, mapping it to port `32769`.
4. ECS automatically updates the **Application Load Balancer (ALB) Target Group** to point to EC2 Instance A on ports `32768` and `32769`.

Traffic from the public hits the ALB on port 443. The ALB routes it directly to the ephemeral port on the EC2 instance, which Docker routes into the container.

*(Note: In AWS Fargate with `awsvpc` mode, port mapping is bypassed entirely. The ALB sends traffic directly to the container's dedicated VPC IP address).*

---

## Summary

- Use `-p HOST_PORT:CONTAINER_PORT` to route traffic from the host machine into the isolated container.
- For security, bind sensitive services to localhost using `-p 127.0.0.1:5432:5432` to prevent external network access.
- `EXPOSE` in a Dockerfile is only documentation; it does not open ports. `-p` at runtime opens the ports.
- Omit the host port (`-p 80`) to let Docker assign a random ephemeral port, avoiding conflicts.
- In multi-container architectures (like Compose), do not publish ports unless the service specifically needs to be accessed from outside the host network. Use internal DNS for container-to-container traffic.
- In AWS ECS on EC2, dynamic ephemeral ports combined with an ALB are used to run multiple instances of the same service on a single host.

In the next file, we will explore advanced custom network configurations, including configuring internal networks and integrating with external subnets.
