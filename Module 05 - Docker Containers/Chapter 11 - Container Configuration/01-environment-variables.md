# Environment Variables

## Introduction

Environment variables are the standard way to pass configuration to Docker containers. They allow you to run the same image in different environments — development, staging, production — with different behavior, without rebuilding the image.

This file covers every method for passing environment variables to containers, how they interact with Dockerfile defaults, and the patterns used in real production deployments on AWS.

---

## Why Environment Variables for Configuration

The Twelve-Factor App methodology (the industry standard for cloud-native application design) says: **store configuration in the environment, not in code**.

Configuration that differs between deployments includes:
- Database connection strings
- API keys and secrets
- Feature flags
- Log levels
- Port numbers
- AWS region and resource names

If you hardcode these values in your code or Docker image, you need a different image for each environment. If you pass them as environment variables at runtime, one image works everywhere.

```bash
# Same image, different configuration
docker run -e DATABASE_URL=postgresql://dev-db/myapp myapp:1.4.2      # development
docker run -e DATABASE_URL=postgresql://staging-db/myapp myapp:1.4.2  # staging
docker run -e DATABASE_URL=postgresql://prod-db/myapp myapp:1.4.2     # production
```

---

## Setting Environment Variables at Runtime

### -e / --env — Single Variable

```bash
# Set a single variable
docker run -e APP_ENV=production myapp:1.0

# Set multiple variables
docker run \
  -e APP_ENV=production \
  -e LOG_LEVEL=info \
  -e PORT=8080 \
  -e DATABASE_HOST=db.example.com \
  -e DATABASE_PORT=5432 \
  -e DATABASE_NAME=myapp \
  myapp:1.0

# Set a variable to the value of a host environment variable
docker run -e DATABASE_PASSWORD myapp:1.0
# (no = sign means "use the value from the host's DATABASE_PASSWORD")
```

### --env-file — Load from a File

For many variables, an env file is much cleaner:

```bash
# .env file
APP_ENV=production
LOG_LEVEL=info
PORT=8080
DATABASE_HOST=db.example.com
DATABASE_PORT=5432
DATABASE_NAME=myapp
DATABASE_USER=apiuser
DATABASE_PASSWORD=supersecretpassword
JWT_SECRET=anothersecretvalue
AWS_REGION=us-east-1
S3_BUCKET=myapp-uploads-production

# Use the env file
docker run --env-file .env myapp:1.0

# Use a different env file for each environment
docker run --env-file .env.staging myapp:1.0
docker run --env-file .env.production myapp:1.0
```

### .env File Format Rules

```bash
# Lines starting with # are comments
# APP_NAME=this is ignored

# Key=value pairs (no spaces around =)
APP_ENV=production
LOG_LEVEL=info

# Values with spaces need quotes
APP_DESCRIPTION="My Production API"

# Empty values are allowed
OPTIONAL_FEATURE=

# Shell variable expansion does NOT work in env files
# DATABASE_URL=postgresql://${DB_USER}:${DB_PASS}@${DB_HOST}/myapp  ← does not work
# Write the full value explicitly
DATABASE_URL=postgresql://user:pass@db.example.com:5432/myapp
```

---

## Environment Variables from the Dockerfile

### ENV — Default Values

The `ENV` instruction in a Dockerfile sets default environment variable values. These defaults are used if no override is provided at runtime.

```dockerfile
ENV APP_ENV=development
ENV LOG_LEVEL=info
ENV PORT=8080
```

At runtime, you can override these defaults:

```bash
# Uses Dockerfile default: APP_ENV=development
docker run myapp:1.0

# Overrides the default
docker run -e APP_ENV=production myapp:1.0
```

### The Precedence Chain

When the same variable appears in multiple places, this is the order of precedence (higher overrides lower):

1. `-e` flag at runtime (highest priority)
2. `--env-file` at runtime
3. `ENV` in the Dockerfile (lowest priority)

```bash
# Dockerfile has: ENV DATABASE_HOST=localhost
# env.file has:   DATABASE_HOST=db.example.com
# -e has:         DATABASE_HOST=prod-db.example.com

docker run \
  --env-file .env \
  -e DATABASE_HOST=prod-db.example.com \
  myapp:1.0

# Result: DATABASE_HOST=prod-db.example.com (the -e flag wins)
```

---

## Accessing Environment Variables in Your Application

### Python

```python
import os

# Read a required variable — raises KeyError if not set
database_url = os.environ["DATABASE_URL"]

# Read with a default fallback
app_env = os.environ.get("APP_ENV", "development")
port = int(os.environ.get("PORT", "8080"))
log_level = os.environ.get("LOG_LEVEL", "INFO")

# Check if a feature flag is enabled
debug_mode = os.environ.get("DEBUG", "false").lower() == "true"
```

### Node.js

```javascript
// Access environment variables via process.env
const databaseUrl = process.env.DATABASE_URL;
if (!databaseUrl) {
  throw new Error("DATABASE_URL environment variable is required");
}

const port = parseInt(process.env.PORT || "3000");
const appEnv = process.env.APP_ENV || "development";
const logLevel = process.env.LOG_LEVEL || "info";
```

### Validating Required Variables at Startup

Always validate that required environment variables are set when your application starts. Fail fast — do not let the application start with missing configuration and then crash mysteriously later.

```python
import os
import sys

REQUIRED_ENV_VARS = [
    "DATABASE_URL",
    "JWT_SECRET",
    "AWS_REGION",
    "S3_BUCKET",
]

def validate_environment():
    missing = []
    for var in REQUIRED_ENV_VARS:
        if not os.environ.get(var):
            missing.append(var)
    
    if missing:
        print(f"ERROR: Missing required environment variables: {', '.join(missing)}", file=sys.stderr)
        sys.exit(1)
    
    print("Environment validation passed")

# Call this at application startup
validate_environment()
```

---

## Secrets — What NOT to Do

Environment variables are not encrypted. They are visible:
- In `docker inspect` output
- In `/proc/<pid>/environ` on the host
- In ECS task definitions stored in the console
- In CloudTrail logs if you set them via AWS CLI

**Never put secrets directly in environment variables in a way that exposes them:**

```bash
# BAD — anyone with docker inspect access sees this
docker run -e DATABASE_PASSWORD=mysecretpassword myapp:1.0

# BAD — this is stored in your shell history
docker run -e DATABASE_PASSWORD=$(cat /etc/myapp/db_password) myapp:1.0
```

---

## Handling Secrets Properly

### AWS Secrets Manager (Recommended for Production)

Fetch secrets at application startup — never put them in environment variables or Dockerfiles:

```python
import boto3
import json
import os

def get_secrets(secret_name: str, region: str = "us-east-1") -> dict:
    client = boto3.client("secretsmanager", region_name=region)
    response = client.get_secret_value(SecretId=secret_name)
    return json.loads(response["SecretString"])

# At application startup, before accepting connections
secrets = get_secrets("myapp/production/database")

DATABASE_URL = f"postgresql://{secrets['username']}:{secrets['password']}@{secrets['host']}/{secrets['database']}"
```

The application uses IAM role permissions to access Secrets Manager — no credentials needed in the environment.

### AWS ECS — Secrets from Parameter Store or Secrets Manager

ECS supports injecting secrets from SSM Parameter Store or Secrets Manager into container environment variables at task launch time. The secrets are fetched by ECS and injected — your container sees them as normal environment variables, but they were never stored in your task definition in plaintext.

```json
{
  "containerDefinitions": [
    {
      "name": "api",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2",
      "environment": [
        {"name": "APP_ENV", "value": "production"},
        {"name": "LOG_LEVEL", "value": "info"}
      ],
      "secrets": [
        {
          "name": "DATABASE_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:myapp/production/db:password::"
        },
        {
          "name": "JWT_SECRET",
          "valueFrom": "arn:aws:ssm:us-east-1:123456789012:parameter/myapp/production/jwt_secret"
        }
      ]
    }
  ]
}
```

ECS fetches these secrets at task launch. The secrets are injected as environment variables. The ECS task role needs permission to read from Secrets Manager/Parameter Store.

---

## Inspecting Environment Variables in Running Containers

```bash
# List all environment variables in a container
docker exec api env

# List in sorted order
docker exec api env | sort

# Find a specific variable
docker exec api env | grep DATABASE

# Inspect via docker inspect (shows all env vars including Dockerfile defaults)
docker inspect --format '{{range .Config.Env}}{{println .}}{{end}}' api

# Show in JSON format
docker inspect --format '{{json .Config.Env}}' api | python3 -m json.tool
```

---

## Environment Variables in Docker Compose

In Docker Compose, environment variables can be set inline, from an env file, or from the host:

```yaml
version: "3.8"

services:
  api:
    image: myapp:1.0
    
    # Option 1: Inline values
    environment:
      - APP_ENV=production
      - LOG_LEVEL=info
      - PORT=8080
    
    # Option 2: From host environment (no value = use host's value)
    environment:
      - DATABASE_URL      # uses $DATABASE_URL from host shell
    
    # Option 3: From env file
    env_file:
      - .env
      - .env.production   # overrides values from .env

  worker:
    image: myapp:1.0
    environment:
      APP_ENV: production
      WORKER_CONCURRENCY: "4"
    env_file: .env
```

Compose automatically loads `.env` from the project directory. Variables in `.env` are available for substitution in the compose file:

```yaml
# .env
APP_VERSION=1.4.2
AWS_REGION=us-east-1

# docker-compose.yml
services:
  api:
    image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:${APP_VERSION}
    environment:
      - AWS_REGION=${AWS_REGION}
```

---

## Summary

- Environment variables are the standard way to pass configuration to Docker containers — the same image runs in all environments with different configuration
- Use `-e VAR=value` for individual variables and `--env-file .env` for many variables
- `ENV` in the Dockerfile sets defaults that are overridden by runtime `-e` and `--env-file` settings
- Always validate that required environment variables are set at application startup — fail fast rather than crashing mysteriously later
- Never put sensitive secrets in plain environment variables on the command line, in Dockerfiles, or in docker-compose.yml — use AWS Secrets Manager or SSM Parameter Store
- On ECS, use the `secrets` field in the task definition to inject secrets from Secrets Manager/Parameter Store — ECS fetches and injects them securely at task launch

In the next file, we cover port mapping — how to expose container services to the host network and to other containers.
