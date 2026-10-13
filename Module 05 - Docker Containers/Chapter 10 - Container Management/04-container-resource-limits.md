# Container Resource Limits

## Introduction

Without resource limits, a container can consume all the CPU and memory on the host machine. When that happens, every other container and the host itself suffers — slow responses, OOM kills, and potentially a complete system outage.

Resource limits protect your host and other containers from a single misbehaving container. They also give you the data you need to right-size your infrastructure on AWS — knowing how much CPU and memory your application actually uses lets you choose the right EC2 instance type and set the right ECS task resource allocations.

---

## Memory Limits

### Setting Memory Limits

```bash
# Set a hard memory limit of 512MB
docker run -d \
  --name api \
  --memory 512m \
  myapp:1.0

# Also set swap limit equal to memory (disables swap)
docker run -d \
  --name api \
  --memory 512m \
  --memory-swap 512m \
  myapp:1.0
```

Memory units: `b` (bytes), `k` (kilobytes), `m` (megabytes), `g` (gigabytes)

```bash
--memory 512m     # 512 megabytes
--memory 1g       # 1 gigabyte
--memory 2048m    # 2 gigabytes
```

### What Happens When a Container Exceeds Memory Limit

When a container exceeds its memory limit, the Linux kernel's OOM (Out of Memory) killer terminates the container process. Docker records this as an OOM kill.

```bash
# Check if container was OOM killed
docker inspect --format '{{.State.OOMKilled}}' api
# true

# Check exit code — OOM kill exits with 137
docker inspect --format '{{.State.ExitCode}}' api
# 137
```

Exit code 137 is always `SIGKILL` (128 + 9). When combined with `OOMKilled: true`, it tells you the container was killed because it ran out of memory.

### Memory vs Swap

The relationship between `--memory` and `--memory-swap`:

```bash
# memory=512m, memory-swap=1g → container can use 512MB RAM + 512MB swap
docker run --memory 512m --memory-swap 1g myapp:1.0

# memory=512m, memory-swap=512m → no swap (recommended for predictable behavior)
docker run --memory 512m --memory-swap 512m myapp:1.0

# memory=512m, memory-swap=-1 → unlimited swap
docker run --memory 512m --memory-swap -1 myapp:1.0
```

For production containers, disable swap by setting `--memory-swap` equal to `--memory`. Swap makes memory exhaustion slow and unpredictable — you want the OOM kill to happen fast so the container restarts quickly.

### Soft Memory Limit (Reservation)

`--memory-reservation` sets a soft limit. Docker does not enforce it strictly but uses it for scheduling decisions:

```bash
# Hard limit: 1GB. Soft reservation: 512MB
docker run -d \
  --memory 1g \
  --memory-swap 1g \
  --memory-reservation 512m \
  --name api \
  myapp:1.0
```

On AWS ECS, the `memory` field in the task definition corresponds to the hard limit, and `memoryReservation` corresponds to the soft limit.

---

## CPU Limits

### Limiting CPU with --cpus

`--cpus` sets how many CPU cores the container can use:

```bash
# Container can use at most 1.0 full CPU core
docker run -d --cpus 1.0 myapp:1.0

# Container can use at most 0.5 of a CPU core
docker run -d --cpus 0.5 myapp:1.0

# Container can use at most 2.5 CPU cores
docker run -d --cpus 2.5 myapp:1.0
```

This does not pin the container to specific CPU cores — it throttles total CPU usage. A container with `--cpus 1.0` can run on any core, but its total CPU time is capped at the equivalent of one full core.

### CPU Shares (Relative Weight)

`--cpu-shares` sets a relative CPU weight rather than an absolute limit. The default is 1024.

```bash
# This container gets twice the CPU time as containers with default 1024 shares
docker run -d --cpu-shares 2048 high-priority-app:1.0

# This container gets half the CPU time
docker run -d --cpu-shares 512 low-priority-worker:1.0
```

CPU shares only matter when the host is under CPU pressure. If the host has idle CPU, all containers get as much as they need regardless of shares.

### Pinning to Specific CPU Cores

```bash
# Pin container to CPUs 0 and 1 only
docker run -d --cpuset-cpus "0,1" myapp:1.0

# Pin to a range of CPUs
docker run -d --cpuset-cpus "0-3" myapp:1.0
```

CPU pinning is an advanced optimization for latency-sensitive workloads. Most applications do not need it.

---

## Monitoring Resource Usage

### docker stats — Live Resource Dashboard

```bash
# Monitor all running containers
docker stats

# Output:
CONTAINER   CPU %   MEM USAGE / LIMIT   MEM %   NET I/O           BLOCK I/O
api         2.3%    245MiB / 512MiB     47.9%   12.4MB / 4.5MB    0B / 0B
webserver   0.0%    8.5MiB / 256MiB     3.3%    1.2kB / 648B      0B / 0B
postgres    0.4%    89MiB / 512MiB      17.4%   1.1MB / 980kB     5.2MB / 8.4MB

# Monitor specific containers
docker stats api webserver postgres

# One-time snapshot (no live update)
docker stats --no-stream

# Custom format
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}"
```

### Interpreting docker stats Output

| Column | What it means |
|---|---|
| `CPU %` | Percentage of total host CPU used |
| `MEM USAGE / LIMIT` | Current memory used / hard limit |
| `MEM %` | Memory usage as a percentage of limit |
| `NET I/O` | Network bytes received / sent |
| `BLOCK I/O` | Disk bytes read / written |

Watch for:
- `MEM %` above 80% consistently — consider increasing the memory limit or fixing a memory leak
- `CPU %` consistently at 100% of limit — the container is CPU-throttled; increase the limit or optimize the code
- `MEM USAGE` growing continuously — likely a memory leak

---

## Setting Appropriate Limits

Choosing the right limits requires measurement. Setting limits too low causes OOM kills. Setting them too high wastes money on AWS.

### Step 1: Run Without Limits and Measure

```bash
# Run with no limits and measure under realistic load
docker run -d --name api-measure myapp:1.0

# Apply load (run your test suite or use a load testing tool like k6 or wrk)
wrk -t4 -c100 -d60s http://localhost:8080/api/endpoint

# Monitor while load runs
docker stats api-measure --no-stream

# Output under load:
CPU %   MEM USAGE
45.2%   380MiB
```

### Step 2: Set Limits with Headroom

Add 25-50% headroom above the measured peak:

```bash
# Measured peak: 380MB RAM, 45% CPU on 4-core host (= 1.8 CPU cores)
# Set limits: 480MB RAM (380 * 1.25), 2.25 CPU (1.8 * 1.25)

docker run -d \
  --name api \
  --memory 480m \
  --memory-swap 480m \
  --cpus 2.25 \
  myapp:1.0
```

### Step 3: Monitor in Production

```bash
# Set up continuous monitoring
# AWS CloudWatch Container Insights monitors ECS containers automatically
# For standalone Docker, use Prometheus + cAdvisor
docker run -d \
  --name cadvisor \
  --volume /var/run/docker.sock:/var/run/docker.sock:ro \
  --volume /sys:/sys:ro \
  --volume /var/lib/docker/:/var/lib/docker:ro \
  -p 8081:8080 \
  gcr.io/cadvisor/cadvisor:latest
```

---

## Resource Limits in AWS ECS

On AWS ECS, resource limits are defined in the task definition. ECS uses them to:
- Schedule tasks on instances with sufficient resources
- Enforce limits at runtime
- Report CloudWatch Container Insights metrics

```json
{
  "family": "myapp-task",
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "api",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2",
      "cpu": 256,
      "memory": 512,
      "memoryReservation": 256,
      "essential": true
    }
  ]
}
```

ECS CPU units: 1024 units = 1 vCPU

```
256 units  = 0.25 vCPU
512 units  = 0.5 vCPU
1024 units = 1 vCPU
2048 units = 2 vCPU
```

On AWS Fargate, you specify CPU and memory at the task level. Available combinations:

| CPU | Memory Options |
|---|---|
| 256 (.25 vCPU) | 512MB, 1GB, 2GB |
| 512 (.5 vCPU) | 1GB to 4GB |
| 1024 (1 vCPU) | 2GB to 8GB |
| 2048 (2 vCPU) | 4GB to 16GB |
| 4096 (4 vCPU) | 8GB to 30GB |

---

## Resource Limits Complete Example

A production-ready docker run command with all resource controls set:

```bash
docker run -d \
  --name api \
  --restart unless-stopped \
  \
  # Memory — hard limit 512MB, no swap
  --memory 512m \
  --memory-swap 512m \
  --memory-reservation 256m \
  \
  # CPU — max 1.5 cores
  --cpus 1.5 \
  \
  # Application configuration
  -p 8080:8080 \
  -e APP_ENV=production \
  -e DATABASE_URL=postgresql://user:pass@db:5432/myapp \
  \
  # Logging
  --log-driver json-file \
  --log-opt max-size=50m \
  --log-opt max-file=3 \
  \
  # Labels for identification
  --label app=myapp \
  --label version=1.4.2 \
  --label environment=production \
  \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2
```

---

## Summary

- Without resource limits, a container can consume all host resources and cause system-wide outages
- `--memory` sets a hard memory limit — the container is OOM killed if it exceeds this
- Set `--memory-swap` equal to `--memory` to disable swap for predictable, fast OOM behavior
- Exit code 137 with `OOMKilled: true` means the container was killed for exceeding its memory limit — check `docker inspect`
- `--cpus` sets a CPU usage cap as a number of CPU cores (supports decimals like `1.5` or `0.5`)
- Use `docker stats --no-stream` to measure actual resource usage under load before setting limits
- Set limits 25-50% above measured peak to provide headroom for traffic spikes
- On AWS ECS, define CPU and memory in the task definition — ECS uses these for scheduling and enforcement

In the next file, we cover `docker exec` in depth — how to inspect, debug, and interact with running containers.
