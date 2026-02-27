# Docker Logging

## Introduction

Container logging is fundamentally different from traditional server logging. Containers are ephemeral — when a container is removed, all its local filesystem data, including log files, is gone. If logs only exist inside the container, you lose them when the container dies.

The correct pattern is to write logs to stdout and stderr and let Docker's logging driver handle collection, routing, and storage. This makes logs accessible without entering the container, enables centralized log aggregation, and survives container restarts and replacements.

---

## The Twelve-Factor App Logging Principle

The Twelve-Factor App methodology states: **Treat logs as event streams. Write to stdout.**

Applications should not manage log files. They should write structured events to standard output. The execution environment (Docker, ECS, Kubernetes) captures that stream and routes it to wherever logs need to be stored.

```python
# WRONG — writing to a file inside the container
import logging
logging.basicConfig(filename='/app/logs/app.log', level=logging.INFO)

# CORRECT — writing to stdout
import logging
import sys
logging.basicConfig(stream=sys.stdout, level=logging.INFO)
```

```bash
# Application writes to stdout → Docker captures it → available via docker logs
docker logs myapp_api_1
docker logs -f myapp_api_1      # follow in real time
docker logs --tail 100 myapp_api_1
docker logs --since 30m myapp_api_1
```

---

## Docker Logging Drivers

Docker captures stdout/stderr and routes it through a logging driver. The driver determines where the logs go.

```bash
# Check the default logging driver
docker info --format '{{.LoggingDriver}}'
# json-file

# Check a container's logging driver
docker inspect myapp_api_1 --format '{{.HostConfig.LogConfig.Type}}'
```

### Available Logging Drivers

| Driver | Description | Best For |
|---|---|---|
| `json-file` | Default — writes JSON to host filesystem | Development, default |
| `local` | Optimized binary format, smaller footprint | Production on-host |
| `journald` | Sends to systemd journal | systemd-based hosts |
| `syslog` | Sends to syslog daemon | Enterprise syslog stacks |
| `fluentd` | Sends to Fluentd/Fluent Bit | ECS, Kubernetes log aggregation |
| `awslogs` | Sends directly to CloudWatch Logs | AWS ECS on EC2 |
| `splunk` | Sends to Splunk HTTP Event Collector | Splunk environments |
| `gelf` | Sends to Graylog GELF | Graylog |
| `none` | Discards all logs | Batch jobs, test containers |

---

## json-file Driver — Default Behavior and Configuration

The default `json-file` driver writes logs to the host at:
```
/var/lib/docker/containers/<container-id>/<container-id>-json.log
```

Without size limits, these files grow indefinitely and fill the host disk.

```json
// /etc/docker/daemon.json — set global defaults for all containers
{
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "100m",    // rotate when file reaches 100MB
        "max-file": "5",       // keep last 5 rotated files
        "compress": "true"     // gzip old log files
    }
}
```

Or per-container in Docker Compose:

```yaml
services:
  api:
    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "3"
        compress: "true"

  worker:
    logging:
      driver: local       # local driver — more efficient, still host-local
      options:
        max-size: "50m"
        max-file: "3"
```

---

## AWS CloudWatch Logs — awslogs Driver

For ECS on EC2, the `awslogs` driver sends container logs directly to CloudWatch Logs — no sidecar or agent needed.

```yaml
# Docker Compose
services:
  api:
    logging:
      driver: awslogs
      options:
        awslogs-group: /myapp/production     # CloudWatch log group
        awslogs-region: us-east-1
        awslogs-stream-prefix: api           # prefix for stream name
        awslogs-create-group: "true"         # create group if not exists
        awslogs-datetime-format: "%Y-%m-%dT%H:%M:%S"
```

```json
// ECS task definition — logConfiguration
{
    "containerDefinitions": [
        {
            "name": "api",
            "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                    "awslogs-group": "/myapp/production",
                    "awslogs-region": "us-east-1",
                    "awslogs-stream-prefix": "api",
                    "awslogs-create-group": "true"
                }
            }
        }
    ]
}
```

The container's IAM role (via task execution role) must have:

```json
{
    "Effect": "Allow",
    "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogStreams"
    ],
    "Resource": "arn:aws:logs:*:*:*"
}
```

---

## Structured Logging (JSON Logs)

Applications that write JSON to stdout produce logs that log aggregation tools (CloudWatch Logs Insights, Elasticsearch, Splunk) can automatically parse, filter, and query.

```python
# Python — structured JSON logging
import logging
import sys
import json

class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_record = {
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "message": record.getMessage(),
            "module": record.module,
            "function": record.funcName,
        }
        if record.exc_info:
            log_record["exception"] = self.formatException(record.exc_info)
        return json.dumps(log_record)

handler = logging.StreamHandler(sys.stdout)
handler.setFormatter(JSONFormatter())
logging.root.addHandler(handler)
logging.root.setLevel(logging.INFO)

# Usage
logging.info("User logged in", extra={"user_id": 12345, "ip": "192.168.1.1"})
# {"timestamp": "2024-01-15 09:23:41", "level": "INFO", "message": "User logged in", ...}
```

```javascript
// Node.js — pino (fastest structured logger)
const pino = require('pino');
const logger = pino({ level: 'info' });

logger.info({ userId: 12345, event: 'login' }, 'User logged in');
// {"level":30,"time":1705312221,"msg":"User logged in","userId":12345,"event":"login"}
```

---

## AWS CloudWatch Logs Insights — Querying Logs

With JSON logs in CloudWatch, you can query them with SQL-like syntax:

```sql
-- Find all errors in the last hour
fields @timestamp, level, message, module
| filter level = "ERROR"
| sort @timestamp desc
| limit 50

-- Count errors by module
fields module
| filter level = "ERROR"
| stats count() as error_count by module
| sort error_count desc

-- Find requests slower than 500ms
fields @timestamp, message, duration_ms, path
| filter duration_ms > 500
| sort duration_ms desc
| limit 100
```

---

## Fluent Bit — Log Routing Sidecar (ECS)

AWS Fargate does not support the `awslogs` driver by default for custom log routing. Instead, use Fluent Bit as a sidecar container that collects logs from other containers and routes them to CloudWatch, S3, Kinesis, or Elasticsearch.

```json
// ECS task definition with Fluent Bit sidecar
{
    "containerDefinitions": [
        {
            "name": "api",
            "logConfiguration": {
                "logDriver": "awsfirelens",
                "options": {
                    "Name": "cloudwatch",
                    "region": "us-east-1",
                    "log_group_name": "/myapp/production",
                    "log_stream_prefix": "api/"
                }
            }
        },
        {
            "name": "log-router",
            "image": "public.ecr.aws/aws-observability/aws-for-fluent-bit:stable",
            "essential": true,
            "firelensConfiguration": {
                "type": "fluentbit"
            },
            "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                    "awslogs-group": "/myapp/firelens",
                    "awslogs-region": "us-east-1",
                    "awslogs-stream-prefix": "fluent-bit"
                }
            }
        }
    ]
}
```

---

## Querying Logs Locally

```bash
# Get logs from a running container
docker logs myapp_api_1
docker logs -f myapp_api_1                  # follow
docker logs --tail 100 myapp_api_1          # last 100 lines
docker logs --since 30m myapp_api_1         # last 30 minutes
docker logs -t myapp_api_1                  # include timestamps

# Get logs from Docker Compose services
docker compose logs
docker compose logs -f api db               # follow specific services
docker compose logs --tail 50 api

# Grep across container logs
docker logs myapp_api_1 2>&1 | grep ERROR
docker logs myapp_api_1 2>&1 | grep -v "health check"   # exclude health noise
```

---

## Summary

- Write logs to stdout and stderr — let Docker's logging driver handle collection and routing; never write log files inside containers
- The default `json-file` driver writes to the host disk — always configure `max-size` and `max-file` to prevent disk exhaustion
- The `awslogs` driver sends logs directly to CloudWatch Logs — the standard approach for ECS on EC2; requires the task execution role to have CloudWatch Logs permissions
- Use structured JSON logging in your application — enables SQL-like querying in CloudWatch Logs Insights, Elasticsearch, and Splunk without custom parsing
- Fluent Bit (AWS FireLens) is the sidecar pattern for Fargate — routes logs to multiple destinations (CloudWatch, S3, Kinesis) with filtering and transformation
- Always suppress health check logs from your aggregated log stream — they generate enormous volume with zero diagnostic value; filter them in Fluent Bit or at the application level

In the next file, we cover Docker monitoring — metrics collection, container health tracking, and integration with Prometheus and CloudWatch.
