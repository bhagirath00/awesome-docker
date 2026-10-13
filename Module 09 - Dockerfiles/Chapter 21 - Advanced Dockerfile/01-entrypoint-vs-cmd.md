# ENTRYPOINT vs CMD

## Introduction

`ENTRYPOINT` and `CMD` are the two Dockerfile instructions that define what runs when a container starts. They are both commonly used, frequently confused, and their interaction is subtle. Getting this wrong means containers that cannot be overridden cleanly, that break when run with arguments, or that behave differently than the documentation says they should.

This file explains exactly what each instruction does, how they interact, and what the correct pattern is for each type of application.

---

## What They Do

### CMD

`CMD` sets the **default command** for the container. It is what runs when no command is specified in `docker run`. If a command is provided at runtime, `CMD` is completely replaced.

```dockerfile
FROM alpine
CMD ["echo", "Hello from CMD"]
```

```bash
# Uses the CMD default
docker run myimage
# Output: Hello from CMD

# CMD is replaced by the runtime command
docker run myimage echo "Custom command"
# Output: Custom command

# CMD is replaced entirely
docker run myimage ls /etc
# Output: (lists /etc contents)
```

### ENTRYPOINT

`ENTRYPOINT` sets the **executable** that always runs. It is not replaced by runtime arguments — the runtime arguments are **appended** to the entrypoint as additional arguments.

```dockerfile
FROM alpine
ENTRYPOINT ["echo"]
```

```bash
# Runs: echo (with no args)
docker run myimage
# Output: (empty line)

# Runs: echo "Hello"
docker run myimage "Hello"
# Output: Hello

# Runs: echo "Custom message"
docker run myimage "Custom message"
# Output: Custom message
```

The entrypoint is always `echo`. The text after the image name becomes the argument.

---

## How CMD and ENTRYPOINT Interact

When both are defined, `CMD` provides the **default arguments** for the `ENTRYPOINT`.

```dockerfile
FROM alpine
ENTRYPOINT ["echo"]
CMD ["Hello, World"]
```

```bash
# No runtime command → runs: echo "Hello, World"
docker run myimage
# Output: Hello, World

# Runtime args replace CMD (not ENTRYPOINT) → runs: echo "Custom message"
docker run myimage "Custom message"
# Output: Custom message
```

This is the most powerful pattern: `ENTRYPOINT` is the program, `CMD` is the default argument. Users can customize the behavior by providing different arguments without knowing or caring about the underlying executable.

---

## Shell Form vs Exec Form

Both instructions support two forms, and the choice matters significantly.

### Exec Form (Recommended)

```dockerfile
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "8080"]
```

The exec form runs the command **directly** — no shell wrapping. The process becomes PID 1 inside the container. This means it receives signals (like `SIGTERM` from `docker stop`) directly.

### Shell Form (Avoid for Entrypoints)

```dockerfile
ENTRYPOINT python app.py
CMD --port 8080
```

The shell form wraps the command in `/bin/sh -c`. This creates a shell as PID 1, and the shell runs the actual command. When `docker stop` sends `SIGTERM`, it goes to the shell — not to your application. The shell may or may not forward it. This leads to containers that do not shut down gracefully and must be killed.

**Rule:** Always use exec form (`["..."]`) for `ENTRYPOINT`. Use exec form for `CMD` when paired with an `ENTRYPOINT`.

---

## The Complete Interaction Matrix

| Dockerfile | docker run | Result |
|---|---|---|
| `ENTRYPOINT ["ep"]`, `CMD ["cmd"]` | (nothing) | `ep cmd` |
| `ENTRYPOINT ["ep"]`, `CMD ["cmd"]` | `arg1 arg2` | `ep arg1 arg2` |
| `ENTRYPOINT ["ep"]`, `CMD ["cmd"]` | `--entrypoint ep2 arg` | `ep2 arg` |
| Only `CMD ["cmd"]` | (nothing) | `cmd` |
| Only `CMD ["cmd"]` | `arg1 arg2` | `arg1 arg2` |
| Only `ENTRYPOINT ["ep"]` | (nothing) | `ep` |
| Only `ENTRYPOINT ["ep"]` | `arg1 arg2` | `ep arg1 arg2` |

---

## The Correct Pattern for Application Containers

For a service that runs as a daemon (web server, API, worker), the standard pattern is:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY src/ .

# ENTRYPOINT = the executable (never changes)
# CMD = default arguments (can be overridden)
ENTRYPOINT ["gunicorn"]
CMD ["--workers", "4", "--bind", "0.0.0.0:8080", "app:app"]
```

```bash
# Normal production run
docker run myapp:1.0
# Runs: gunicorn --workers 4 --bind 0.0.0.0:8080 app:app

# Override workers for a low-memory environment
docker run myapp:1.0 --workers 2 --bind 0.0.0.0:8080 app:app

# Override for development with reload
docker run myapp:1.0 --workers 1 --reload --bind 0.0.0.0:8080 app:app
```

Users can adjust Gunicorn arguments without knowing how the image was built.

---

## The Wrapper Script Pattern (Production Best Practice)

Many production containers use a shell wrapper script as the `ENTRYPOINT`. This script performs initialization — setting up environment, running migrations, waiting for dependencies — before handing off to the main application.

```bash
# docker-entrypoint.sh
#!/bin/sh
set -e

# Wait for database to be available
until pg_isready -h "$DB_HOST" -U "$DB_USER" -d "$DB_NAME"; do
  echo "Waiting for database..."
  sleep 2
done

# Run migrations
python manage.py migrate

# Execute the CMD (whatever was passed to docker run)
exec "$@"
```

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
COPY docker-entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-entrypoint.sh

ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["gunicorn", "--workers", "4", "--bind", "0.0.0.0:8080", "app:app"]
```

The crucial last line of the script is `exec "$@"`. This uses `exec` to **replace** the shell process with the CMD — making the CMD the new PID 1 and ensuring it receives signals correctly. Without `exec`, the shell stays as PID 1 and swallows signals.

This pattern is used by all official Docker Hub images (postgres, redis, nginx, etc.) — it is the production standard.

---

## When to Use CMD Alone

Use `CMD` alone (no `ENTRYPOINT`) when:
- You want the container to be usable as a general-purpose shell environment
- You are building a base image for others to extend
- The container needs to be easily overridden with any command

```dockerfile
FROM ubuntu:22.04

# Basic Ubuntu image — CMD alone so users can run anything
CMD ["/bin/bash"]
```

```bash
docker run -it ubuntu bash       # runs bash
docker run ubuntu ls /etc        # runs ls
docker run ubuntu python3 app.py # runs python3
```

---

## When to Use ENTRYPOINT Alone

Use `ENTRYPOINT` alone when the container is a specific tool and should always run that tool:

```dockerfile
FROM alpine

RUN apk add --no-cache curl

ENTRYPOINT ["curl"]
```

```bash
# Container behaves like the curl command itself
docker run curlimage https://api.example.com
docker run curlimage -v -X POST https://api.example.com/data -d '{"key":"value"}'
```

---

## Overriding ENTRYPOINT at Runtime

If you need to bypass the entrypoint entirely (useful for debugging):

```bash
# Override ENTRYPOINT to get a shell
docker run --entrypoint /bin/sh -it myapp:1.0

# Override ENTRYPOINT for a quick command
docker run --entrypoint python myapp:1.0 -c "import app; app.debug()"
```

---

## AWS ECS Context

In ECS task definitions, `command` corresponds to `CMD` and `entryPoint` corresponds to `ENTRYPOINT`:

```json
{
  "containerDefinitions": [
    {
      "name": "api",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2",
      "entryPoint": ["docker-entrypoint.sh"],
      "command": ["gunicorn", "--workers", "4", "--bind", "0.0.0.0:8080", "app:app"]
    }
  ]
}
```

If you override `command` in the task definition, the `entryPoint` from the image still runs first — same behavior as `docker run`.

---

## Summary

- `CMD` sets the default command — completely replaced when arguments are provided at runtime
- `ENTRYPOINT` sets the executable — runtime arguments are appended, not replaced; use `--entrypoint` to override it
- When both are defined, `ENTRYPOINT` is the executable and `CMD` provides its default arguments — runtime args replace the `CMD` portion
- Always use exec form (`["..."]`) for `ENTRYPOINT` to ensure your application is PID 1 and receives signals directly
- The wrapper script pattern (`ENTRYPOINT ["docker-entrypoint.sh"]` + `CMD ["app"]`) is the production standard — it enables initialization steps while preserving signal handling via `exec "$@"`
- Use `CMD` alone for general-purpose base images; use `ENTRYPOINT` alone for container-as-tool patterns

In the next file, we cover the `ARG` vs `ENV` distinction — two instructions that look similar but have fundamentally different scopes and security implications.
