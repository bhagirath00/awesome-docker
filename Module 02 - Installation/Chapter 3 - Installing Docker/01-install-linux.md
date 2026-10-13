# Installing Docker on Linux

## Introduction

Linux is the primary operating system for running Docker in production. Whether you are setting up a development server, configuring an AWS EC2 instance, or preparing a CI/CD runner, the ability to install and configure Docker on Linux is a core DevOps skill.

This file covers three installation methods for the most common Linux distributions. You will also learn how to verify the installation, configure post-install settings, and troubleshoot common problems.

---

## Prerequisites

Before installing Docker, confirm the following:

- You are running a 64-bit Linux distribution
- Your kernel version is 3.10 or higher (check with `uname -r`)
- You have `sudo` privileges
- Your system has internet access

```bash
# Check your kernel version
uname -r

# Check your OS version
cat /etc/os-release

# Check available disk space (Docker needs at least 5GB free)
df -h /
```

---

## Method 1: Official Docker Repository (Recommended)

This is the recommended method. It gives you the latest stable Docker version and automatic updates through your package manager.

### On Ubuntu and Debian

**Step 1: Remove any old versions**

Old or unofficial Docker packages can conflict with the official installation. Remove them first.

```bash
sudo apt-get remove docker docker-engine docker.io containerd runc
```

Do not worry if the command reports that none of these packages are installed. That is fine.

**Step 2: Update the package index and install dependencies**

```bash
sudo apt-get update

sudo apt-get install -y \
  ca-certificates \
  curl \
  gnupg \
  lsb-release
```

**Step 3: Add Docker's official GPG key**

The GPG key ensures packages you download are genuine and have not been tampered with.

```bash
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

**Step 4: Add the Docker repository**

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

**Step 5: Install Docker Engine**

```bash
sudo apt-get update

sudo apt-get install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

This installs:
- `docker-ce` — Docker Engine (the daemon)
- `docker-ce-cli` — Docker CLI
- `containerd.io` — the container runtime
- `docker-buildx-plugin` — extended build capabilities
- `docker-compose-plugin` — Docker Compose v2

**Step 6: Verify the installation**

```bash
sudo docker run hello-world
```

You should see output that ends with:
```
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

---

### On CentOS, RHEL, and Amazon Linux 2

**Step 1: Remove old versions**

```bash
sudo yum remove docker \
  docker-client \
  docker-client-latest \
  docker-common \
  docker-latest \
  docker-latest-logrotate \
  docker-logrotate \
  docker-engine
```

**Step 2: Install the yum-utils package**

```bash
sudo yum install -y yum-utils
```

**Step 3: Add the Docker repository**

```bash
sudo yum-config-manager \
  --add-repo \
  https://download.docker.com/linux/centos/docker-ce.repo
```

**Step 4: Install Docker Engine**

```bash
sudo yum install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

**Step 5: Start and enable Docker**

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

**Step 6: Verify**

```bash
sudo docker run hello-world
```

---

### On Amazon Linux 2023

Amazon Linux 2023 uses `dnf` instead of `yum`:

```bash
# Install Docker
sudo dnf install -y docker

# Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# Verify
sudo docker run hello-world
```

---

## Method 2: Convenience Script

Docker provides a script that automates the installation for development environments. It is NOT recommended for production — use the repository method for production servers.

```bash
# Download and run the install script
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

This script detects your distribution and installs Docker automatically. Useful for quickly setting up development machines and CI runners.

---

## Method 3: Installing a Specific Version

In production environments, you should pin Docker to a specific version to avoid unexpected changes during system updates.

### On Ubuntu/Debian — List available versions:

```bash
apt-cache madison docker-ce
```

Output:
```
docker-ce | 5:24.0.7-1~ubuntu.22.04~jammy | https://download.docker.com/linux/ubuntu jammy/stable amd64 Packages
docker-ce | 5:24.0.6-1~ubuntu.22.04~jammy | https://download.docker.com/linux/ubuntu jammy/stable amd64 Packages
docker-ce | 5:24.0.5-1~ubuntu.22.04~jammy | https://download.docker.com/linux/ubuntu jammy/stable amd64 Packages
```

Install a specific version:

```bash
VERSION_STRING=5:24.0.7-1~ubuntu.22.04~jammy

sudo apt-get install -y \
  docker-ce=$VERSION_STRING \
  docker-ce-cli=$VERSION_STRING \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

### Prevent automatic upgrades

```bash
# Hold Docker at the current version
sudo apt-mark hold docker-ce docker-ce-cli containerd.io

# To allow upgrades again later
sudo apt-mark unhold docker-ce docker-ce-cli containerd.io
```

---

## Post-Installation Configuration

### Run Docker Without sudo

By default, running Docker commands requires `sudo`. This is cumbersome for daily use. Add your user to the `docker` group to fix this.

```bash
# Add current user to the docker group
sudo usermod -aG docker $USER

# Apply the change without logging out
newgrp docker

# Verify — this should work without sudo now
docker run hello-world
```

**Important:** The `docker` group grants privileges equivalent to root. Only add trusted users to this group.

### Configure Docker to Start on Boot

```bash
# Enable Docker to start automatically with the system
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```

### Configure Log Rotation

Without log rotation, container logs can fill your disk. Always configure this in production.

Create or edit `/etc/docker/daemon.json`:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "3"
  }
}
```

Restart Docker to apply:

```bash
sudo systemctl restart docker
```

### Configure Docker Data Directory

By default, Docker stores all data (images, containers, volumes) in `/var/lib/docker`. If your root filesystem is small, you can move this to a larger disk.

```bash
# Stop Docker
sudo systemctl stop docker

# Move existing data (if any)
sudo mv /var/lib/docker /mnt/data/docker

# Configure the new path in daemon.json
sudo nano /etc/docker/daemon.json
```

Add this to `daemon.json`:

```json
{
  "data-root": "/mnt/data/docker"
}
```

Restart Docker:

```bash
sudo systemctl start docker
docker info | grep "Docker Root Dir"
```

---

## Verifying Your Installation

After installation, confirm everything is working correctly.

```bash
# Check Docker version
docker version

# Check Docker system information
docker info

# Run a real container
docker run --rm alpine echo "Docker is working"

# Check the daemon status
sudo systemctl status docker

# Check Docker disk usage
docker system df
```

---

## Common Installation Issues

### Issue: "Got permission denied while trying to connect to the Docker daemon socket"

```bash
# Fix: Add user to docker group
sudo usermod -aG docker $USER
newgrp docker
```

### Issue: Docker service fails to start

```bash
# Check the daemon logs for errors
sudo journalctl -u docker --since "5 minutes ago" --no-pager

# Check daemon configuration for syntax errors
sudo docker daemon --validate
```

### Issue: "Cannot connect to the Docker daemon"

```bash
# Start the daemon manually
sudo systemctl start docker

# Check if it is running
sudo systemctl status docker
```

### Issue: Insufficient disk space

```bash
# Clean up unused Docker resources
docker system prune -a

# Check what is using disk space
docker system df -v
```

---

## Summary

- The recommended installation method on Linux is through the official Docker repository — it gives you the latest version with automatic updates
- Install `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin`, and `docker-compose-plugin` together
- After installation, add your user to the `docker` group to avoid needing `sudo` for every command
- Always enable Docker to start on boot with `systemctl enable docker`
- Always configure log rotation in `/etc/docker/daemon.json` before running containers in production
- Pin Docker to a specific version in production environments to prevent unexpected upgrades

In the next file, we cover installing Docker on macOS using Docker Desktop.
