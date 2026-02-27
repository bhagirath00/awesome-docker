# Registry Authentication

## Introduction

Every interaction with a private container registry — pushing images, pulling images, inspecting manifests — requires authentication. Getting authentication right is critical both for security (only authorized users and systems can access images) and for reliability (authentication failures break CI/CD pipelines and production deployments).

This file covers how Docker authentication works, where credentials are stored, and the correct patterns for each environment: developer machines, CI/CD pipelines, and production containers on ECS.

---

## How Docker Authentication Works

When you run `docker login`, Docker sends your credentials to the registry, receives an authorization token, and stores the token (or credential reference) in a local config file.

```bash
# Where Docker stores credentials
~/.docker/config.json

# Contents after login
{
    "auths": {
        "https://index.docker.io/v1/": {
            "auth": "base64(username:password)"
        },
        "123456789012.dkr.ecr.us-east-1.amazonaws.com": {
            "auth": "base64(AWS:eyJwYXlsb...)"
        }
    }
}
```

The `auth` field is Base64-encoded `username:password` — it is **not encrypted**. Anyone with access to this file has your registry credentials. On developer machines, this is mitigated by using a credential helper (keychain, keystore). In CI/CD and containers, this file is a security concern.

---

## Developer Machines

### Standard docker login

```bash
# Docker Hub
docker login
# Prompts for username and password

# ECR (AWS)
aws ecr get-login-password --region us-east-1 | \
    docker login --username AWS --password-stdin \
    123456789012.dkr.ecr.us-east-1.amazonaws.com

# Private registry
docker login registry.mycompany.com
# Prompts for username and password
```

### Credential Helpers — Secure Storage

Instead of storing credentials as Base64 in `config.json`, credential helpers delegate to the system's secure credential store:

```bash
# macOS — uses macOS Keychain
docker-credential-osxkeychain

# Windows — uses Windows Credential Manager
docker-credential-wincred

# Linux — uses gnome-libsecret or other keystore
docker-credential-secretservice

# Cross-platform — uses pass password store
docker-credential-pass
```

Docker Desktop for Mac and Windows configures the credential helper automatically. On Linux, configure manually:

```json
// ~/.docker/config.json — use credential helper instead of base64 storage
{
    "credsStore": "osxkeychain"    // macOS
    "credsStore": "secretservice"  // Linux with GNOME keyring
    "credsStore": "wincred"        // Windows
}
```

With a credential helper configured, `docker login` stores credentials in the system keystore — not in `config.json`. The `auths` section in `config.json` remains empty.

---

## CI/CD Pipeline Authentication

### GitHub Actions — ECR

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
    aws-region: us-east-1

- name: Login to Amazon ECR
  id: login-ecr
  uses: aws-actions/amazon-ecr-login@v2

# The action outputs the registry URL
- name: Push image
  run: |
    docker push ${{ steps.login-ecr.outputs.registry }}/myapp:${{ github.sha }}
```

### GitHub Actions — Docker Hub

```yaml
- name: Login to Docker Hub
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}   # Use access token, not password
```

### Jenkins — ECR

```groovy
withCredentials([
    usernamePassword(
        credentialsId: 'aws-credentials',
        usernameVariable: 'AWS_ACCESS_KEY_ID',
        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
    )
]) {
    sh '''
        aws ecr get-login-password --region us-east-1 | \
            docker login --username AWS --password-stdin \
            123456789012.dkr.ecr.us-east-1.amazonaws.com
    '''
}
```

### GitLab CI — Auto Authentication

```yaml
build:
  image: docker:24
  services:
    - docker:dind
  script:
    # GitLab pre-populates these variables
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
```

### AWS CodeBuild — IAM Role (No Explicit Auth Needed)

CodeBuild runs with an IAM service role. If the role has `ecr:GetAuthorizationToken` and ECR push/pull permissions, you still need to call `get-login-password` — but no credentials are needed because the IAM role provides them:

```bash
# CodeBuild: IAM role provides credentials automatically
aws ecr get-login-password --region $AWS_REGION | \
    docker login --username AWS --password-stdin $ECR_REGISTRY
```

---

## ECS and Fargate Authentication

ECS tasks pull images from ECR using the **task execution role** — not your credentials. No `docker login` needed anywhere in the deployment process.

The task execution role must have:

```json
{
    "Effect": "Allow",
    "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage"
    ],
    "Resource": "*"
}
```

This is included in `arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy`.

For pulling from a private registry (not ECR), store credentials in AWS Secrets Manager and reference them in the task definition:

```json
{
    "containerDefinitions": [
        {
            "name": "api",
            "image": "registry.mycompany.com/myapp:1.4.2",
            "repositoryCredentials": {
                "credentialsParameter": "arn:aws:secretsmanager:us-east-1:123456789012:secret:myregistry-credentials-AbCdEf"
            }
        }
    ]
}
```

The secret must contain:

```json
{
    "username": "myregistryuser",
    "password": "myregistrypassword"
}
```

ECS fetches the secret at task launch time and uses it to authenticate to the registry before pulling the image. The credentials are never stored on disk.

---

## Docker Config in Kubernetes

Kubernetes uses `imagePullSecrets` to authenticate to registries when pulling images for pods.

### Creating an Image Pull Secret for ECR

```bash
# Create a Kubernetes secret from ECR credentials
aws ecr get-login-password --region us-east-1 | \
kubectl create secret docker-registry ecr-secret \
    --docker-server=123456789012.dkr.ecr.us-east-1.amazonaws.com \
    --docker-username=AWS \
    --docker-password=$(aws ecr get-login-password --region us-east-1) \
    --namespace production
```

The ECR token expires after 12 hours — refresh it with a CronJob:

```yaml
# cronjob-ecr-token-refresh.yml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: ecr-token-refresh
spec:
  schedule: "0 */6 * * *"   # every 6 hours
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: ecr-token-refresher
          containers:
            - name: refresh
              image: amazon/aws-cli:latest
              command:
                - /bin/sh
                - -c
                - |
                  TOKEN=$(aws ecr get-login-password --region us-east-1)
                  kubectl create secret docker-registry ecr-secret \
                    --docker-server=123456789012.dkr.ecr.us-east-1.amazonaws.com \
                    --docker-username=AWS \
                    --docker-password=$TOKEN \
                    --dry-run=client -o yaml | kubectl apply -f -
```

On EKS, use the [ECR Credential Helper](https://github.com/awslabs/amazon-ecr-credential-helper) or IRSA (IAM Roles for Service Accounts) to avoid managing tokens at all.

---

## Security Best Practices for Registry Auth

### Use Access Tokens, Not Passwords

For Docker Hub, GitHub GHCR, and similar registries, always use **access tokens** rather than account passwords:

- Tokens can be scoped to specific permissions (read-only, read-write)
- Tokens can be rotated without changing your account password
- If a token leaks, you can revoke it without locking your account

### Rotate Credentials Regularly

```bash
# Rotate Docker Hub access token
# 1. Create a new token in Docker Hub → Security → Access Tokens
# 2. Update the secret in GitHub/Jenkins
# 3. Delete the old token
```

### Prefer IAM Roles Over Long-Lived Keys

For AWS services, use IAM roles instead of access keys:
- GitHub Actions: OIDC role assumption (no keys in secrets)
- CodeBuild: service role (no keys anywhere)
- ECS: task execution role (no keys on instances)
- EC2: instance profile (no keys on disk)

### Restrict Registry Permissions by Repository

Grant the minimum required permissions per repository:

```json
// CI role for myapp: can only push/pull the myapp repository
{
    "Effect": "Allow",
    "Action": ["ecr:BatchCheckLayerAvailability", "ecr:PutImage", ...],
    "Resource": "arn:aws:ecr:us-east-1:123456789012:repository/myapp"
}

// Deployment role for myapp: can only pull the myapp repository
{
    "Effect": "Allow",
    "Action": ["ecr:BatchCheckLayerAvailability", "ecr:GetDownloadUrlForLayer", "ecr:BatchGetImage"],
    "Resource": "arn:aws:ecr:us-east-1:123456789012:repository/myapp"
}
```

### Avoid docker login in Containers

Running `docker login` inside a container stores credentials in `~/.docker/config.json` inside the container. If the container image is later pushed or exported, those credentials are embedded in a layer.

Instead, pass credentials at build time using BuildKit secrets, or mount `config.json` as a volume at runtime:

```bash
# Mount host's docker config into the build container (CI pattern)
docker run --rm \
    -v ~/.docker:/root/.docker:ro \
    -v $(pwd):/workspace \
    docker:24 \
    sh -c "cd /workspace && docker push myapp:1.0"
```

---

## Summary

- Docker stores credentials in `~/.docker/config.json` as Base64-encoded text — use credential helpers (osxkeychain, secretservice, wincred) to store them in the system keystore instead
- In CI/CD, authenticate to ECR with `aws ecr get-login-password | docker login` — use IAM roles (OIDC, instance profile, service role) rather than long-lived access keys
- ECS and Fargate tasks authenticate to ECR automatically via the task execution role — no `docker login` needed anywhere in the deployment flow
- For ECS pulling from non-ECR private registries, store credentials in AWS Secrets Manager and reference the secret ARN in the task definition via `repositoryCredentials`
- In Kubernetes (non-EKS), create `imagePullSecret` objects and refresh them every 6 hours because ECR tokens expire after 12 hours
- Always use access tokens (not account passwords) for Docker Hub, GHCR, and other token-based registries — they can be scoped, rotated, and revoked independently

This completes Module 10 Chapter 24 on Docker Registries. In the next chapter, we cover Docker in production — logging, monitoring, security, and Swarm orchestration.
