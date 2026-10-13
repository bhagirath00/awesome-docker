# Advanced Dockerfile Instructions

## Introduction

Beyond the core instructions (`FROM`, `RUN`, `COPY`, `ENV`, `CMD`, `ENTRYPOINT`, `EXPOSE`), Docker provides a set of advanced instructions for specialized use cases. These instructions — `ONBUILD`, `STOPSIGNAL`, `SHELL`, `LABEL`, and `WORKDIR` with advanced patterns — give you fine-grained control over how images behave, how they communicate metadata, and how they integrate into organizational build pipelines.

This file covers each advanced instruction, what it does, and the specific scenarios where it adds real value.

---

## LABEL — Structured Image Metadata

`LABEL` attaches key-value metadata to an image. Labels are searchable, inspectable, and serve as the primary way to document image provenance, ownership, and lifecycle information.

```dockerfile
FROM python:3.11-slim

# OCI-standard image labels
LABEL org.opencontainers.image.title="My API Service" \
      org.opencontainers.image.description="REST API for MyApp platform" \
      org.opencontainers.image.version="${APP_VERSION}" \
      org.opencontainers.image.revision="${GIT_COMMIT}" \
      org.opencontainers.image.created="${BUILD_DATE}" \
      org.opencontainers.image.source="https://github.com/myorg/myapp" \
      org.opencontainers.image.url="https://myapp.example.com" \
      org.opencontainers.image.documentation="https://docs.myapp.example.com" \
      org.opencontainers.image.vendor="My Organization" \
      org.opencontainers.image.licenses="MIT"

# Team and project labels
LABEL com.mycompany.team="platform-engineering" \
      com.mycompany.project="myapp" \
      com.mycompany.maintainer="platform-team@mycompany.com" \
      com.mycompany.environment="production"
```

### OCI Image Labels (Standard)

The Open Container Initiative (OCI) defined a standard set of well-known label keys under the `org.opencontainers.image.*` namespace. Using these lets tooling (Renovate, Dependabot, security scanners) automatically understand your images.

### Querying Labels

```bash
# Inspect all labels on an image
docker inspect myapp:1.0 --format '{{json .Config.Labels}}' | python3 -m json.tool

# Filter images by label
docker images --filter label=com.mycompany.team=platform-engineering

# Filter running containers by label
docker ps --filter label=com.mycompany.project=myapp
```

### Best Practices for Labels

- Use namespaced keys (`com.mycompany.*`) to avoid collisions with Docker built-ins
- Use ARGs for dynamic values (version, git commit, build date) so they are set at build time
- Consolidate all `LABEL` instructions into a single `LABEL` command to minimize layers

---

## STOPSIGNAL — Graceful Shutdown Signal

When you run `docker stop`, Docker sends a signal to the container's PID 1 to request graceful shutdown. By default, this signal is `SIGTERM`. After the stop timeout (`--time`, default 10 seconds), Docker sends `SIGKILL` to force-terminate the process.

`STOPSIGNAL` overrides which signal Docker sends on stop:

```dockerfile
# Tell Docker to use SIGINT instead of SIGTERM (some apps prefer this)
STOPSIGNAL SIGINT

# Use signal number (equivalent to SIGTERM)
STOPSIGNAL 15
```

### Why This Matters

Different applications expect different shutdown signals:
- **Gunicorn**: responds to `SIGTERM` (graceful shutdown), `SIGQUIT` (graceful with core dump), or `SIGINT`
- **Nginx**: responds to `SIGQUIT` for graceful shutdown (drains existing connections)
- **Some JVM apps**: respond to `SIGTERM` but may not handle `SIGINT` cleanly
- **Redis**: responds to `SIGTERM` for graceful shutdown

```dockerfile
# Nginx — use SIGQUIT for graceful connection draining
FROM nginx:1.25-alpine
STOPSIGNAL SIGQUIT
```

```dockerfile
# Python/Gunicorn — SIGTERM is the standard
FROM python:3.11-slim
# STOPSIGNAL defaults to SIGTERM — usually no override needed
```

### Checking What Signal Your Application Needs

```bash
# Look at the container's configured stop signal
docker inspect myapp:1.0 --format '{{.Config.StopSignal}}'

# Manually test graceful shutdown
docker kill --signal SIGTERM myapp_api_1   # sends SIGTERM
docker kill --signal SIGQUIT myapp_nginx_1  # sends SIGQUIT (nginx graceful)
```

---

## SHELL — Override the Default Shell

By default, Docker uses `/bin/sh -c` for `RUN` commands in shell form. `SHELL` changes which shell is used for all subsequent shell-form `RUN`, `CMD`, and `ENTRYPOINT` instructions.

```dockerfile
FROM ubuntu:22.04

# Default: /bin/sh -c
RUN echo $0   # prints /bin/sh

# Switch to bash
SHELL ["/bin/bash", "-c"]

# Now uses bash for all subsequent shell-form RUN commands
RUN source /etc/environment && echo "Using bash now"

# Enable strict mode for bash scripts
SHELL ["/bin/bash", "-euo", "pipefail", "-c"]
RUN complex_command1 | complex_command2   # fails fast on any error
```

### Most Important Use Case: Strict Mode

When you have complex shell commands in `RUN` instructions, bash strict mode catches errors that `/bin/sh` would silently swallow:

```dockerfile
FROM ubuntu:22.04

# Enable strict mode bash
SHELL ["/bin/bash", "-euo", "pipefail", "-c"]

# This now fails the build if any command in the pipe fails
RUN wget -q https://example.com/install.sh | bash
```

Without `pipefail`, if `wget` fails, `/bin/sh -c` will still run the pipe and the `RUN` may succeed even though the download failed.

### PowerShell on Windows

For Windows-based containers:

```dockerfile
FROM mcr.microsoft.com/windows/servercore:ltsc2022

# Use PowerShell instead of cmd.exe
SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop';"]

RUN Install-WindowsFeature -Name Web-Server
```

---

## ONBUILD — Instructions for Child Images

`ONBUILD` registers instructions that execute when the current image is used as a base image in another Dockerfile's `FROM`. The instructions do not run when you build the image itself — they run when someone builds a child image.

```dockerfile
# Parent image: myapp-base:latest
FROM python:3.11-slim

WORKDIR /app

# These run when building the CHILD image:
ONBUILD COPY requirements.txt .
ONBUILD RUN pip install --no-cache-dir -r requirements.txt
ONBUILD COPY . .

CMD ["python", "app.py"]
```

Now, a developer can create a minimal Dockerfile that just says:

```dockerfile
# Child image
FROM myapp-base:latest
# The ONBUILD instructions from myapp-base automatically run here:
# 1. COPY requirements.txt .
# 2. RUN pip install ...
# 3. COPY . .
# No additional instructions needed!
```

### When to Use ONBUILD

`ONBUILD` is useful for creating organization-wide base images that enforce a standard build pattern. Every team that extends the base image automatically gets:
- The correct working directory
- The correct user setup
- The correct dependency installation pattern

```dockerfile
# Organization base image: my-org/python-base:3.11
FROM python:3.11-slim

RUN groupadd --system appuser && \
    useradd --system --no-create-home --gid appuser appuser

WORKDIR /app

ONBUILD COPY requirements.txt .
ONBUILD RUN pip install --no-cache-dir -r requirements.txt && \
            chown -R appuser:appuser /app
ONBUILD COPY --chown=appuser:appuser . .
ONBUILD USER appuser
```

Any team's Dockerfile becomes:

```dockerfile
FROM my-org/python-base:3.11
EXPOSE 8080
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "app:app"]
```

**Caution:** `ONBUILD` is powerful but can create surprising behavior. Document it clearly so users know what will happen when they use your base image. It is marked in `docker inspect` output with `OnBuild:` so it is discoverable.

---

## WORKDIR — Advanced Usage

`WORKDIR` sets the working directory for subsequent `RUN`, `CMD`, `ENTRYPOINT`, `COPY`, and `ADD` instructions. Several advanced patterns:

### Multiple WORKDIR Declarations

Each `WORKDIR` builds on the previous one (relative paths) or resets it (absolute paths):

```dockerfile
WORKDIR /app
WORKDIR src          # now at /app/src
WORKDIR ../tests     # now at /app/tests (relative to previous)
WORKDIR /new-dir     # now at /new-dir (absolute — resets)
```

### Creating Directories via WORKDIR

`WORKDIR` creates the directory if it does not exist. No need for a separate `RUN mkdir`:

```dockerfile
# This creates /app/logs, /app/data, /app/config
WORKDIR /app/logs
WORKDIR /app/data
WORKDIR /app/config
WORKDIR /app        # reset back to /app
```

### WORKDIR with ARG

```dockerfile
ARG APP_ROOT=/app

WORKDIR ${APP_ROOT}
```

---

## EXPOSE — Documentation and Port Publication

`EXPOSE` documents which ports the container's application listens on. It does not actually publish the port — that is done by `-p` at runtime.

```dockerfile
# TCP (default)
EXPOSE 8080

# UDP
EXPOSE 53/udp

# Multiple ports
EXPOSE 8080 9090 8443
```

`EXPOSE` enables:
- `docker run -P` (capital P) to automatically map all exposed ports to random host ports
- Documentation for operators — tells them which ports to map
- Network scanning tools and image inspection to discover what the image uses

Despite not opening actual ports, `EXPOSE` is considered a best practice for clear documentation.

---

## ADD vs COPY — When ADD is Appropriate

`ADD` does everything `COPY` does, plus:
- Automatically extracts tar archives (`.tar`, `.tar.gz`, `.tar.bz2`, `.tar.xz`, `.zip`)
- Fetches files from URLs (not recommended — use `RUN wget` instead for better caching)

```dockerfile
# COPY — always prefer this for regular files
COPY app.py /app/
COPY ./config/ /app/config/

# ADD — only when you need tar extraction
ADD ./data.tar.gz /data/   # extracts the tar into /data/
```

Use `COPY` by default. Use `ADD` only when you specifically need the tar extraction feature.

---

## Summary

- `LABEL` attaches structured metadata to images — use OCI standard keys (`org.opencontainers.image.*`) for compatibility with ecosystem tooling and namespace your custom keys with a reverse-DNS prefix
- `STOPSIGNAL` overrides the signal sent by `docker stop` — use `SIGQUIT` for nginx graceful shutdown, leave as default `SIGTERM` for most applications
- `SHELL` overrides the default shell for subsequent RUN commands — use `SHELL ["/bin/bash", "-euo", "pipefail", "-c"]` to enable strict mode in complex build scripts
- `ONBUILD` registers instructions that run when the image is used as a base — powerful for organizational base images that enforce build standards across all teams
- `WORKDIR` creates directories automatically and accumulates relative paths — always prefer it over `RUN mkdir && cd`
- Use `COPY` for regular file copying; use `ADD` only when you need automatic tar extraction

In the next chapter, we move to production Dockerfile patterns — non-root users, build-time secrets, linting, and the complete security hardening checklist.
