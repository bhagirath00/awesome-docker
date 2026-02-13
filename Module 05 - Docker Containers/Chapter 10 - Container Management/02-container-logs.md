# Container Logs

## Introduction

Container logs are your primary window into what is happening inside a running container. When a service is returning errors, when a container keeps restarting, when performance degrades — you go to the logs first.

Docker captures everything your container's processes write to stdout and stderr and makes it available through the `docker logs` command and configurable log drivers. In production on AWS, these logs flow to CloudWatch Logs where they are searchable, retained, and can trigger alerts.

This file covers reading logs locally, filtering them efficiently, configuring log drivers, and setting up the production logging pipeline to AWS CloudWatch.

---

## How Container Logging Works

Docker captures stdout and stderr from the container's main process (PID 1) and all processes spawned by it. This is automatic — your application does not need to do anything special except write to stdout and stderr.

```python
# Python — write to stdout (automatically captured by Docker)
print("Server started on port 8080")
import logging
logging.basicConfig(level=logging.INFO)
logging.info("Database connection established")

# Write to stderr (also captured)
import sys
print("ERROR: Connection failed", file=sys.stderr)
```

```javascript
// Node.js — stdout and stderr automatically captured
console.log("Server started");       // stdout
console.error("Connection failed");  // stderr
```

By default, Docker stores logs as JSON files on the host:

```bash
# Where Docker stores log files on Linux
ls /var/lib/docker/containers/<container_id>/
# <container_id>-json.log   ← the log file
```

Each log entry is a JSON object with the log message, stream (stdout/stderr), and timestamp.

---

## Reading Logs with docker logs

### Basic Log Viewing

```bash
# View all logs
docker logs webserver

# View logs for a container with a complex name
docker logs my-api-container

# View logs by container ID
docker logs a1b2c3d4ef56
```

### Following Logs in Real Time

```bash
# Follow logs as they are written (like tail -f)
docker logs -f webserver

# Press Ctrl+C to stop following
```

### Filtering by Time

```bash
# Show logs since a specific timestamp (RFC 3339 format)
docker logs --since 2024-01-15T09:00:00 webserver

# Show logs since a relative time
docker logs --since 30m webserver        # last 30 minutes
docker logs --since 2h webserver         # last 2 hours
docker logs --since 1d webserver         # last 24 hours

# Show logs until a specific time
docker logs --until 2024-01-15T10:00:00 webserver

# Time range — from 9am to 10am
docker logs --since 2024-01-15T09:00:00 --until 2024-01-15T10:00:00 webserver
```

### Limiting Output

```bash
# Show only the last N lines (like tail -n)
docker logs --tail 100 webserver

# Show last 50 lines and follow
docker logs --tail 50 -f webserver

# Show all logs from the beginning
docker logs webserver
```

### Showing Timestamps

```bash
# Add Docker's timestamp to each log line
docker logs -t webserver

# Output:
2024-01-15T09:23:41.123456789Z server started on port 8080
2024-01-15T09:23:42.456789012Z database connection established
```

### Combining Options — Most Useful Patterns

```bash
# Last 100 lines with timestamps, followed live
docker logs --tail 100 -t -f webserver

# Last 30 minutes of logs with timestamps
docker logs --since 30m -t webserver

# Last 200 lines since 1 hour ago, followed
docker logs --tail 200 --since 1h -f api
```

### Filtering Log Content

Docker does not have built-in grep — pipe to grep:

```bash
# Filter for errors
docker logs webserver 2>&1 | grep -i error

# Filter for a specific request path
docker logs api 2>&1 | grep "/api/users"

# Show only lines with status 500
docker logs api 2>&1 | grep " 500 "

# Case-insensitive search for warnings
docker logs webserver 2>&1 | grep -i "warn\|error\|critical"
```

Note: `2>&1` redirects stderr to stdout so you can grep both streams together.

---

## Logs from Exited Containers

`docker logs` works on stopped containers too — as long as the container has not been removed:

```bash
# Container crashed — logs are still accessible
docker ps -a | grep crashed-app
# crashed-app   Exited (1) 2 minutes ago

# Read the logs to find out why
docker logs crashed-app

# Get the last 200 lines
docker logs --tail 200 crashed-app
```

This is critical for post-mortem debugging. Always read logs before removing a failed container.

---

## Log Drivers

By default, Docker uses the `json-file` log driver which stores logs as JSON files on disk. Docker supports many other drivers that route logs to different destinations.

| Driver | Destination |
|---|---|
| `json-file` | JSON files on host disk (default) |
| `awslogs` | AWS CloudWatch Logs |
| `syslog` | System syslog daemon |
| `journald` | systemd journal |
| `fluentd` | Fluentd log aggregator |
| `splunk` | Splunk |
| `gelf` | Graylog |
| `none` | Discard all logs |

### Configuring the Default Log Driver

Set the default log driver for all containers in `/etc/docker/daemon.json`:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "3"
  }
}
```

`max-size` and `max-file` enable log rotation — essential in production. Without rotation, log files grow until they fill your disk.

Restart Docker to apply:

```bash
sudo systemctl restart docker
```

### Setting Log Driver Per Container

```bash
# Use json-file with rotation for this container
docker run -d \
  --name api \
  --log-driver json-file \
  --log-opt max-size=50m \
  --log-opt max-file=3 \
  myapp:1.0

# Discard all logs (for very high-throughput containers where you use another logging mechanism)
docker run -d \
  --name high-volume-processor \
  --log-driver none \
  myapp:1.0
```

---

## AWS CloudWatch Logs — Production Logging

In production on AWS, container logs should go to **CloudWatch Logs** where they are:
- Retained beyond the container's lifetime
- Searchable across all instances
- Available for metric extraction and alerting
- Accessible from the AWS console

### Using the awslogs Driver on EC2

```bash
# Run a container that sends logs to CloudWatch
docker run -d \
  --name api \
  --log-driver awslogs \
  --log-opt awslogs-region=us-east-1 \
  --log-opt awslogs-group=/myapp/production \
  --log-opt awslogs-stream=api-server-1 \
  --log-opt awslogs-create-group=true \
  myapp:1.0
```

The EC2 instance must have an IAM role with the `logs:CreateLogGroup`, `logs:CreateLogStream`, and `logs:PutLogEvents` permissions.

### IAM Policy for CloudWatch Logging

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogStreams"
      ],
      "Resource": "arn:aws:logs:us-east-1:123456789012:log-group:/myapp/*"
    }
  ]
}
```

### CloudWatch Logging in ECS Task Definitions

On AWS ECS, configure CloudWatch logging in the task definition — this is the standard approach:

```json
{
  "family": "myapp-task",
  "containerDefinitions": [
    {
      "name": "api",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2",
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/myapp-production",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "api",
          "awslogs-create-group": "true"
        }
      }
    }
  ]
}
```

With this configuration, every log line written to stdout or stderr by the container automatically appears in CloudWatch Logs under `/ecs/myapp-production/api/<task-id>`.

### Viewing CloudWatch Logs

```bash
# List log streams for a log group
aws logs describe-log-streams \
  --log-group-name /ecs/myapp-production \
  --region us-east-1

# Tail logs from a log stream
aws logs tail /ecs/myapp-production \
  --follow \
  --region us-east-1

# Search logs with CloudWatch Insights
aws logs start-query \
  --log-group-name /ecs/myapp-production \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc | limit 50' \
  --region us-east-1
```

---

## Structured Logging — Making Logs Searchable

Plain text logs are hard to query at scale. **Structured logging** (writing logs as JSON) makes them machine-readable and enables powerful CloudWatch Insights queries.

### Python Structured Logging

```python
import json
import logging
import sys
from datetime import datetime

class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_entry = {
            "timestamp": datetime.utcnow().isoformat() + "Z",
            "level": record.levelname,
            "message": record.getMessage(),
            "logger": record.name,
        }
        if record.exc_info:
            log_entry["exception"] = self.formatException(record.exc_info)
        return json.dumps(log_entry)

handler = logging.StreamHandler(sys.stdout)
handler.setFormatter(JSONFormatter())
logger = logging.getLogger()
logger.addHandler(handler)
logger.setLevel(logging.INFO)

# Usage:
logger.info("Request processed", extra={"user_id": 123, "duration_ms": 45})
```

Output:
```json
{"timestamp": "2024-01-15T09:23:41Z", "level": "INFO", "message": "Request processed", "logger": "root"}
```

With structured logs in CloudWatch Insights:

```
# Find all errors for a specific user
fields @timestamp, message, user_id
| filter level = "ERROR" and user_id = 123
| sort @timestamp desc
```

---

## Summary

- Docker captures stdout and stderr from all container processes automatically — your app just needs to write to these streams
- `docker logs -f --tail 100 CONTAINER` is your most-used command for watching logs in real time
- Use `--since 30m`, `--until`, and `--tail` to filter the volume of logs to what you need
- Always configure log rotation on the default `json-file` driver — `max-size=50m` and `max-file=3` prevents disk fill
- In production on AWS, use the `awslogs` log driver to route logs to CloudWatch Logs — logs persist beyond container lifetime and are searchable across all instances
- On ECS, configure `awslogs` in the task definition's `logConfiguration` — logs flow to CloudWatch automatically
- Use structured (JSON) logging to make logs machine-readable and enable powerful CloudWatch Insights queries

In the next file, we cover resource limits — how to set CPU and memory limits on containers and why this matters in production.
