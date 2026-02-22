# Environment Variables in Compose

## Introduction

Docker Compose provides a layered, flexible system for managing environment variables. Understanding how variables flow from the `.env` file into containers, how variable substitution works in the Compose file itself, and where secrets should and should not live is essential for building secure, configurable applications.

This file covers every mechanism Compose provides for handling environment variables and the production patterns that protect secrets.

---

## The .env File — Compose Variable Substitution

When Docker Compose starts, it automatically reads a file named `.env` in the same directory as the Compose file. Variables in `.env` are available for **substitution inside the Compose YAML file itself** — they are not automatically passed to containers unless explicitly referenced.

```bash
# .env file
APP_VERSION=1.4.2
DB_PASSWORD=supersecretpassword
AWS_REGION=us-east-1
ECR_REGISTRY=123456789012.dkr.ecr.us-east-1.amazonaws.com
```

```yaml
# docker-compose.yml — uses .env variables for substitution
version: "3.8"

services:
  api:
    image: ${ECR_REGISTRY}/myapp:${APP_VERSION}   # substituted from .env
    environment:
      APP_VERSION: ${APP_VERSION}     # also passed to the container
      DATABASE_URL: postgresql://user:${DB_PASSWORD}@db:5432/myapp
```

This is the distinction: `.env` populates Compose's own variable context (for the YAML file). The service's `environment:` block controls what the container actually receives.

### Default Values

You can provide defaults for variables that might not be set:

```yaml
# Syntax: ${VARIABLE:-default}
services:
  api:
    environment:
      APP_ENV: ${APP_ENV:-development}   # uses "development" if APP_ENV is unset
      PORT: ${PORT:-8080}
      LOG_LEVEL: ${LOG_LEVEL:-info}
```

### Required Variables (No Default)

To make a variable required (Compose fails if it is not set):

```yaml
# Syntax: ${VARIABLE:?error message}
services:
  api:
    image: ${ECR_REGISTRY:?ECR_REGISTRY must be set in .env}/myapp:${APP_VERSION:?APP_VERSION must be set}
```

If `ECR_REGISTRY` is not defined in `.env` or the environment, Compose prints the error message and exits.

---

## Passing Variables to Containers

There are four distinct ways to set environment variables on a container in Compose:

### 1. Inline in environment: block

```yaml
services:
  api:
    environment:
      APP_ENV: production         # static value
      PORT: "8080"
      LOG_LEVEL: info
```

### 2. Substituted from .env in environment: block

```yaml
services:
  api:
    environment:
      DATABASE_URL: postgresql://user:${DB_PASSWORD}@db:5432/myapp
      ECR_REGISTRY: ${ECR_REGISTRY}
```

The container receives the resolved value — `DB_PASSWORD` is substituted and the container gets the actual password string.

### 3. Passed through from host shell

```yaml
services:
  api:
    environment:
      AWS_ACCESS_KEY_ID:       # no value = pass host's value through
      AWS_SECRET_ACCESS_KEY:
      AWS_SESSION_TOKEN:
```

If `AWS_ACCESS_KEY_ID` is set in your shell, it is passed to the container unchanged. If it is not set, the variable is not set in the container either.

### 4. From env_file

```yaml
services:
  api:
    env_file:
      - .env.base
      - .env.production
```

Variables in the env files are passed **directly to the container** — this is different from the `.env` file which only populates Compose's substitution context. Both the `.env` (Compose substitution) and `env_file` (direct container injection) are often used together:

```
.env              → Compose substitution (image tags, registry URLs)
.env.production   → env_file (application config, passed to containers)
```

---

## Variable Precedence

When the same variable is defined in multiple places, this is the resolution order (highest to lowest):

1. Values set directly in `environment:` block inline
2. Shell environment variables of the person running `docker compose`
3. `.env` file in the project directory
4. Defaults in the Compose file (`${VAR:-default}`)

```yaml
services:
  api:
    environment:
      LOG_LEVEL: info    # This wins over everything else for LOG_LEVEL
```

---

## Environment Variables for the Compose CLI

Some environment variables control how the Compose CLI itself behaves — they are not passed to containers:

```bash
# Set the project name
COMPOSE_PROJECT_NAME=myapp docker compose up -d

# Use a specific file
COMPOSE_FILE=docker-compose.prod.yml docker compose up -d

# Use multiple files (colon-separated on Linux, semicolon on Windows)
COMPOSE_FILE=docker-compose.yml:docker-compose.override.yml docker compose up -d
```

---

## Secrets in Compose — What to Do and What Not to Do

### The Problem: .env Files and Version Control

```bash
# BAD — committed to Git
.env
DB_PASSWORD=supersecretpassword
JWT_SECRET=myjwtsecret
```

If `.env` is committed to your repository, every developer and CI system that clones the repo has your production secrets. If the repo is public, your secrets are public.

**Rule:** Always add `.env` to `.gitignore`. Commit a `.env.example` with placeholder values instead.

```bash
# .gitignore
.env
.env.production
.env.local

# .env.example (committed to Git — no real values)
APP_VERSION=1.0.0
DB_PASSWORD=change-me
JWT_SECRET=change-me
AWS_REGION=us-east-1
```

### Approach 1: .env File with Manual Distribution

The simplest approach for small teams. Each developer and deployment server has a `.env` file that is not committed to Git. You distribute secrets manually (via 1Password, LastPass, or a secure Slack channel):

```bash
# Developer copies from .env.example and fills in real values
cp .env.example .env
vim .env
```

This does not scale — when you rotate a secret, you must update every developer's `.env` manually.

### Approach 2: AWS Secrets Manager + Env Injection Script

Fetch secrets from AWS Secrets Manager at deployment time and inject them into the environment before running Compose:

```bash
#!/bin/bash
# deploy.sh

# Fetch secrets from AWS Secrets Manager
DB_PASSWORD=$(aws secretsmanager get-secret-value \
  --secret-id myapp/production/database \
  --query 'SecretString' \
  --output text | python3 -c "import sys,json; print(json.load(sys.stdin)['password'])")

JWT_SECRET=$(aws secretsmanager get-secret-value \
  --secret-id myapp/production/jwt \
  --query 'SecretString' \
  --output text)

# Export for Compose substitution
export DB_PASSWORD JWT_SECRET

# Run compose — values come from shell environment, not .env file
docker compose -f docker-compose.prod.yml up -d
```

The secrets are never written to disk. They live only in the shell's environment during the deployment.

### Approach 3: AWS SSM Parameter Store

Similar to Secrets Manager but uses `aws ssm get-parameter`:

```bash
DB_PASSWORD=$(aws ssm get-parameter \
  --name /myapp/production/db_password \
  --with-decryption \
  --query 'Parameter.Value' \
  --output text)
```

---

## Practical Complete Example

```yaml
version: "3.8"

services:
  api:
    image: ${ECR_REGISTRY}/myapp:${APP_VERSION:-latest}
    restart: unless-stopped
    ports:
      - "${API_PORT:-8080}:8080"
    environment:
      # Static config
      APP_ENV: production
      LOG_LEVEL: ${LOG_LEVEL:-info}
      # Secrets injected from shell env (never in .env file)
      DATABASE_URL: postgresql://apiuser:${DB_PASSWORD:?DB_PASSWORD must be set}@db:5432/myapp
      JWT_SECRET: ${JWT_SECRET:?JWT_SECRET must be set}
      # AWS config from host credentials
      AWS_REGION: ${AWS_REGION:-us-east-1}
    env_file:
      - .env.production   # non-sensitive app config
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16
    restart: always
    environment:
      POSTGRES_USER: apiuser
      POSTGRES_PASSWORD: ${DB_PASSWORD:?DB_PASSWORD must be set}
      POSTGRES_DB: myapp
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

```bash
# .env.production (committed with placeholder values marked, non-sensitive config)
LOG_LEVEL=info
API_PORT=8080
```

```bash
# deploy.sh — fetches secrets, never writes to disk
export DB_PASSWORD=$(aws secretsmanager get-secret-value ...)
export JWT_SECRET=$(aws secretsmanager get-secret-value ...)
export ECR_REGISTRY=123456789012.dkr.ecr.us-east-1.amazonaws.com
export APP_VERSION=1.4.2

docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## Summary

- The `.env` file in the project directory provides variables for **Compose YAML substitution** — it populates `${VARIABLE}` references in the Compose file itself
- `env_file:` in a service definition passes variables **directly to the container** — separate mechanism from the `.env` substitution file
- Variable precedence: inline `environment:` values > shell variables > `.env` file > Compose defaults
- Use `${VARIABLE:-default}` for optional variables with fallbacks and `${VARIABLE:?error}` for required variables that cause Compose to fail if unset
- Never commit `.env` files with real secrets to version control — always add `.env` to `.gitignore` and commit only a `.env.example` with placeholder values
- For production, fetch secrets from AWS Secrets Manager or Parameter Store at deploy time and inject them as shell environment variables — the secrets are never written to disk

In the next file, we cover volumes in Compose — the complete configuration options for the top-level volumes section and all volume mount patterns.
