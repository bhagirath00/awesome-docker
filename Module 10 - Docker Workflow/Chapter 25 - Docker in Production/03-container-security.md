# Container Security in Production

## Introduction

The Dockerfile security chapter covered build-time hardening — non-root users, minimal images, no secrets in layers. Runtime security is the complementary layer: restricting what a running container can do, even if the application is compromised.

Container isolation is not a security boundary by default. Linux namespaces and cgroups provide process isolation and resource limits, but a container process with root privileges and full capabilities can still interact with the kernel in ways that may allow container escape. Runtime security controls harden the execution environment.

---

## Linux Capabilities

Linux capabilities divide root privileges into distinct units. Docker grants a default set of capabilities to containers. Most applications need zero of these capabilities — they should be dropped.

### Default Docker Capabilities

Docker grants these capabilities by default:

```
CHOWN, DAC_OVERRIDE, FOWNER, FSETID, KILL, SETGID, SETUID, SETPCAP,
NET_BIND_SERVICE, NET_RAW, SYS_CHROOT, MKNOD, AUDIT_WRITE, SETFCAP
```

Most web APIs and workers need none of these.

### Dropping All Capabilities

```bash
# Drop all capabilities
docker run --cap-drop ALL myapp:1.0

# Add back only what is needed
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE myapp:1.0
```

In Docker Compose:

```yaml
services:
  api:
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE    # only if binding to port < 1024
```

In AWS ECS task definitions:

```json
{
    "containerDefinitions": [
        {
            "name": "api",
            "linuxParameters": {
                "capabilities": {
                    "drop": ["ALL"],
                    "add": []
                }
            }
        }
    ]
}
```

### Common Capability Requirements

| Application | Required Capabilities |
|---|---|
| Web API on port 8080+ | None |
| Nginx on port 80 (root) | NET_BIND_SERVICE |
| Redis | None |
| PostgreSQL | None |
| Ping command | NET_RAW |
| Change file ownership at runtime | CHOWN, FOWNER |
| Bind mount NFS | SYS_ADMIN |

If you cannot identify a specific capability requirement, drop all and observe what breaks.

---

## Read-Only Root Filesystem

Prevent the container from writing to its own filesystem — limits what an attacker can do even with arbitrary code execution inside the container:

```bash
# Enable read-only root filesystem
docker run --read-only myapp:1.0

# The application likely needs /tmp for temporary files
docker run --read-only --tmpfs /tmp:size=256m myapp:1.0

# Multiple writable paths
docker run --read-only \
  --tmpfs /tmp:size=256m \
  --tmpfs /app/logs:size=100m \
  myapp:1.0
```

In Docker Compose:

```yaml
services:
  api:
    image: myapp:1.0
    read_only: true
    tmpfs:
      - /tmp:size=268435456   # 256MB in bytes
      - /app/cache:size=104857600  # 100MB
```

In ECS task definitions:

```json
{
    "containerDefinitions": [
        {
            "name": "api",
            "readonlyRootFilesystem": true,
            "mountPoints": [
                {
                    "containerPath": "/tmp",
                    "sourceVolume": "tmp-volume"
                }
            ]
        }
    ]
}
```

Test read-only compatibility:

```bash
docker run --read-only myapp:1.0
# Permission denied: '/tmp/gunicorn.pid'
# → Add --tmpfs /tmp
```

---

## Seccomp — Syscall Filtering

Seccomp (Secure Computing Mode) restricts which Linux system calls a container can make. Docker applies a default seccomp profile that blocks ~44 dangerous syscalls (e.g., `add_key`, `keyctl`, `ptrace`, `reboot`).

```bash
# Run with the default seccomp profile (applies automatically)
docker run myapp:1.0

# Run without any seccomp filtering (not recommended)
docker run --security-opt seccomp=unconfined myapp:1.0

# Apply a custom seccomp profile
docker run --security-opt seccomp=/path/to/custom-seccomp.json myapp:1.0
```

### Custom Seccomp Profile

A minimal seccomp profile for a Python web server (allows only what Python + networking needs):

```json
// custom-seccomp.json
{
    "defaultAction": "SCMP_ACT_ERRNO",
    "architectures": ["SCMP_ARCH_X86_64"],
    "syscalls": [
        {
            "names": [
                "accept4", "access", "bind", "brk", "chdir", "chmod",
                "clock_gettime", "clone", "close", "connect", "dup", "dup2",
                "epoll_create1", "epoll_ctl", "epoll_wait", "execve", "exit",
                "exit_group", "fcntl", "fstat", "futex", "getcwd", "getdents64",
                "getegid", "geteuid", "getgid", "getpid", "getppid", "getuid",
                "listen", "lseek", "mmap", "mprotect", "munmap", "nanosleep",
                "open", "openat", "pipe", "poll", "prctl", "pread64", "prlimit64",
                "read", "recvfrom", "recvmsg", "rt_sigaction", "rt_sigprocmask",
                "rt_sigreturn", "select", "sendmsg", "sendto", "setitimer",
                "set_robust_list", "setsockopt", "set_tid_address", "shutdown",
                "sigaltstack", "socket", "stat", "statfs", "sysinfo", "tgkill",
                "uname", "wait4", "write", "writev"
            ],
            "action": "SCMP_ACT_ALLOW"
        }
    ]
}
```

For most applications, the default Docker seccomp profile is sufficient. Create custom profiles only if you have a specific threat model that requires tighter syscall restrictions.

---

## AppArmor — Mandatory Access Control

AppArmor enforces per-process security profiles that restrict what resources a process can access. Docker applies the `docker-default` AppArmor profile automatically on supported systems (Ubuntu, Debian).

```bash
# Check if AppArmor is enabled
aa-status

# Run with the default Docker AppArmor profile (default)
docker run myapp:1.0

# Run with a custom AppArmor profile
docker run --security-opt apparmor=myapp-profile myapp:1.0

# Run without AppArmor (not recommended)
docker run --security-opt apparmor=unconfined myapp:1.0
```

### Custom AppArmor Profile

```apparmor
# /etc/apparmor.d/docker-myapp
#include <tunables/global>

profile docker-myapp flags=(attach_disconnected,mediate_deleted) {
    #include <abstractions/base>

    # Allow reading app files
    /app/** r,

    # Allow writing only to /tmp
    /tmp/** rw,

    # Network access
    network tcp,
    network udp,

    # Deny everything else
    deny /etc/** w,
    deny /var/** w,
    deny /root/** rw,
}
```

```bash
# Load the profile
sudo apparmor_parser -r /etc/apparmor.d/docker-myapp

# Apply to a container
docker run --security-opt apparmor=docker-myapp myapp:1.0
```

---

## no-new-privileges

The `no-new-privileges` flag prevents the container process from gaining new privileges through `setuid` binaries:

```bash
docker run --security-opt no-new-privileges:true myapp:1.0
```

In Docker Compose:

```yaml
services:
  api:
    security_opt:
      - no-new-privileges:true
```

In ECS:

```json
{
    "containerDefinitions": [
        {
            "name": "api",
            "privileged": false,
            "linuxParameters": {
                "capabilities": {
                    "drop": ["ALL"]
                }
            }
        }
    ]
}
```

`no-new-privileges` is safe to apply to virtually all production containers. Enable it universally.

---

## The Privileged Flag — Never in Production

```bash
# NEVER DO THIS IN PRODUCTION
docker run --privileged myapp:1.0
```

`--privileged` gives the container full access to the host — effectively root on the Docker host. It disables all seccomp, AppArmor, and capability restrictions. It is used only for:
- Docker-in-Docker (DinD) CI runners (with careful isolation)
- System-level tools that genuinely need host access (cAdvisor on some configurations)

In ECS task definitions, `"privileged": true` is similarly disabled in Fargate and should never be set to true for application containers.

---

## Complete Security Hardening in Docker Compose

```yaml
version: "3.8"

services:
  api:
    image: myapp:1.4.2
    restart: unless-stopped
    user: "1001:1001"         # non-root user
    read_only: true           # immutable root filesystem
    tmpfs:
      - /tmp:size=256m
    cap_drop:
      - ALL                   # drop all Linux capabilities
    security_opt:
      - no-new-privileges:true   # prevent privilege escalation
    mem_limit: 512m
    cpus: 1.5
    ports:
      - "127.0.0.1:8080:8080"   # bind to loopback only (nginx/ALB proxies)
```

---

## AWS ECS Security Controls

```json
{
    "containerDefinitions": [
        {
            "name": "api",
            "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2",
            "essential": true,
            "user": "1001:1001",
            "readonlyRootFilesystem": true,
            "privileged": false,
            "linuxParameters": {
                "capabilities": {
                    "drop": ["ALL"],
                    "add": []
                },
                "tmpfs": [
                    {
                        "containerPath": "/tmp",
                        "size": 256,
                        "mountOptions": ["rw", "noexec", "nosuid", "nodev"]
                    }
                ]
            }
        }
    ]
}
```

---

## Security Checklist for Production Containers

```
[ ] Non-root user (USER 1001)
[ ] Read-only root filesystem (--read-only)
[ ] Temporary filesystems for writable paths (/tmp via tmpfs)
[ ] All capabilities dropped (--cap-drop ALL)
[ ] No additional capabilities added unless specifically needed
[ ] no-new-privileges flag set
[ ] Container is not privileged (--privileged NOT set)
[ ] Resource limits set (mem_limit, cpus)
[ ] Vulnerability scan passes (no Critical/High CVEs)
[ ] Image is based on minimal base image (slim/alpine/distroless)
[ ] Ports bound to 127.0.0.1 or proxied (not 0.0.0.0 directly)
```

---

## Summary

- Drop all Linux capabilities (`--cap-drop ALL`) for application containers — most web services need zero capabilities; add back only what is specifically required
- Enable read-only root filesystem (`--read-only`) — forces writable paths to be explicitly declared as tmpfs mounts, limits what an attacker can persist
- Apply `no-new-privileges:true` to all production containers — prevents setuid binaries from escalating privileges
- The default Docker seccomp profile blocks dangerous syscalls automatically — only override with a custom profile for specific advanced use cases
- Never use `--privileged` for application containers in production — it disables all security isolation and grants full host access
- In ECS, enforce security at the task definition level: `readonlyRootFilesystem: true`, `privileged: false`, `user: "1001:1001"`, `linuxParameters.capabilities.drop: ["ALL"]`

In the final file of this module, we cover Docker Swarm — Docker's native clustering solution for multi-host container orchestration.
