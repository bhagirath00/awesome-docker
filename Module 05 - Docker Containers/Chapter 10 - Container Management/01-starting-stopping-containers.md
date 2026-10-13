# Starting and Stopping Containers

## Introduction

Starting and stopping containers is something you do dozens of times per day. But there is much more to it than just `docker start` and `docker stop`. How you start a container determines its behavior, resource usage, and how it integrates with the rest of your system. How you stop it determines whether your application shuts down cleanly or loses in-flight work.

This file covers the complete picture — every option for starting containers, the correct way to stop them, and the patterns used in real production environments on AWS.

---

## Starting Containers

### docker run — Create and Start

`docker run` creates a new container from an image and starts it immediately. This is the most common way to start a container.

```bash
# Basic — creates and starts, attaches to output, blocks your terminal
docker run nginx

# Detached — runs in background, returns container ID
docker run -d nginx

# Detached with name and port mapping
docker run -d --name webserver -p 8080:80 nginx

# Detached with all production options
docker run -d \
  --name api \
  --restart unless-stopped \
  -p 8080:8080 \
  --memory 512m \
  --cpus 1.0 \
  -e APP_ENV=production \
  -e DATABASE_URL=postgresql://user:pass@db:5432/myapp \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2
```

### docker start — Start an Existing Stopped Container

`docker start` starts a container that already exists but is stopped. The container was previously created with `docker run` or `docker create`.

```bash
# Start a stopped container
docker start webserver

# Start multiple containers
docker start webserver api database

# Start and attach to output
docker start -a webserver

# Start interactively
docker start -ai webserver
```

Unlike `docker run`, `docker start` does not let you change container configuration. The ports, environment variables, and volumes are fixed from when the container was created. If you need to change configuration, you must remove and recreate the container.

### docker create — Create Without Starting

`docker create` creates a container but does not start it. Useful when you need to copy files into the container before it starts.

```bash
# Create without starting
docker create --name webserver -p 8080:80 nginx

# Copy a config file into the created container
docker cp ./nginx.conf webserver:/etc/nginx/nginx.conf

# Now start it with the custom config
docker start webserver
```

---

## Stopping Containers

### docker stop — Graceful Shutdown

`docker stop` sends `SIGTERM` to the container's main process (PID 1). This gives the process a chance to shut down cleanly — finish in-flight requests, flush buffers, close database connections, write pending data.

After a configurable grace period (default 10 seconds), if the process has not exited, Docker sends `SIGKILL` to force termination.

```bash
# Stop with default 10-second grace period
docker stop webserver

# Stop with a longer grace period (let the app finish its work)
docker stop --time 30 webserver

# Stop multiple containers
docker stop webserver api database

# Stop all running containers
docker stop $(docker ps -q)
```

### When to Increase the Grace Period

The default 10-second grace period is often too short for:
- Web servers finishing long-running requests
- Workers processing a job that should not be interrupted
- Applications flushing large write buffers

```bash
# Give a database container 60 seconds to finish writes and flush WAL
docker stop --time 60 postgres

# Give a message consumer 30 seconds to finish processing the current message
docker stop --time 30 consumer
```

### Your Application Must Handle SIGTERM

For graceful shutdown to work, your application must listen for `SIGTERM` and shut down cleanly when it receives it.

**Python Flask with Gunicorn:**

Gunicorn handles SIGTERM automatically. It finishes in-flight requests and exits.

```bash
docker run -d \
  --name api \
  --stop-timeout 30 \
  myapp:1.0 \
  gunicorn --workers 4 --timeout 25 app:app
```

**Node.js:**

```javascript
// Handle SIGTERM in your Node.js application
process.on('SIGTERM', () => {
  console.log('SIGTERM received. Starting graceful shutdown...');
  
  server.close(() => {
    console.log('HTTP server closed');
    db.end(() => {
      console.log('Database connection closed');
      process.exit(0);
    });
  });
  
  // Force exit after 25 seconds if server.close() doesn't complete
  setTimeout(() => {
    console.error('Forcing shutdown after timeout');
    process.exit(1);
  }, 25000);
});
```

**Python (general):**

```python
import signal
import sys

def handle_sigterm(signum, frame):
    print("SIGTERM received. Shutting down gracefully...")
    # Close database connections
    db.close()
    # Finish any pending work
    cleanup()
    sys.exit(0)

signal.signal(signal.SIGTERM, handle_sigterm)
```

### docker kill — Immediate Termination

`docker kill` sends `SIGKILL` directly — the process is terminated immediately with no opportunity to clean up.

```bash
# Immediate termination
docker kill webserver

# Send a specific signal instead of SIGKILL
docker kill --signal SIGINT webserver
docker kill --signal SIGUSR1 webserver  # some apps use SIGUSR1 to reload config
```

Only use `docker kill` when `docker stop` is not working — the container is frozen or not responding. Killing without warning can cause data loss, incomplete transactions, and corrupt state.

---

## Restarting Containers

```bash
# Restart a running or stopped container
docker restart webserver

# Restart with a custom stop timeout
docker restart --time 30 webserver

# Restart multiple containers
docker restart webserver api database
```

Restarting is equivalent to `docker stop` followed by `docker start`. The container configuration is preserved — same ports, volumes, environment variables.

Use cases:
- Apply a new configuration file that was mounted via volume (app reads config on startup)
- Recover a container that is running but in a degraded state
- Force a connection pool to re-establish database connections after a database failover

---

## Pausing and Unpausing

Pausing a container freezes all its processes while keeping the container's state in memory.

```bash
# Freeze all processes in the container
docker pause webserver

# Resume — container continues exactly where it left off
docker unpause webserver
```

Use cases:
- Temporarily free CPU on an overloaded host
- Take a consistent snapshot of a container's in-memory state
- Pause a consumer while performing maintenance on the queue it reads from

Paused containers still consume memory — they just use no CPU.

---

## Monitoring Start and Stop Events

Docker emits events for container state changes. You can monitor them in real time:

```bash
# Watch all Docker events
docker events

# Watch events for a specific container
docker events --filter container=webserver

# Watch only start and stop events
docker events --filter event=start --filter event=stop --filter event=die

# Output:
2024-01-15T09:23:41.123Z container start a1b2c3d4 (image=nginx:1.25.3, name=webserver)
2024-01-15T10:45:12.456Z container stop a1b2c3d4 (image=nginx:1.25.3, name=webserver)
2024-01-15T10:45:13.789Z container die a1b2c3d4 (exitCode=0, image=nginx:1.25.3, name=webserver)
```

This is useful for debugging unexpected restarts and understanding your container's history.

---

## Production Patterns

### Pattern 1: Systemd-Managed Docker Container

On a standalone EC2 instance, you can manage a Docker container with systemd so it starts on boot and restarts on failure:

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=MyApp Container
After=docker.service
Requires=docker.service

[Service]
TimeoutStartSec=0
Restart=always
ExecStartPre=-/usr/bin/docker stop myapp
ExecStartPre=-/usr/bin/docker rm myapp
ExecStartPre=/usr/bin/docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
ExecStart=/usr/bin/docker run \
  --name myapp \
  -p 8080:8080 \
  --memory 512m \
  --cpus 1.0 \
  -e APP_ENV=production \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
ExecStop=/usr/bin/docker stop myapp

[Install]
WantedBy=multi-user.target
```

```bash
# Enable and start the service
sudo systemctl daemon-reload
sudo systemctl enable myapp
sudo systemctl start myapp

# Check status
sudo systemctl status myapp
```

### Pattern 2: AWS ECS (Production Standard)

On AWS ECS, you define how containers start, stop, and recover in a task definition:

```json
{
  "family": "myapp-task",
  "containerDefinitions": [
    {
      "name": "api",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 8080,
          "protocol": "tcp"
        }
      ],
      "stopTimeout": 30,
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
        "interval": 30,
        "timeout": 10,
        "retries": 3,
        "startPeriod": 60
      },
      "environment": [
        {"name": "APP_ENV", "value": "production"}
      ]
    }
  ]
}
```

ECS handles starting, stopping, health checking, and restarting containers automatically based on this definition.

---

## Summary

- `docker run -d` creates and starts a container in the background — use this for services
- `docker start` starts an existing stopped container — configuration cannot be changed
- `docker stop` sends `SIGTERM` for graceful shutdown — your application must handle this signal
- Increase the stop timeout (`--time 30`) for applications that need more time to finish in-flight work
- `docker kill` sends `SIGKILL` for immediate termination — only use when `docker stop` does not work
- `docker restart` combines stop and start — useful for applying new configuration
- `docker pause` and `docker unpause` freeze and resume processes without stopping the container
- In production, ECS manages container start/stop with defined health checks and stop timeouts — prefer ECS over manually running containers on EC2

In the next file, we cover container logs in depth — reading, following, filtering, and routing logs to AWS CloudWatch.
