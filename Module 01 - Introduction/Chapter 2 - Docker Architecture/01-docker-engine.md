# Docker Engine

## Introduction

When you run a Docker command like `docker run nginx`, something happens behind the scenes that makes a container appear almost instantly. Understanding what happens under the hood is not just academic knowledge — it helps you troubleshoot problems, optimize performance, and understand why Docker behaves the way it does.

The component responsible for all of this is **Docker Engine**. It is the core of the entire Docker platform. Everything else — the CLI, Docker Compose, Docker registries — builds on top of it.

This file explains what Docker Engine is, what it is made of, and exactly what happens from the moment you type a Docker command to the moment a container is running.

---

## What is Docker Engine?

Docker Engine is the runtime that builds and runs containers on your machine. It is a client-server application made up of three main components:

1. **Docker Daemon (`dockerd`)** — the long-running background process that does the actual work
2. **Docker REST API** — the interface that programs use to communicate with the daemon
3. **Docker CLI (`docker`)** — the command-line tool you use to interact with Docker

These three components work together every time you run a Docker command. The CLI talks to the daemon through the REST API, and the daemon does the heavy lifting of building images, creating containers, managing networks, and handling storage.

---

## The Three Components in Detail

### 1. Docker Daemon (`dockerd`)

The Docker daemon is a background process that runs continuously on your host machine. It listens for requests from the Docker CLI or other clients and carries them out.

The daemon is responsible for:

- **Building images** — reading Dockerfiles and assembling images layer by layer
- **Running containers** — creating and managing container processes
- **Managing networks** — creating virtual networks that containers communicate through
- **Managing volumes** — handling persistent storage for containers
- **Pulling and pushing images** — communicating with registries like Docker Hub or AWS ECR
- **Managing the container lifecycle** — starting, stopping, restarting, and removing containers

The daemon runs as the `dockerd` process on your system. On Linux, it starts automatically as a systemd service when you install Docker.

You can check if the daemon is running:

```bash
# Check daemon status on Linux
sudo systemctl status docker

# Output:
● docker.service - Docker Application Container Engine
     Loaded: loaded (/lib/systemd/system/docker.service; enabled)
     Active: active (running) since Mon 2024-01-15 09:23:41 UTC; 2h 15min ago
```

### 2. Docker REST API

The Docker REST API is the communication layer between clients and the daemon. Any program that wants to interact with Docker uses this API.

When you run `docker ps`, the Docker CLI makes an HTTP request to the REST API, which the daemon handles and responds to.

The API is a standard HTTP REST API. This means:

- You can interact with Docker from any programming language
- Tools like Docker Compose, Kubernetes, and CI/CD systems all use this API
- You can write your own scripts that control Docker using HTTP requests

Example: checking running containers using the API directly with curl:

```bash
# The Docker daemon exposes a Unix socket at /var/run/docker.sock
# You can send raw HTTP requests to it

curl --unix-socket /var/run/docker.sock http://localhost/containers/json | python3 -m json.tool
```

This returns the same information as `docker ps` but in raw JSON format. This is exactly how the Docker CLI works internally.

### 3. Docker CLI (`docker`)

The Docker CLI is the tool you interact with directly. Every command you type starting with `docker` goes through the CLI.

The CLI:

- Parses your command and arguments
- Formats them into an HTTP request
- Sends the request to the Docker daemon via the REST API
- Receives the response from the daemon
- Formats and displays the output to your terminal

The CLI is a client — it does not run containers itself. It just sends instructions to the daemon.

```bash
# This command:
docker run -d -p 8080:80 nginx

# Translates to an API call that tells the daemon:
# "Pull the nginx image if not available, create a container from it,
#  run it in detached mode, map port 8080 on the host to port 80 in the container"
```

---

## How Docker Engine Components Communicate

By default, the Docker CLI communicates with the daemon through a **Unix socket** at `/var/run/docker.sock`.

A Unix socket is a file on your filesystem that acts as a communication channel. It is faster than a network connection because both the CLI and daemon are on the same machine.

```
You type: docker run nginx
         ↓
Docker CLI
         ↓ (HTTP request over Unix socket)
/var/run/docker.sock
         ↓
Docker Daemon (dockerd)
         ↓
Docker creates the container
```

### Remote Docker Daemon

The Docker CLI can also communicate with a Docker daemon running on a remote machine over a network. This is used in CI/CD environments and when managing Docker on remote servers.

```bash
# Connect to a remote Docker daemon
export DOCKER_HOST=tcp://remote-server:2376

# Now all docker commands run against the remote daemon
docker ps
docker run nginx
```

This is how tools like Docker Machine and some CI/CD systems manage containers on remote hosts.

---

## The Container Runtime Layer

Beneath the Docker daemon, there is one more layer: the **container runtime**.

When Docker creates a container, it does not directly use the Linux kernel features itself. It delegates the actual container creation to a lower-level runtime.

Docker uses two runtime layers:

### containerd

`containerd` is a high-level container runtime that manages the complete container lifecycle:

- Pulling images from registries
- Managing image storage
- Creating and running containers
- Managing container networking and storage

Docker daemon delegates container management to containerd. When you install Docker, containerd is installed automatically as a dependency.

```bash
# Check if containerd is running
sudo systemctl status containerd
```

### runc

`runc` is the low-level runtime that actually creates containers using Linux kernel features (namespaces and cgroups). It is called by containerd for each new container.

runc implements the OCI (Open Container Initiative) runtime specification — an industry standard that ensures containers created by Docker can run on any compatible runtime.

### The Full Stack

```
Docker CLI (docker)
        ↓
Docker Daemon (dockerd)
        ↓
containerd
        ↓
runc
        ↓
Linux Kernel (namespaces + cgroups)
        ↓
Container is running
```

This layered architecture is why you can use containerd directly with Kubernetes without Docker. Kubernetes bypasses the Docker daemon and talks directly to containerd.

---

## What Happens When You Run a Container

Let's trace exactly what happens when you run:

```bash
docker run -d -p 8080:80 --name webserver nginx
```

**Step 1: CLI parses the command**

The Docker CLI parses `run`, reads the flags (`-d`, `-p 8080:80`, `--name webserver`), and identifies the image name (`nginx`).

**Step 2: CLI sends a request to the daemon**

The CLI sends an HTTP POST request to `/containers/create` via the Unix socket.

**Step 3: Daemon checks for the image locally**

The daemon checks if the `nginx` image exists in the local image cache.

```bash
# You can see local images
docker images
```

If the image is not found locally, the daemon automatically pulls it from Docker Hub.

**Step 4: Daemon pulls the image (if needed)**

```
Unable to find image 'nginx:latest' locally
latest: Pulling from library/nginx
a2abf6c4d29d: Pull complete
a9edb18cadd1: Pull complete
589b7251471a: Pull complete
```

The daemon downloads each image layer from the registry and stores it locally.

**Step 5: Daemon creates the container**

The daemon instructs containerd to create a container from the nginx image. containerd calls runc, which:

- Creates a new network namespace for the container
- Creates a new process namespace
- Sets up the container filesystem using image layers
- Applies cgroup resource limits

**Step 6: Daemon configures networking**

The daemon sets up port mapping. Requests arriving on port 8080 of the host will be forwarded to port 80 inside the container.

**Step 7: Container starts**

runc starts the container's main process (the nginx web server in this case).

**Step 8: Daemon returns the container ID**

The daemon sends back the container ID to the CLI, which displays it:

```
7d3c8e1f4a2b9c8d7e6f5a4b3c2d1e0f9a8b7c6d
```

This entire process — from command to running container — takes less than a second for a locally cached image.

---

## Docker Engine Configuration

Docker Engine is configured through a JSON file at `/etc/docker/daemon.json` on Linux.

Common configuration options:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "dns": ["8.8.8.8", "8.8.4.4"],
  "insecure-registries": [],
  "registry-mirrors": []
}
```

After editing this file, restart the daemon to apply changes:

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### Important Configuration Options for DevOps

**Log rotation** — Without this, container logs can fill up your disk:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "5"
  }
}
```

**Private registry** — Configure Docker to use your AWS ECR registry:

```json
{
  "insecure-registries": [],
  "registry-mirrors": ["https://123456789.dkr.ecr.us-east-1.amazonaws.com"]
}
```

**Storage driver** — `overlay2` is the recommended and default driver for modern Linux systems. Never change this unless you have a specific reason.

---

## Checking Docker Engine Status

A few commands every DevOps engineer should know for diagnosing Docker Engine issues:

```bash
# Check Docker daemon status
sudo systemctl status docker

# View Docker system information
docker info

# View Docker version (both CLI and daemon)
docker version

# View Docker daemon logs
sudo journalctl -u docker --since "1 hour ago"

# Check disk usage by Docker
docker system df

# Output:
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          12        8         2.1GB     890MB (42%)
Containers      4         2         124MB     45MB (36%)
Local Volumes   6         3         1.2GB     400MB (33%)
Build Cache     23        0         890MB     890MB
```

---

## Summary

- Docker Engine is the core runtime of the Docker platform, made up of the Docker daemon, the REST API, and the Docker CLI
- The Docker daemon (`dockerd`) is a background process that handles all container operations — building, running, networking, and storage
- The Docker CLI sends commands to the daemon via HTTP requests over a Unix socket at `/var/run/docker.sock`
- Beneath the daemon, containerd manages the container lifecycle, and runc uses Linux kernel features to actually create containers
- When you run `docker run`, the CLI, daemon, containerd, and runc all work together in under a second to produce a running container
- Docker Engine is configured through `/etc/docker/daemon.json` and should always have log rotation configured in production

In the next file, we examine the Docker daemon and Docker client in greater detail — specifically how they communicate and how you can configure that communication for remote and production environments.
