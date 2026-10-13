# Docker Monitoring

## Introduction

Running containers in production without monitoring is flying blind. You cannot know whether your application is slow, whether a container is consuming excessive memory, whether the host is running out of disk space, or whether your deployment degraded performance — unless you have metrics.

Docker exposes container-level resource metrics (CPU, memory, network I/O, disk I/O) through its API. The ecosystem around collecting, aggregating, and alerting on these metrics is mature. This file covers local monitoring with built-in Docker tools, the Prometheus + Grafana stack for self-hosted monitoring, and AWS-native monitoring for ECS workloads.

---

## Built-in: docker stats

`docker stats` shows real-time resource usage for running containers — the `top` command for containers.

```bash
# Live stats for all running containers
docker stats

# Stats for specific containers
docker stats myapp_api_1 myapp_db_1

# Non-streaming snapshot (for scripting)
docker stats --no-stream

# Custom format
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}"

# Output:
# CONTAINER          CPU %    MEM USAGE / LIMIT    NET I/O
# myapp_api_1        12.45%   245MiB / 512MiB      12.1MB / 3.2MB
# myapp_db_1         0.23%    512MiB / 2GiB        5.4MB / 1.2MB
# myapp_redis_1      0.05%    32MiB / 256MiB       892kB / 745kB
```

`docker stats` is useful for immediate investigation but does not retain history. For trends and alerting, you need a metrics database.

---

## The Prometheus + Grafana Stack

[Prometheus](https://prometheus.io/) is the de facto metrics collection system for containerized workloads. It scrapes metrics from HTTP endpoints at regular intervals, stores them in a time-series database, and supports alerting rules.

[Grafana](https://grafana.com/) provides dashboards for visualizing Prometheus metrics.

### cAdvisor — Container Metrics Exporter

[cAdvisor](https://github.com/google/cadvisor) (Container Advisor) exposes Docker container metrics in Prometheus format. Deploy it as a sidecar container alongside your application:

```yaml
# docker-compose.monitoring.yml
version: "3.8"

services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.47.2
    restart: unless-stopped
    privileged: true       # needs access to /sys and cgroups
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    ports:
      - "8080:8080"
    devices:
      - /dev/kmsg
    networks:
      - monitoring

  prometheus:
    image: prom/prometheus:v2.48.0
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:10.2.0
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./monitoring/grafana/provisioning:/etc/grafana/provisioning:ro
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD:-admin}
      GF_USERS_ALLOW_SIGN_UP: "false"
    networks:
      - monitoring

  node-exporter:
    image: prom/node-exporter:v1.7.0
    restart: unless-stopped
    network_mode: host     # needs access to host metrics
    pid: host
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'

volumes:
  prometheus-data:
  grafana-data:

networks:
  monitoring:
    driver: bridge
```

### Prometheus Configuration

```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alerts.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

scrape_configs:
  # Docker host metrics
  - job_name: node-exporter
    static_configs:
      - targets: ['node-exporter:9100']

  # Container metrics
  - job_name: cadvisor
    static_configs:
      - targets: ['cadvisor:8080']

  # Application metrics (if your app exposes /metrics)
  - job_name: myapp-api
    static_configs:
      - targets: ['api:8080']
    metrics_path: /metrics

  # Prometheus self-monitoring
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']
```

### Key Container Metrics from cAdvisor

```promql
# CPU usage percentage per container
rate(container_cpu_usage_seconds_total{name!=""}[5m]) * 100

# Memory usage per container
container_memory_usage_bytes{name!=""}

# Memory limit per container
container_spec_memory_limit_bytes{name!=""}

# Memory usage as percentage of limit
container_memory_usage_bytes{name!=""} / container_spec_memory_limit_bytes{name!=""} * 100

# Network bytes received per container
rate(container_network_receive_bytes_total{name!=""}[5m])

# Container restart count
increase(container_restart_count{name!=""}[1h])
```

### Alert Rules

```yaml
# monitoring/alerts.yml
groups:
  - name: container_alerts
    rules:
      - alert: ContainerHighMemory
        expr: container_memory_usage_bytes{name!=""} / container_spec_memory_limit_bytes{name!=""} > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Container {{ $labels.name }} memory is above 85%"
          description: "Memory: {{ $value | humanizePercentage }}"

      - alert: ContainerRestarting
        expr: increase(container_restart_count{name!=""}[1h]) > 3
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "Container {{ $labels.name }} has restarted {{ $value }} times in the last hour"

      - alert: ContainerHighCPU
        expr: rate(container_cpu_usage_seconds_total{name!=""}[5m]) * 100 > 80
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Container {{ $labels.name }} CPU above 80% for 10 minutes"
```

---

## Application Metrics — Exposing /metrics

For full observability, your application should expose its own metrics in Prometheus format:

```python
# Python — prometheus_client
from prometheus_client import Counter, Histogram, Gauge, start_http_server
import time

REQUEST_COUNT = Counter('http_requests_total', 'Total HTTP requests', ['method', 'endpoint', 'status'])
REQUEST_LATENCY = Histogram('http_request_duration_seconds', 'Request latency', ['endpoint'])
ACTIVE_REQUESTS = Gauge('http_requests_active', 'Currently processing requests')

# Middleware to record metrics
@app.before_request
def before_request():
    ACTIVE_REQUESTS.inc()
    flask.g.start_time = time.time()

@app.after_request
def after_request(response):
    ACTIVE_REQUESTS.dec()
    duration = time.time() - flask.g.start_time
    REQUEST_COUNT.labels(
        method=flask.request.method,
        endpoint=flask.request.path,
        status=response.status_code
    ).inc()
    REQUEST_LATENCY.labels(endpoint=flask.request.path).observe(duration)
    return response
```

```javascript
// Node.js — prom-client
const promClient = require('prom-client');

const httpRequestDuration = new promClient.Histogram({
    name: 'http_request_duration_seconds',
    help: 'Duration of HTTP requests in seconds',
    labelNames: ['method', 'route', 'status_code'],
    buckets: [0.01, 0.05, 0.1, 0.5, 1, 2, 5]
});

app.get('/metrics', async (req, res) => {
    res.set('Content-Type', promClient.register.contentType);
    res.end(await promClient.register.metrics());
});
```

---

## AWS CloudWatch Container Insights

For ECS workloads, AWS CloudWatch Container Insights provides container-level metrics without deploying Prometheus:

```bash
# Enable Container Insights on an ECS cluster
aws ecs update-cluster-settings \
  --cluster production \
  --settings name=containerInsights,value=enabled
```

After enabling, CloudWatch automatically collects:
- CPU and memory utilization per task and container
- Network I/O per task
- Disk I/O per task
- Container restart counts

```bash
# Query Container Insights metrics
aws cloudwatch get-metric-statistics \
  --namespace ECS/ContainerInsights \
  --metric-name CpuUtilized \
  --dimensions Name=ClusterName,Value=production Name=ServiceName,Value=myapp-api \
  --start-time $(date -u -d "1 hour ago" +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 \
  --statistics Average
```

### CloudWatch Alarms for ECS

```bash
# Alert when ECS service CPU > 80% for 15 minutes
aws cloudwatch put-metric-alarm \
  --alarm-name myapp-high-cpu \
  --alarm-description "ECS service CPU > 80%" \
  --namespace ECS/ContainerInsights \
  --metric-name CpuUtilized \
  --dimensions Name=ClusterName,Value=production Name=ServiceName,Value=myapp-api \
  --statistic Average \
  --period 300 \
  --evaluation-periods 3 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:alerts-topic
```

---

## docker system df — Disk Usage

```bash
# Overall Docker disk usage
docker system df

# Detailed breakdown
docker system df -v

# Output:
# TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
# Images          15        8         4.2GB     2.1GB (50%)
# Containers      8         8         1.2GB     0B (0%)
# Local Volumes   12        5         8.5GB     3.2GB (37%)
# Build Cache     0         0         0B        0B

# Clean up everything unused
docker system prune -f              # images, containers, networks
docker system prune -f --volumes    # also removes volumes (careful!)
```

---

## Summary

- `docker stats` provides real-time CPU, memory, and network metrics — essential for immediate triage but does not retain historical data
- cAdvisor exposes all Docker container metrics in Prometheus format — deploy it alongside Prometheus and Grafana for a self-hosted monitoring stack
- Application-level metrics (`/metrics` endpoint with prometheus_client or prom-client) provide insight into request rates, error rates, and latency — infrastructure metrics alone are not enough
- CloudWatch Container Insights on ECS gives container-level metrics without deploying Prometheus — the managed option for teams already invested in the AWS ecosystem
- Set CloudWatch alarms on key ECS metrics (CPU, memory, restart count) and route them to an SNS topic for PagerDuty or Slack notifications
- Monitor Docker disk usage with `docker system df` — dangling images, stopped containers, and unused volumes accumulate and fill disks on long-running hosts

In the next file, we cover container security in production — runtime security controls, Linux capabilities, AppArmor, and seccomp profiles.
