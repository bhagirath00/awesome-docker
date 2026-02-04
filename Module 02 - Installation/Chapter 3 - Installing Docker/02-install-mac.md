# Installing Docker on macOS

## Introduction

macOS is the most common operating system for developers and DevOps engineers working on their local machines. Docker runs natively on Linux — but macOS is not Linux. It uses the Darwin kernel, which does not have Linux namespaces or cgroups.

To run containers on macOS, Docker uses a lightweight Linux virtual machine running in the background. This VM is completely managed for you — you never interact with it directly. From your perspective, Docker just works.

The official way to install Docker on macOS is **Docker Desktop**.

---

## What is Docker Desktop?

Docker Desktop is an application that installs and manages everything you need to run Docker on macOS:

- A lightweight Linux VM (using Apple's Hypervisor framework on Apple Silicon, or HyperKit on Intel)
- Docker Engine inside that VM
- Docker CLI on your macOS terminal
- Docker Compose
- Docker Buildx
- A graphical dashboard for managing containers and images

When you run `docker run nginx` in your terminal, the CLI sends the command to the Docker daemon running inside the Linux VM. The container runs inside that VM, and Docker maps ports to your macOS network so you can access them from your browser as if they were running natively.

---

## System Requirements

Before installing, verify your macOS meets the requirements:

**For Apple Silicon (M1, M2, M3 chips):**
- macOS 13 (Ventura) or later
- At least 4GB of RAM

**For Intel chips:**
- macOS 12 (Monterey) or later
- At least 4GB of RAM

Check your chip type:

```bash
uname -m

# arm64 = Apple Silicon (M1/M2/M3)
# x86_64 = Intel
```

---

## Installation Steps

### Step 1: Download Docker Desktop

Go to the Docker Desktop download page:
`https://www.docker.com/products/docker-desktop/`

Download the correct version for your chip:
- **Apple Silicon** → Docker Desktop for Mac with Apple Silicon
- **Intel** → Docker Desktop for Mac with Intel chip

You can also check which version you need from the terminal:

```bash
# If this returns arm64, download the Apple Silicon version
uname -m
```

### Step 2: Install Docker Desktop

1. Open the downloaded `.dmg` file
2. Drag the Docker icon to the Applications folder
3. Open Docker from your Applications folder (or Spotlight search)
4. macOS will ask for your password to allow Docker to install its helper components — this is required
5. Docker Desktop starts and runs in the menu bar

You will see the Docker whale icon in your macOS menu bar. When the animation stops, Docker is ready.

### Step 3: Verify the Installation

Open your terminal and run:

```bash
# Check Docker version
docker version

# Check Docker Compose version
docker compose version

# Run a test container
docker run hello-world
```

Expected output from `docker run hello-world`:

```
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

---

## Docker Desktop Dashboard

Docker Desktop provides a graphical interface for managing your Docker environment. You can access it by clicking the Docker whale icon in the menu bar and selecting **Dashboard**.

From the dashboard you can:

- See all running and stopped containers
- View container logs in real time
- Start, stop, restart, and remove containers
- Browse local images and delete ones you no longer need
- Monitor resource usage (CPU, memory, network, disk)
- Open a terminal directly inside a running container

The dashboard is useful for beginners who are not yet comfortable with the CLI. However, as you grow as a DevOps engineer, you will rely more heavily on the CLI — it is faster and scriptable.

---

## Configuring Docker Desktop

### Resource Allocation

By default, Docker Desktop allocates a portion of your machine's CPU and RAM to the Linux VM. You can adjust this in **Settings → Resources**.

Recommended settings for development:

| Resource | Minimum | Recommended |
|---|---|---|
| **CPUs** | 2 | 4 |
| **Memory** | 4 GB | 8 GB |
| **Disk image size** | 20 GB | 60 GB |
| **Swap** | 1 GB | 2 GB |

For resource-heavy work (running multiple services with Docker Compose, building large images), allocate more CPU and memory.

### File Sharing

By default, Docker Desktop allows containers to mount directories from your macOS filesystem. You can control which directories are accessible under **Settings → Resources → File Sharing**.

```bash
# Mount a local macOS directory into a container
docker run -v /Users/yourname/projects/myapp:/app python:3.11 python /app/app.py
```

For this to work, `/Users/yourname/projects/myapp` must be in the file sharing list.

### Network Settings

Docker Desktop creates a virtual network. Containers on the same Docker network can talk to each other by service name. From your macOS host, you access containers via `localhost` and the mapped port.

```bash
# Start nginx and access it via localhost
docker run -d -p 8080:80 nginx

# Access from your macOS browser or terminal
curl http://localhost:8080
```

---

## Apple Silicon and Multi-Platform Images

Apple Silicon Macs use the ARM64 architecture. Most Docker images on Docker Hub are built for AMD64 (Intel/AMD). Docker Desktop on Apple Silicon can run AMD64 images through emulation using **Rosetta 2**, but this has a performance cost.

### Check the Platform of an Image

```bash
docker inspect python:3.11-slim | grep Architecture
```

### Pull the Correct Architecture

```bash
# Pull specifically for your platform
docker pull --platform linux/arm64 python:3.11-slim

# Pull specifically for AMD64 (for compatibility testing)
docker pull --platform linux/amd64 python:3.11-slim
```

### Build Multi-Platform Images

When building images that need to run on both Intel and ARM (e.g., for deployment to AWS EC2 instances which may be ARM-based Graviton), use Docker Buildx:

```bash
# Create a multi-platform builder
docker buildx create --name multiplatform --use

# Build for both ARM64 and AMD64 and push to registry
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0 \
  --push \
  .
```

This single image works on Intel EC2 instances and AWS Graviton (ARM) instances.

---

## Enabling Rosetta for x86/amd64 Emulation

On Apple Silicon, you can enable Rosetta to improve AMD64 emulation performance.

Go to **Settings → General** and enable **Use Rosetta for x86/amd64 emulation on Apple Silicon**.

With Rosetta enabled, AMD64 containers run significantly faster on M-series Macs.

---

## Using Docker Desktop with AWS

Docker Desktop integrates with AWS in several useful ways for DevOps work.

### Authenticate with AWS ECR

```bash
# Configure AWS CLI credentials first
aws configure

# Then authenticate Docker with ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Pull your private image
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
```

### Test AWS Deployments Locally

You can run the same containers locally on Docker Desktop that will run in production on AWS ECS or EKS. This lets you test your full application stack — including environment variables, networking, and storage configurations — before deploying.

```yaml
# docker-compose.yml — same config works locally and on AWS
version: "3.8"

services:
  api:
    image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp-api:latest
    ports:
      - "8080:8080"
    environment:
      - ENV=development
      - AWS_REGION=us-east-1

  db:
    image: postgres:16
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
```

```bash
docker compose up
```

---

## Common macOS Docker Issues

### Issue: "docker: command not found" after installation

The Docker CLI is usually added to your PATH automatically. If it is not:

```bash
# Check if Docker CLI exists
ls /usr/local/bin/docker

# If it exists, add /usr/local/bin to PATH
echo 'export PATH="/usr/local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

### Issue: Docker Desktop is slow or using too much RAM

Containers are running inside a Linux VM. If the VM is starved of resources, everything slows down.

Fix: Increase memory allocation in **Settings → Resources → Memory**.

Also check if you have too many running containers consuming resources:

```bash
# See resource usage per container
docker stats
```

### Issue: "Cannot connect to Docker daemon" even though Docker Desktop is running

Sometimes the socket gets into a bad state. Try:

```bash
# Restart Docker Desktop from the menu bar
# Click the whale icon → Restart

# Or kill and restart from terminal
killall Docker
open /Applications/Docker.app
```

### Issue: Port already in use

```bash
# Find which process is using port 8080
lsof -i :8080

# Kill the process
kill -9 <PID>

# Or choose a different host port
docker run -p 8081:80 nginx
```

---

## Keeping Docker Desktop Updated

Docker Desktop updates frequently with security patches and new features. Enable automatic updates in **Settings → General → Automatically check for updates**.

To update manually:
- Click the Docker whale in the menu bar
- Select **Check for Updates**
- Follow the update prompts

In team environments, pin Docker Desktop to a specific version to ensure everyone uses the same version. You can download older versions from Docker's release notes page.

---

## Summary

- Docker Desktop is the official way to run Docker on macOS — it runs Docker Engine inside a lightweight Linux VM
- Download the correct version for your chip: Apple Silicon (ARM64) or Intel (AMD64)
- Configure resource limits in Docker Desktop settings — allocate enough CPU and RAM for your development workload
- Apple Silicon Macs can run AMD64 images through emulation; enable Rosetta in settings for better performance
- Use Docker Buildx to build multi-platform images that run on both Intel and ARM — important when deploying to mixed AWS environments
- Docker Desktop integrates smoothly with AWS — authenticate with ECR and test your full production containers locally

In the next file, we cover installing Docker on Windows.
