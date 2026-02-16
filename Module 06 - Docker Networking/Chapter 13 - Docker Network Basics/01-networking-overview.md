# Networking Overview

## Introduction

Networking is often considered the most complex part of Docker, but it does not have to be. Understanding how containers communicate with each other, with the host machine, and with the outside world is essential for building distributed applications and microservices.

This file provides a high-level overview of Docker networking — the problem it solves, the abstractions it uses, and how it integrates into production environments like AWS.

---

## The Container Networking Problem

Without Docker, an application runs directly on the host OS. It binds to a port on the host's network interface (e.g., `eth0`), and traffic comes directly to it. If you run two applications that both want port 80, you have a port conflict. The second application fails to start.

Docker solves this by giving every container its own **isolated network stack**.

When a container starts, it gets:
- Its own virtual network interface (`eth0` inside the container)
- Its own IP address (e.g., `172.17.0.2`)
- Its own routing table
- Its own iptables rules
- Its own loopback interface (`localhost` or `127.0.0.1`)

Inside the container, it looks exactly like a standard Linux server. The container can bind to port 80, port 443, or port 5432 without conflicting with any other container on the host, because it is binding to its own private IP address, not the host's IP address.

---

## How Docker Connects Containers

Isolation is great, but containers need to communicate. Docker connects these isolated container networks to the host (and to each other) using **virtual bridges**.

A virtual bridge acts like a physical network switch inside your computer. When you install Docker, it creates a default bridge named `docker0`.

```
Host Machine
│
├── Physical Interface (eth0: 192.168.1.100)  ← Reaches the Internet
│
└── Virtual Bridge (docker0: 172.17.0.1)      ← Reaches Containers
      │
      ├── Container 1 (eth0: 172.17.0.2)
      │
      └── Container 2 (eth0: 172.17.0.3)
```

In this setup:
- Container 1 and Container 2 can talk to each other using their `172.17.x.x` IP addresses because they are plugged into the same virtual switch (`docker0`).
- They can reach the internet because Docker sets up NAT (Network Address Translation) rules that route their outbound traffic through the host's `eth0`.
- The outside world cannot reach them unless you explicitly map a port (e.g., `docker run -p 8080:80`).

---

## The Container Network Namespace

The Linux kernel feature that makes this possible is called **network namespaces**. A network namespace is a logically isolated copy of the network stack.

You can actually see these namespaces on a Linux host (though Docker hides them by default):

```bash
# Start a container
docker run -d --name web nginx

# Get the container's PID
PID=$(docker inspect -f '{{.State.Pid}}' web)

# Create a symlink to make the namespace visible to the ip command
sudo mkdir -p /var/run/netns
sudo ln -s /proc/$PID/ns/net /var/run/netns/$PID

# Now you can run commands inside the container's network namespace from the host
sudo ip netns exec $PID ip addr show
```

You will see the container's loopback interface and its virtual `eth0` interface. This is exactly what the container sees when you run `ip addr` inside it.

---

## Docker Network Commands Overview

Docker provides a dedicated CLI for managing networks: `docker network`.

```bash
# List all networks on the host
docker network ls

# Output (default networks):
NETWORK ID     NAME      DRIVER    SCOPE
1a2b3c4d5e6f   bridge    bridge    local
2b3c4d5e6f7a   host      host      local
3c4d5e6f7a8b   none      null      local
```

```bash
# Inspect a specific network (shows subnet, gateway, and connected containers)
docker network inspect bridge

# Create a new custom network
docker network create my-app-net

# Connect a running container to a network
docker network connect my-app-net webserver

# Disconnect a container from a network
docker network disconnect bridge webserver

# Remove a network
docker network rm my-app-net

# Remove all unused networks
docker network prune
```

---

## Networking in Production (AWS Context)

The networking model changes depending on where you run your containers.

### Standalone Docker on EC2

When running Docker manually on an EC2 instance, you use the standard Docker bridge networks. Traffic flows from the internet → EC2 Security Group → EC2 ENI (Elastic Network Interface) → Docker Port Mapping (`-p`) → Docker Bridge → Container.

### AWS ECS on EC2 (bridge mode)

When using ECS with `bridge` network mode, it behaves exactly like standalone Docker. ECS manages the port mappings for you. If you run three containers on port 8080, ECS maps them to random high ports (e.g., 32768, 32769, 32770) on the host and registers those ports with the Application Load Balancer.

### AWS ECS and Fargate (awsvpc mode)

This is the modern, recommended way to run containers on AWS. In `awsvpc` mode, Docker's bridge networking is bypassed entirely.

Instead of plugging the container into a virtual bridge on the host, AWS provisions a dedicated ENI (Elastic Network Interface) directly from your VPC and attaches it straight to the container.

```
VPC Subnet (10.0.1.0/24)
│
├── EC2 Instance (ENI: 10.0.1.55)
│
├── ECS Task 1 (ENI: 10.0.1.102)  ← Container has a real VPC IP!
│
└── ECS Task 2 (ENI: 10.0.1.103)
```

In `awsvpc` mode:
- Every container (task) gets its own IP address from the VPC subnet
- Containers communicate with each other using real VPC IP addresses
- You apply AWS Security Groups directly to containers, not just to the host EC2 instance
- There are no port mappings (`-p`) or port conflicts — two containers can both listen on port 8080 because they have different IP addresses
- The Application Load Balancer routes traffic directly to the container's IP

This completely flattens the network architecture and removes the complexity of Docker bridge networking in production.

---

## The Network Drivers

Docker uses a pluggable architecture for networking called **drivers**. When you create a network, you specify which driver to use. The driver dictates how the network behaves.

The main drivers are:
- **bridge** — The default. Creates an isolated network on a single host.
- **host** — Removes isolation. The container uses the host's network directly.
- **none** — Complete isolation. The container has no network interfaces at all (except localhost).
- **overlay** — Multi-host networking. Connects containers across different physical machines (used in Docker Swarm).
- **macvlan** — Assigns a physical MAC address to the container, making it appear as a physical device on your network.

We will cover each of these drivers in detail in the following chapters.

---

## Summary

- Docker networking solves port conflicts by giving every container an isolated network namespace (its own IP address, interfaces, and routing table)
- Virtual bridges (like `docker0`) connect isolated containers to each other and to the host's physical network interface
- Use `docker network` commands (ls, inspect, create, connect) to manage networks
- In standard Docker, you must map ports (`-p`) to allow outside traffic into the container's isolated network
- In modern AWS ECS/Fargate deployments (`awsvpc` mode), containers bypass Docker bridges entirely and get real IP addresses directly from your AWS VPC
- Network behavior is controlled by drivers (`bridge`, `host`, `none`, `overlay`)

In the next file, we will look closer at these network drivers and how they fundamentally change how a container interacts with the network.
