# What is a Container?

## Introduction

You have heard the word "container" throughout this course. Now it is time to build a precise, deep understanding of what a container actually is — not just the analogy, but the technical reality.

A container is more than just an isolated environment. Understanding what it is made of, how it runs, and what makes it different from everything else in computing will make you a much better DevOps engineer. Every time you debug a container issue, optimize a Dockerfile, or design a production deployment, this understanding is what you draw on.

---

## The Technical Definition

A **container** is a group of Linux processes that share a kernel with the host machine but are isolated from other processes using Linux namespaces and have their resource usage constrained by control groups (cgroups).

Let's break that down:

- **A group of Linux processes** — at its core, a container is just one or more processes running on your Linux host. When you run `docker run nginx`, an nginx process starts. That is your container.
- **Share a kernel with the host** — unlike a virtual machine, there is no separate OS or kernel. The container uses the same Linux kernel as the host machine.
- **Isolated using namespaces** — the kernel creates an isolated view of the system for each container. Processes inside the container cannot see processes outside it, cannot see the host filesystem, and have their own network interface.
- **Resource usage constrained by cgroups** — the kernel tracks and limits how much CPU, memory, and disk I/O the container can use.

This is the complete picture. Everything else about containers — images, Dockerfiles, registries — is infrastructure built around this core concept of isolated, resource-limited processes.

---

## What a Container Contains

When Docker creates a container, it sets up several things:

### 1. A Filesystem

The container has its own filesystem, separate from the host. This filesystem is assembled from:

- **Image layers** — read-only layers from the Docker image (more on this in the Images chapter)
- **A writable layer** — a thin layer added on top where the container can write files

When the container writes a file, it goes into the writable layer. The image layers below are never modified. When the container is removed, the writable layer is deleted — changes are lost unless you use volumes.

### 2. A Process Tree

The container has its own process namespace. From inside the container:

```bash
# Inside the container — sees only its own processes
ps aux

# Output:
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0  10628  1552 ?        Ss   09:30   0:00 nginx: master process
nginx       29  0.0  0.0  11092  2584 ?        S    09:30   0:00 nginx: worker process
```

From the host machine, you see all container processes alongside host processes — but from different perspectives:

```bash
# On the host — you can see container processes but with host PIDs
ps aux | grep nginx

# Output (different PIDs than inside the container):
root     15234  0.0  0.0  10628  1552 ?        Ss   09:30   0:00 nginx: master process
www-data 15235  0.0  0.0  11092  2584 ?        S    09:30   0:00 nginx: worker process
```

The container thinks its nginx process has PID 1. The host sees it as PID 15234. This is the process namespace in action.

### 3. A Network Interface

Each container gets its own network namespace with:

- Its own virtual network interface (typically `eth0` inside the container)
- Its own IP address (assigned by Docker)
- Its own routing table
- Its own set of network ports

From inside the container, it looks like the container has its own network card. It does not know it is sharing the host's physical network.

```bash
# Inside the container
ip addr show

# Output:
1: lo: <LOOPBACK,UP,LOWER_UP> ...
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    inet 172.17.0.2/16
```

### 4. Resource Limits

Docker uses cgroups to limit resource usage:

```bash
# Run a container with resource limits
docker run -d \
  --name api \
  --memory 512m \
  --memory-swap 512m \
  --cpus 1.0 \
  myapp:1.0
```

This container cannot use more than 512MB of RAM or more than 1 full CPU core — even if the host has 32GB and 16 cores. Other containers and host processes are protected from this container consuming all resources.

---

## Container vs Image

This distinction confuses many beginners. It is critical to understand.

An **image** is a blueprint — a static, read-only package that contains your application and everything it needs to run. An image is not running. It just sits there.

A **container** is a running instance created from an image. It is the live, active process.

The relationship is like this:

| Concept | Analogy | Docker equivalent |
|---|---|---|
| Image | A cookie-cutter mold | Defines the shape |
| Container | The actual cookie | The running instance |

You can create many containers from the same image:

```bash
# One image — three running containers
docker run -d -p 8081:80 --name web1 nginx
docker run -d -p 8082:80 --name web2 nginx
docker run -d -p 8083:80 --name web3 nginx

# All three run independently from the same nginx image
docker ps
# NAMES   IMAGE   STATUS
# web1    nginx   Up 2 minutes
# web2    nginx   Up 2 minutes
# web3    nginx   Up 1 minute
```

Each container has its own writable layer, its own network interface, and its own processes — but all share the same nginx image layers underneath.

---

## Container Isolation in Practice

To really feel what container isolation means, look at this example.

You run two containers on the same host — one Python 3.8, one Python 3.11:

```bash
# Container 1 — Python 3.8
docker run -it --name py38 python:3.8 bash

# Inside py38:
python --version
# Python 3.8.18

# Container 2 — Python 3.11 (in another terminal)
docker run -it --name py311 python:3.11 bash

# Inside py311:
python --version
# Python 3.11.7
```

Both containers run on the same host at the same time. They do not conflict. Each has its own isolated filesystem with its own Python version. Installing a package in one container does not affect the other.

This is exactly the solution to the dependency conflict problem described in Module 1. Containers make conflicting dependencies coexist without issue.

---

## The Container's Main Process (PID 1)

Every container has one main process — the process with PID 1 inside the container. This is the process defined by the `CMD` or `ENTRYPOINT` in the Dockerfile.

The container's lifecycle is tied to this main process:

- **Container starts** → PID 1 starts
- **Container running** → PID 1 is running
- **PID 1 exits** → container stops

This is a critical concept. If your main process exits — even successfully — the container stops.

```bash
# This container runs, prints "hello", and exits immediately
docker run alpine echo "hello"

# Check — container is stopped (Exited)
docker ps -a | grep alpine
# alpine    Exited (0) 2 seconds ago
```

```bash
# This container runs nginx (which runs in foreground) — stays up
docker run -d nginx

# Check — container is running
docker ps | grep nginx
# nginx    Up 2 minutes
```

When you design a containerized application, make sure your main process runs in the foreground and does not daemonize itself. If a process daemonizes (moves to the background), PID 1 exits and the container stops.

---

## Container Isolation Boundaries

Understanding what containers can and cannot see from each other helps you design secure deployments.

### What containers CANNOT see by default:
- Processes in other containers
- Filesystems of other containers
- Network traffic of other containers
- Environment variables of other containers

### What containers CAN share if you configure it:
- A network (using `--network`) — containers on the same Docker network can reach each other
- Volumes (`-v`) — multiple containers can mount the same volume
- The host network (`--network host`) — the container uses the host's network directly (removes network isolation)
- The host PID namespace (`--pid host`) — the container can see host processes (almost never do this)

For production applications on AWS, containers in the same ECS task share a network namespace by default. Containers in different tasks are fully isolated.

---

## Ephemeral Nature of Containers

Containers are designed to be **ephemeral** — temporary and replaceable.

When a container is removed, its writable layer is deleted. Any files written inside the container during its lifetime are gone.

```bash
# Start a container, write a file
docker run -it --name test ubuntu bash

# Inside container:
echo "important data" > /tmp/myfile.txt
cat /tmp/myfile.txt
exit

# Remove the container
docker rm test

# The data is gone — there is no way to recover it
```

This is intentional. Ephemeral containers encourage:

- **Stateless application design** — your application should not store important data inside the container
- **Reproducible deployments** — every container starts from a clean, known state
- **Easy updates** — replace the container with a new version; no cleanup needed

For data that needs to persist across container restarts and replacements, you use **Docker volumes** — covered in Module 7.

---

## Summary

- A container is a group of Linux processes isolated with namespaces and resource-limited with cgroups — it shares the host kernel but cannot see outside its isolation boundary
- Each container gets its own filesystem, process tree, network interface, and resource limits
- An image is the static blueprint; a container is the running instance created from that blueprint — you can run many containers from one image
- The container's lifecycle is tied to its main process (PID 1) — when PID 1 exits, the container stops
- Containers are ephemeral — data written inside a container is lost when the container is removed
- By default, containers cannot see each other's processes, filesystems, or network traffic — they are fully isolated unless you explicitly configure sharing

In the next file, we trace the full lifecycle of a container — from creation through running to stopped and removed — and understand what happens at each stage.
