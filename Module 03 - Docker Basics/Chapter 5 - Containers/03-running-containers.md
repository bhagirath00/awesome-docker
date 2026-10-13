# Running Containers

## Introduction

Running containers is the most frequent task you will do with Docker. You already know the basic `docker run` command. This file goes much deeper — covering every practical pattern you need for real DevOps work: running services, running tasks, passing configuration, handling resource limits, and debugging.

By the end of this file, you will be comfortable running containers in any scenario you encounter on the job.

---

## docker run — The Full Picture

The `docker run` command creates a new container from an image and starts it. Its full syntax is:

```
docker run [OPTIONS] IMAGE[:TAG] [COMMAND] [ARGS]
```

Every part matters:

- **OPTIONS** — flags that control how the container runs (`-d`, `-p`, `-e`, `-v`, etc.)
- **IMAGE** — the image to create the container from (pulled from registry if not local)
- **TAG** — the image version (defaults to `latest` if not specified — always specify in production)
- **COMMAND** — overrides the default command defined in the image
- **ARGS** — arguments passed to the command

```bash
# Full example — all parts
docker run -d -p 8080:80 --name webserver nginx:1.25.3

# Override command — run bash instead of nginx
docker run -it nginx:1.25.3 bash

# Override command with args
docker run --rm alpine:3.18 ping -c 4 google.com
```

---

## Running Background Services

Most production containers run in the background — web servers, API services, databases, message queues.

Use `-d` (detach) to run them in the background:

```bash
# Run nginx in the background
docker run -d --name webserver -p 8080:80 nginx

# Run a PostgreSQL database
docker run -d \
  --name postgres \
  -e POSTGRES_USER=myuser \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -e POSTGRES_DB=myapp \
  -p 5432:5432 \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16

# Run Redis
docker run -d \
  --name redis \
  -p 6379:6379 \
  redis:7-alpine

# Verify all three are running
docker ps
```

---

## Running Interactive Containers

Use `-it` together to get an interactive shell. This is essential for debugging.

- `-i` keeps stdin open (allows you to type)
- `-t` allocates a pseudo-TTY (gives you a proper terminal experience)

```bash
# Open a shell in an Ubuntu container
docker run -it ubuntu:22.04 bash

# Open a shell in an Alpine container (sh, not bash)
docker run -it alpine:3.18 sh

# Open a Python interpreter
docker run -it python:3.11 python

# Test something in a specific environment without installing it locally
docker run -it node:20-alpine node
```

When you exit the shell, the container stops (because the shell was PID 1).

---

## Running One-Off Task Containers

Some containers are not services — they run a task and exit. Use `--rm` to clean them up automatically.

```bash
# Run a database migration
docker run --rm \
  -e DATABASE_URL=postgresql://user:pass@db:5432/myapp \
  myapp:1.0 \
  python manage.py migrate

# Run tests
docker run --rm \
  -e TEST_ENV=true \
  myapp:1.0 \
  python -m pytest tests/

# Run a linter
docker run --rm \
  -v $(pwd):/code \
  -w /code \
  python:3.11 \
  pip install flake8 && flake8 .

# Use a tool without installing it locally
docker run --rm \
  -v $(pwd):/data \
  -w /data \
  hashicorp/terraform:1.6.6 \
  terraform init

# Convert a file using ImageMagick
docker run --rm \
  -v $(pwd):/images \
  -w /images \
  dpokidov/imagemagick \
  convert input.png output.jpg
```

This pattern is powerful — you use tools without installing them on your machine. Different projects can use different tool versions without conflict.

---

## Passing Configuration to Containers

The correct way to pass configuration to containers is through environment variables — not baking secrets into images.

### Using -e for Individual Variables

```bash
docker run -d \
  --name api \
  -e APP_ENV=production \
  -e LOG_LEVEL=info \
  -e DATABASE_HOST=db.example.com \
  -e DATABASE_PORT=5432 \
  -e DATABASE_NAME=myapp \
  myapp:1.0
```

### Using --env-file for Many Variables

For many variables, using an env file is cleaner:

```bash
# Create a .env file (never commit this to Git)
cat > .env << 'EOF'
APP_ENV=production
LOG_LEVEL=info
DATABASE_HOST=db.example.com
DATABASE_PORT=5432
DATABASE_NAME=myapp
DATABASE_USER=apiuser
DATABASE_PASSWORD=supersecret
JWT_SECRET=anothersecretvalue
AWS_REGION=us-east-1
EOF

# Pass the env file to the container
docker run -d \
  --name api \
  --env-file .env \
  myapp:1.0
```

**Important:** Never put your `.env` file in version control. Add it to `.gitignore`.

### Using AWS Secrets Manager (Production Pattern)

In production on AWS, secrets should come from AWS Secrets Manager, not environment variables in your deploy command. Your application code fetches the secrets at startup:

```python
# Inside your application — not in docker run
import boto3
import json

def get_secret(secret_name, region="us-east-1"):
    client = boto3.client("secretsmanager", region_name=region)
    response = client.get_secret_value(SecretId=secret_name)
    return json.loads(response["SecretString"])

secrets = get_secret("myapp/production/db")
DATABASE_PASSWORD = secrets["password"]
```

---

## Port Mapping

Port mapping connects a port on the host to a port inside the container.

```bash
# Map host port 8080 to container port 80
docker run -p 8080:80 nginx

# Map multiple ports
docker run -p 8080:80 -p 8443:443 nginx

# Map to a specific host interface (only accessible from localhost, not other machines)
docker run -p 127.0.0.1:8080:80 nginx

# Let Docker pick a random available host port
docker run -p 80 nginx

# Find the assigned port
docker port nginx_container_name 80
# 0.0.0.0:32768
```

In production on AWS ECS, port mapping is handled by the task definition — the container listens on a port and ECS maps it to the load balancer. You specify ports in your ECS task definition, not in the docker run command.

---

## Working Directory and Volume Mounts

### Setting the Working Directory

```bash
# Set the directory that commands run from inside the container
docker run -w /app \
  -v $(pwd):/app \
  python:3.11 \
  python app.py
```

### Bind Mounts — Mount a Host Directory

```bash
# Mount current directory into /app in the container
docker run -v $(pwd):/app myapp:1.0

# Mount a specific host path (read-only)
docker run -v /etc/ssl/certs:/certs:ro myapp:1.0

# Mount multiple paths
docker run \
  -v $(pwd)/config:/app/config \
  -v $(pwd)/logs:/app/logs \
  myapp:1.0
```

Bind mounts are useful in development — changes you make to files on the host immediately appear inside the container. This means you can edit code on your machine and see the changes without rebuilding the image.

### Named Volumes — Managed by Docker

```bash
# Create and use a named volume
docker run -v myapp-data:/app/data myapp:1.0

# The volume persists even when the container is removed
docker rm -f myapp-container
# Data in myapp-data volume is safe

# Run a new container with the same volume — data is still there
docker run -v myapp-data:/app/data myapp:1.0
```

---

## Resource Limits

Always set resource limits for production containers. Without limits, a misbehaving container can consume all host resources and bring down everything else.

```bash
# Memory limit — container cannot exceed 512MB RAM
docker run -d \
  --memory 512m \
  --memory-swap 512m \
  --name api \
  myapp:1.0

# CPU limit — container gets at most 1 full CPU core
docker run -d \
  --cpus 1.0 \
  --name api \
  myapp:1.0

# Both together — realistic production limits
docker run -d \
  --name api \
  --memory 512m \
  --memory-swap 512m \
  --cpus 1.0 \
  --restart unless-stopped \
  -p 8080:8080 \
  myapp:1.0
```

### Understanding Memory Limits

- `--memory 512m` — the container can use up to 512MB of RAM
- `--memory-swap 512m` — set this equal to `--memory` to disable swap (recommended for predictable behavior)
- If the container exceeds the memory limit, it is killed with exit code 137 (OOM killed)

```bash
# Detect OOM kills
docker inspect --format '{{.State.OOMKilled}}' api
# true  ← this means the container was killed due to memory limit
```

On AWS ECS, resource limits are set in the task definition. ECS uses them to schedule containers on instances that have enough available resources.

---

## Naming and Labels

### Naming Containers

```bash
# Without a name — Docker generates a random one
docker run -d nginx
# e.g., "mystifying_curie"

# With a name — always use names in production
docker run -d --name webserver nginx
```

Named containers are much easier to manage:

```bash
docker stop webserver
docker logs webserver
docker exec -it webserver bash
docker inspect webserver
```

### Adding Labels

Labels are key-value metadata you can attach to containers:

```bash
docker run -d \
  --name api \
  --label app=myapp \
  --label environment=production \
  --label version=1.4.2 \
  --label team=backend \
  myapp:1.0

# Filter containers by label
docker ps --filter label=environment=production

# Inspect a container's labels
docker inspect --format '{{json .Config.Labels}}' api
```

Labels are useful for:
- Filtering and querying containers in scripts
- Integrating with monitoring tools (Prometheus, Datadog)
- Organizing containers in large environments

---

## A Complete Production-Ready docker run Command

Putting it all together — a realistic `docker run` command for a production API service:

```bash
docker run -d \
  --name api-server \
  --restart unless-stopped \
  -p 8080:8080 \
  --memory 1g \
  --memory-swap 1g \
  --cpus 2.0 \
  -e APP_ENV=production \
  -e LOG_LEVEL=info \
  -e DATABASE_URL=postgresql://user:pass@db.example.com:5432/myapp \
  -v /var/log/myapp:/app/logs \
  --label app=myapp \
  --label environment=production \
  --label version=1.4.2 \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2
```

This command:
- Runs in the background (`-d`)
- Has a meaningful name (`--name`)
- Restarts automatically unless manually stopped
- Maps port 8080
- Has 1GB memory limit and 2 CPU cores
- Has environment variables for configuration
- Mounts a host directory for persistent logs
- Has labels for organization and monitoring
- Uses a specific tagged image from AWS ECR (never `latest` in production)

---

## Summary

- `docker run` creates and starts a container from an image — use `-d` for background services and `-it` for interactive shells
- Task containers should use `--rm` to clean up automatically after they exit
- Pass configuration through environment variables (`-e`) or env files (`--env-file`) — never bake secrets into images
- Use port mapping (`-p host:container`) to expose container services to the host
- Always set memory and CPU limits in production to prevent runaway containers from affecting the host
- Use named volumes for persistent data and bind mounts for development workflows
- Always use named containers (`--name`) and specific image tags — never `latest` in production

In the next file, we cover the most important container commands in depth — inspect, logs, exec, and more — the tools you use every day to manage and debug running containers.
