# Private Container Registries

## Introduction

Managed registries like AWS ECR and Docker Hub cover most use cases. But some organizations cannot use cloud-managed registries due to air-gapped environments, data sovereignty requirements, regulatory compliance, or cost constraints. In these cases, you run your own private registry.

This file covers the self-hosted registry options — from the simple Docker Registry (distribution) container to enterprise-grade platforms like Harbor — and the operational considerations for running them in production.

---

## Docker Registry (Distribution) — The Official Self-Hosted Registry

The Docker Registry (`registry:2`) is an official Docker image that implements the Docker Registry HTTP API V2 — the same API used by Docker Hub and ECR. Any Docker client can push to and pull from it without modification.

### Basic Setup (Development Only)

```bash
# Run a local registry — no auth, no TLS — development only
docker run -d \
  --name registry \
  --restart unless-stopped \
  -p 5000:5000 \
  -v registry-data:/var/lib/registry \
  registry:2
```

```bash
# Push to the local registry
docker tag myapp:1.0 localhost:5000/myapp:1.0
docker push localhost:5000/myapp:1.0

# Pull from the local registry
docker pull localhost:5000/myapp:1.0
```

### Production Setup with TLS

Without TLS, Docker clients refuse to push/pull (unless the registry is explicitly listed as an insecure registry). For production, TLS is required.

```bash
# Generate a self-signed certificate (or use Let's Encrypt for public-facing registries)
mkdir -p /opt/registry/certs
openssl req -x509 -nodes -days 365 -newkey rsa:4096 \
  -keyout /opt/registry/certs/domain.key \
  -out /opt/registry/certs/domain.crt \
  -subj "/CN=registry.internal.example.com"
```

```yaml
# docker-compose.yml for production registry
version: "3.8"

services:
  registry:
    image: registry:2
    restart: always
    ports:
      - "443:5000"
    volumes:
      - /opt/registry/data:/var/lib/registry
      - /opt/registry/certs:/certs:ro
      - /opt/registry/auth:/auth:ro
      - /opt/registry/config.yml:/etc/docker/registry/config.yml:ro
    environment:
      REGISTRY_HTTP_ADDR: 0.0.0.0:5000
      REGISTRY_HTTP_TLS_CERTIFICATE: /certs/domain.crt
      REGISTRY_HTTP_TLS_KEY: /certs/domain.key
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: Registry Realm
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
```

```yaml
# /opt/registry/config.yml
version: 0.1
log:
  level: warn
storage:
  filesystem:
    rootdirectory: /var/lib/registry
  delete:
    enabled: true
  maintenance:
    uploadpurging:
      enabled: true
      age: 168h
      interval: 24h
      dryprun: false
http:
  addr: :5000
  tls:
    certificate: /certs/domain.crt
    key: /certs/domain.key
auth:
  htpasswd:
    realm: Registry Realm
    path: /auth/htpasswd
```

### Setting Up Authentication

```bash
# Create htpasswd file with bcrypt hashing
docker run --rm --entrypoint htpasswd httpd:alpine \
  -Bbn myuser mypassword > /opt/registry/auth/htpasswd

# Add additional users
docker run --rm --entrypoint htpasswd httpd:alpine \
  -Bbn anotheruser anotherpassword >> /opt/registry/auth/htpasswd
```

### Configuring Docker to Trust the Registry

```bash
# For self-signed certificates — add the CA cert to Docker's trust store
sudo mkdir -p /etc/docker/certs.d/registry.internal.example.com
sudo cp /opt/registry/certs/domain.crt \
  /etc/docker/certs.d/registry.internal.example.com/ca.crt

# Push and pull
docker login registry.internal.example.com
docker tag myapp:1.0 registry.internal.example.com/myapp:1.0
docker push registry.internal.example.com/myapp:1.0
```

### Insecure Registries (Development Only)

For local development, you can add the registry as insecure (no TLS):

```json
// /etc/docker/daemon.json
{
    "insecure-registries": [
        "localhost:5000",
        "192.168.1.100:5000"
    ]
}
```

Restart Docker after changing `daemon.json`. Never use insecure registries in production.

---

## Storing Registry Data in S3

The default registry storage driver writes to the local filesystem. For production, use S3 for durability:

```yaml
# docker-compose.yml with S3 storage
services:
  registry:
    image: registry:2
    environment:
      REGISTRY_STORAGE: s3
      REGISTRY_STORAGE_S3_REGION: us-east-1
      REGISTRY_STORAGE_S3_BUCKET: my-docker-registry-bucket
      REGISTRY_STORAGE_S3_ENCRYPT: "false"
      REGISTRY_STORAGE_S3_SECURE: "true"
      # Use IAM instance profile — no keys needed on EC2
```

With S3 storage, the registry is stateless — multiple registry containers can run behind a load balancer sharing the same S3 bucket.

---

## Harbor — Enterprise Registry

[Harbor](https://goharbor.io/) is a CNCF graduated project that builds on the Docker Registry with enterprise features:

- **Role-based access control (RBAC)** — organizations, projects, and user roles
- **Vulnerability scanning** — integrated Trivy and Clair scanning
- **Content trust** — image signing and verification
- **Replication** — sync images between Harbor instances or to ECR/GCR
- **Audit log** — full history of who pushed/pulled what and when
- **Quota management** — per-project storage limits
- **Immutable tags** — prevent tag overwrites
- **Web UI** — browse repositories, view scan results, manage users

### Quick Installation with Helm

```bash
# Install Harbor on Kubernetes via Helm
helm repo add harbor https://helm.goharbor.io

helm install harbor harbor/harbor \
  --namespace harbor \
  --create-namespace \
  --set externalURL=https://registry.mycompany.com \
  --set harborAdminPassword=adminpassword \
  --set persistence.enabled=true \
  --set persistence.persistentVolumeClaim.registry.size=200Gi
```

### Docker Compose Installation

Harbor also provides an offline installer that uses Docker Compose:

```bash
# Download the offline installer
wget https://github.com/goharbor/harbor/releases/download/v2.10.0/harbor-offline-installer-v2.10.0.tgz
tar xzf harbor-offline-installer-v2.10.0.tgz
cd harbor

# Edit harbor.yml with your domain, TLS certs, admin password, storage
cp harbor.yml.tmpl harbor.yml
vim harbor.yml

# Install
./install.sh --with-trivy   # includes Trivy for vulnerability scanning
```

---

## GitLab Container Registry

If your organization uses GitLab, its built-in container registry is often the simplest private registry option:

```bash
# Registry URL for a GitLab project
# gitlab.example.com:5005/namespace/project:tag

# Log in with GitLab credentials or a deploy token
docker login gitlab.example.com:5005 -u myuser -p mytoken

# Tag and push
docker tag myapp:1.0 gitlab.example.com:5005/mycompany/myapp:1.0
docker push gitlab.example.com:5005/mycompany/myapp:1.0
```

In GitLab CI, the registry is pre-authenticated automatically:

```yaml
# .gitlab-ci.yml
build:
  image: docker:24
  services:
    - docker:dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
```

The `CI_REGISTRY_USER`, `CI_REGISTRY_PASSWORD`, and `CI_REGISTRY_IMAGE` variables are automatically populated by GitLab CI.

---

## Registry Comparison

| Registry | Best For | Auth | Scanning | Cost |
|---|---|---|---|---|
| Docker Hub | Public images, small teams | Username/Token | Basic | Free tier / paid |
| AWS ECR | AWS deployments | IAM / OIDC | Enhanced (Inspector) | $0.10/GB/month |
| Docker Registry | Air-gapped, simple setups | htpasswd / custom | No | Free (self-hosted) |
| Harbor | Enterprise on-prem, RBAC | LDAP/OIDC/local | Trivy/Clair | Free (self-hosted) |
| GitLab Registry | GitLab CI users | GitLab tokens | Yes | Free with GitLab |
| GitHub GHCR | GitHub Actions users | GitHub tokens | Dependabot | Free for public |

---

## Summary

- Docker Registry (`registry:2`) is the self-hosted baseline — simple, widely compatible, but requires you to manage TLS, authentication, storage, and cleanup
- For production, use S3 as the storage backend — makes the registry stateless and highly available behind a load balancer
- Harbor is the enterprise-grade self-hosted option — adds RBAC, vulnerability scanning, image signing, replication, and a web UI on top of the base registry
- GitLab Container Registry is the zero-friction option for teams already on GitLab — CI jobs are pre-authenticated with no additional configuration
- For AWS workloads, always use ECR — it is managed, IAM-integrated, and the path of least resistance for ECS and Fargate deployments
- Self-hosted registries require ongoing operational work: TLS certificate renewal, authentication management, storage capacity planning, and garbage collection to remove unreferenced blobs

In the next file, we cover registry authentication — the patterns for securely logging in to registries across different environments.
