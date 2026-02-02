# Docker Daemon and Client

## Introduction

In the previous file, you learned that Docker Engine has three main parts: the daemon, the REST API, and the CLI. You got an overview of all three.

This file goes deeper into the relationship between the **Docker daemon** and the **Docker client**. Understanding this relationship in detail matters because it affects how you work with Docker in real environments — on your local machine, on remote servers, inside CI/CD pipelines, and on AWS.

You will also learn how to configure this communication, troubleshoot connection issues, and use the Docker socket safely.

---

## The Docker Client-Server Model

Docker follows a classic **client-server architecture**.

- The **server** is the Docker daemon (`dockerd`). It is the brain. It manages all Docker objects — images, containers, networks, and volumes.
- The **client** is the Docker CLI (`docker`). It is the interface. It takes your commands, sends them to the daemon, and shows you the results.

The key point: **the client does not run containers.** It only sends instructions to the daemon. All actual work happens in the daemon.

This separation is powerful because it means your client and server do not need to be on the same machine. You can run the Docker CLI on your laptop and have it control a Docker daemon on a remote server.

```
┌─────────────────────────────────────────────┐
│              YOUR MACHINE                   │
│                                             │
│  ┌──────────────┐     ┌──────────────────┐  │
│  │  Docker CLI  │────▶│  Docker Daemon   │  │
│  │   (client)   │     │    (dockerd)     │  │
│  └──────────────┘     └──────────────────┘  │
│                                             │
└─────────────────────────────────────────────┘

OR across machines:

┌───────────────┐              ┌───────────────────────┐
│  Your Laptop  │              │    Remote Server      │
│               │              │                       │
│  Docker CLI   │─── TCP ─────▶│   Docker Daemon       │
│  (client)     │              │   (dockerd)           │
└───────────────┘              └───────────────────────┘
```

---

## How the Client Communicates with the Daemon

The Docker client and daemon communicate using the Docker REST API. The transport used depends on how Docker is configured.

### Unix Socket (Default on Linux and macOS)

By default, the Docker daemon listens on a **Unix domain socket** at `/var/run/docker.sock`.

A Unix socket is a special file that acts as a communication endpoint. It is only accessible from the same machine — which makes it more secure than exposing a network port.

```bash
# The socket file exists when dockerd is running
ls -la /var/run/docker.sock

# Output:
srw-rw---- 1 root docker 0 Jan 15 09:23 /var/run/docker.sock
```

Notice the permissions: the socket is owned by `root` and the `docker` group. Your user must be in the `docker` group to run Docker commands without `sudo`.

```bash
# Add your user to the docker group (log out and back in after this)
sudo usermod -aG docker $USER

# Verify group membership
groups $USER
```

### TCP Socket (For Remote Access)

To connect a client to a remote daemon, you configure the daemon to listen on a TCP socket.

**Warning:** Exposing the Docker daemon over TCP without TLS is dangerous. Anyone who can reach that port has full control over your Docker daemon — and effectively your entire server.

To enable TLS-secured remote access, edit `/etc/docker/daemon.json`:

```json
{
  "hosts": ["unix:///var/run/docker.sock", "tcp://0.0.0.0:2376"],
  "tls": true,
  "tlscert": "/etc/docker/certs/server-cert.pem",
  "tlskey": "/etc/docker/certs/server-key.pem",
  "tlsca": "/etc/docker/certs/ca.pem"
}
```

On the client side:

```bash
# Set the remote daemon address
export DOCKER_HOST=tcp://your-server-ip:2376
export DOCKER_TLS_VERIFY=1
export DOCKER_CERT_PATH=~/.docker/certs

# Now all docker commands go to the remote daemon
docker info
docker ps
docker run nginx
```

### Named Pipe (Windows)

On Windows, Docker uses a named pipe instead of a Unix socket:

```
npipe:////./pipe/docker_engine
```

This is handled automatically. You do not need to configure it unless you are doing advanced Windows Docker work.

---

## The Docker Socket in CI/CD Pipelines

The Docker socket is commonly used in CI/CD pipelines to give a CI container the ability to build and run Docker images.

### Mounting the Docker Socket

When a CI job needs to run Docker commands, you mount the Docker socket from the host into the CI container:

```yaml
# GitLab CI example
build-image:
  image: docker:24
  variables:
    DOCKER_HOST: unix:///var/run/docker.sock
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock
  script:
    - docker build -t myapp:${CI_COMMIT_SHA} .
    - docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:${CI_COMMIT_SHA}
```

```yaml
# GitHub Actions example
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Build Docker image
        run: docker build -t myapp:${{ github.sha }} .
      - name: Push to AWS ECR
        run: |
          aws ecr get-login-password --region us-east-1 | \
          docker login --username AWS --password-stdin \
          123456789.dkr.ecr.us-east-1.amazonaws.com
          docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:${{ github.sha }}
```

In the GitHub Actions example, the runner already has Docker installed with access to the Docker socket, so you can use Docker commands directly.

### Security Warning: Docker Socket Access

Mounting the Docker socket into a container gives that container complete control over the Docker daemon. A process inside that container can:

- Start containers with `--privileged` access
- Mount host directories
- Read files from the host
- Escape the container isolation

In a CI/CD environment this is usually acceptable because CI runners are controlled environments. In production application containers, never mount the Docker socket unless you have a specific, well-understood reason.

---

## Docker Context

**Docker context** is a feature that lets you manage connections to multiple Docker daemons without manually setting environment variables each time.

### Creating and Using Contexts

```bash
# View current contexts
docker context ls

# Output:
NAME        TYPE    DESCRIPTION                               DOCKER ENDPOINT
default *   moby    Current DOCKER_HOST based configuration   unix:///var/run/docker.sock

# Create a context for a remote development server
docker context create dev-server \
  --docker "host=tcp://dev.example.com:2376,ca=/path/to/ca.pem,cert=/path/to/cert.pem,key=/path/to/key.pem"

# Create a context for a production server
docker context create production \
  --docker "host=tcp://prod.example.com:2376,ca=/path/to/ca.pem,cert=/path/to/cert.pem,key=/path/to/key.pem"

# Switch to the dev server context
docker context use dev-server

# Now all commands run against the dev server
docker ps
docker images

# Switch back to local
docker context use default
```

This is far cleaner than managing `DOCKER_HOST` environment variables manually, especially when switching between multiple environments.

### AWS ECS with Docker Context

Docker Desktop supports AWS ECS directly through Docker contexts. You can deploy containers to ECS using familiar Docker commands:

```bash
# Create an ECS context
docker context create ecs myecscontext

# Switch to ECS context
docker context use myecscontext

# Deploy a Compose file directly to ECS
docker compose up

# This deploys your docker-compose.yml to AWS ECS automatically
```

---

## The Docker Daemon Lifecycle

Understanding how the daemon starts, stops, and handles signals is important for production operations.

### Starting the Daemon

On Linux with systemd:

```bash
# Start Docker
sudo systemctl start docker

# Enable Docker to start on boot
sudo systemctl enable docker

# Restart Docker
sudo systemctl restart docker
```

### Stopping the Daemon Gracefully

When the daemon stops, what happens to running containers depends on your configuration.

By default, stopping the daemon does NOT stop running containers. The containers continue running because they are processes managed by containerd, which runs independently of the daemon.

```bash
# Stop the daemon (containers keep running)
sudo systemctl stop docker

# Verify containers are still running
# (After restarting the daemon)
sudo systemctl start docker
docker ps
```

### Live Restore

Docker has a feature called **live restore** that allows the daemon to restart without interrupting running containers. Enable it in `/etc/docker/daemon.json`:

```json
{
  "live-restore": true
}
```

With live restore enabled:

- The daemon can be restarted for updates or configuration changes
- Running containers are not affected
- This is critical for production servers where you cannot afford container downtime during daemon updates

---

## Checking Daemon and Client Version Compatibility

The Docker client and daemon can sometimes be different versions. Check both:

```bash
docker version
```

Output:

```
Client: Docker Engine - Community
 Version:           24.0.7
 API version:       1.43
 Go version:        go1.20.10
 Git commit:        afdd53b
 Built:             Thu Oct 26 09:07:41 2023
 OS/Arch:           linux/amd64

Server: Docker Engine - Community
 Engine:
  Version:          24.0.7
  API version:      1.43 (minimum version 1.12)
  Go version:       go1.20.10
```

The **API version** is what matters for compatibility. A newer client can talk to an older daemon, but the client will limit itself to the API version the daemon supports. If the daemon is much newer than the client, you might miss features.

In CI/CD environments, always pin the Docker version to avoid unexpected changes:

```yaml
# Pin Docker version in GitHub Actions
- name: Setup Docker
  uses: docker/setup-buildx-action@v3
  with:
    version: v24.0.7
```

---

## Common Client-Daemon Connection Issues

### "Cannot connect to the Docker daemon"

```
docker: Cannot connect to the Docker daemon at unix:///var/run/docker.sock.
Is the docker daemon running?
```

**Fix:**

```bash
# Check if daemon is running
sudo systemctl status docker

# If not running, start it
sudo systemctl start docker

# Check daemon logs for errors
sudo journalctl -u docker --since "5 minutes ago"
```

### "Permission denied while trying to connect to the Docker daemon socket"

```
permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock
```

**Fix:**

```bash
# Add user to docker group
sudo usermod -aG docker $USER

# Apply group change (or log out and back in)
newgrp docker

# Verify
docker ps
```

### "Error response from daemon: conflict"

```
docker: Error response from daemon: Conflict. The container name "/webserver" is already in use
```

**Fix:**

```bash
# Remove the existing container
docker rm webserver

# Or force remove if it is running
docker rm -f webserver

# Then re-run your command
docker run --name webserver nginx
```

---

## Summary

- Docker uses a client-server architecture — the Docker CLI is the client, the Docker daemon is the server
- By default on Linux, they communicate through a Unix socket at `/var/run/docker.sock`
- The daemon can also be configured to accept remote connections over TCP with TLS for secure communication
- Mounting the Docker socket into a CI container is a common pattern for building Docker images in pipelines, but comes with security implications
- Docker contexts let you manage connections to multiple Docker daemons cleanly, including remote servers and AWS ECS
- Live restore (`"live-restore": true`) is an important production setting that prevents container interruption during daemon restarts

In the next file, we cover Docker registries — where Docker images are stored, how to pull from and push to them, and how AWS ECR fits into your production Docker workflow.
