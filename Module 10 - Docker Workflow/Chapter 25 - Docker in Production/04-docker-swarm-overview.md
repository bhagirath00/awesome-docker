# Docker Swarm Overview

## Introduction

Docker Swarm is Docker's native clustering and orchestration system. It lets you manage a cluster of Docker hosts as a single logical unit — deploying services, scaling containers across multiple machines, distributing traffic via load balancing, and performing rolling updates with zero downtime.

Swarm is built into Docker itself — no additional software to install. While Kubernetes has become the dominant orchestrator for complex enterprise deployments, Swarm remains a practical choice for teams that want multi-host orchestration without Kubernetes' operational complexity.

---

## When to Use Docker Swarm

**Swarm is a good fit when:**
- You need multi-host orchestration without Kubernetes complexity
- Your team is already comfortable with Docker Compose (Swarm uses the same file format)
- You have a small to medium cluster (2-50 nodes)
- You are migrating from single-host Docker Compose to multi-host

**Kubernetes (EKS) is a better fit when:**
- You are deploying on AWS and want native ECS/Fargate integration
- You need advanced scheduling (affinity, taints, tolerations)
- You need Kubernetes-specific ecosystem tools (Helm, Istio, ArgoCD)
- Your cluster will have 50+ nodes
- You need fine-grained RBAC and audit logging

**AWS ECS is a better fit when:**
- Your entire stack is on AWS
- You want zero cluster management overhead (Fargate)
- You need native IAM, CloudWatch, and ALB integration without configuration

---

## Swarm Architecture

A Docker Swarm cluster consists of two roles:

**Manager nodes** — control plane. Run the Swarm orchestration engine (Raft consensus), accept service deployments, schedule containers, maintain cluster state. An odd number of managers is required for high availability (1, 3, or 5 is typical).

**Worker nodes** — data plane. Run the actual containers. Do not participate in Swarm management decisions.

```
┌───────────────────────────────────────┐
│           Docker Swarm Cluster        │
│                                       │
│  ┌─────────────┐  ┌─────────────┐    │
│  │  Manager 1  │  │  Manager 2  │    │
│  │  (Leader)   │  │  (Follower) │    │
│  └─────────────┘  └─────────────┘    │
│                                       │
│  ┌──────────┐ ┌──────────┐ ┌──────┐ │
│  │ Worker 1 │ │ Worker 2 │ │  ... │ │
│  └──────────┘ └──────────┘ └──────┘ │
└───────────────────────────────────────┘
```

---

## Initializing a Swarm

```bash
# On the first manager node
docker swarm init --advertise-addr 10.0.0.1

# Output:
# Swarm initialized: current node (abc123) is now a manager.
# To add a worker to this swarm, run the following command:
#
#     docker swarm join --token SWMTKN-1-xyz... 10.0.0.1:2377
#
# To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.

# Get the worker join token
docker swarm join-token worker

# Get the manager join token
docker swarm join-token manager
```

```bash
# On worker nodes
docker swarm join --token SWMTKN-1-xyz... 10.0.0.1:2377

# Verify the cluster
docker node ls

# Output:
# ID           HOSTNAME     STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
# abc123 *     manager-1    Ready     Active         Leader           24.0.0
# def456       manager-2    Ready     Active         Reachable        24.0.0
# ghi789       worker-1     Ready     Active                          24.0.0
# jkl012       worker-2     Ready     Active                          24.0.0
```

---

## Swarm Services vs Containers

In Swarm, you do not run containers directly — you create **services**. A service is the definition of how many replicas of a container should run, how it should be updated, and how it should behave under failure.

```bash
# Create a service with 3 replicas
docker service create \
  --name myapp-api \
  --replicas 3 \
  --publish published=80,target=8080 \
  --network myapp-net \
  myapp:1.4.2

# List all services
docker service ls

# List tasks (instances) of a service
docker service ps myapp-api

# Scale the service
docker service scale myapp-api=5

# Update the service's image (rolling update)
docker service update \
  --image myapp:1.5.0 \
  --update-parallelism 1 \
  --update-delay 30s \
  myapp-api

# Remove the service
docker service rm myapp-api
```

---

## Swarm Networking — Overlay Networks

Swarm uses overlay networks to enable containers on different hosts to communicate with each other using service names (DNS).

```bash
# Create an overlay network
docker network create \
  --driver overlay \
  --attachable \
  myapp-net

# Create services on the overlay network
docker service create --name api --network myapp-net myapp:1.4.2
docker service create --name db --network myapp-net postgres:16
```

Containers across all worker nodes can reach each other by service name (`api`, `db`) via Docker's embedded DNS.

### Ingress Network — Load Balancing

Swarm's built-in routing mesh (`ingress` network) automatically distributes published port traffic across all service replicas, regardless of which node the container runs on:

```bash
# Any request to port 80 on ANY node is distributed to one of the 3 api replicas
docker service create \
  --name api \
  --replicas 3 \
  --publish published=80,target=8080 \
  myapp:1.4.2
```

A request to `http://worker-1:80` may be served by a container running on `worker-2`. Swarm's IPVS-based load balancer handles this transparently.

---

## Deploying with Docker Stack

Docker Stack extends Docker Compose to Swarm — you deploy a Compose file to the cluster:

```yaml
# docker-compose.prod.yml (also valid as a Swarm stack file)
version: "3.8"

services:
  api:
    image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:${APP_VERSION}
    deploy:
      replicas: 3
      update_config:
        parallelism: 1          # update one replica at a time
        delay: 30s              # wait 30s between each update
        failure_action: rollback # auto-rollback on failure
        order: start-first      # start new before stopping old (zero-downtime)
      rollback_config:
        parallelism: 0          # rollback all replicas simultaneously
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      resources:
        limits:
          cpus: "1.5"
          memory: 512M
        reservations:
          memory: 256M
      placement:
        constraints:
          - node.role == worker   # only run on worker nodes, not managers
    ports:
      - "8080:8080"
    networks:
      - app-net
    secrets:
      - db_password
    environment:
      APP_ENV: production
      DATABASE_URL: postgresql://apiuser:{{.}}@db:5432/myapp

  db:
    image: postgres:16
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.db == "true"   # only on nodes labeled for DB
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - app-net
    secrets:
      - db_password
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
      POSTGRES_USER: apiuser
      POSTGRES_DB: myapp

  nginx:
    image: nginx:1.25-alpine
    deploy:
      replicas: 2
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    networks:
      - app-net

volumes:
  pgdata:

networks:
  app-net:
    driver: overlay

secrets:
  db_password:
    external: true    # pre-created with: echo "mysecret" | docker secret create db_password -
```

```bash
# Deploy the stack
docker stack deploy -c docker-compose.prod.yml myapp

# List stacks
docker stack ls

# List services in the stack
docker stack services myapp

# List all tasks with placement info
docker stack ps myapp

# Remove the stack
docker stack rm myapp
```

---

## Swarm Secrets

Swarm provides a native secrets management system — secrets are encrypted at rest in the Raft log and only decrypted in memory inside containers that explicitly request them:

```bash
# Create a secret from stdin
echo "supersecretpassword" | docker secret create db_password -

# Create from a file
docker secret create ssl_cert ./certs/domain.crt

# List secrets (values are never shown)
docker secret ls

# Secrets are available inside the container at /run/secrets/SECRET_NAME
docker service create \
  --name api \
  --secret db_password \
  myapp:1.4.2

# Inside the container:
# cat /run/secrets/db_password
# supersecretpassword
```

Swarm secrets are significantly more secure than environment variables — they are not visible in `docker inspect`, not logged, and not stored in image layers.

---

## Rolling Updates and Rollbacks

```bash
# Update the service image with a rolling deployment
docker service update \
  --image myapp:1.5.0 \
  --update-parallelism 1 \    # update 1 replica at a time
  --update-delay 30s \        # wait 30s between updates
  --update-failure-action rollback \  # auto-rollback on failure
  myapp-api

# Monitor the update progress
docker service ps myapp-api

# Manual rollback if needed
docker service rollback myapp-api
```

---

## Node Management

```bash
# List all nodes with status
docker node ls

# Add a label to a node (for placement constraints)
docker node update --label-add db=true worker-1

# Drain a node before maintenance (stops scheduling new containers on it)
docker node update --availability drain worker-2

# Re-activate after maintenance
docker node update --availability active worker-2

# Remove a node from the swarm
docker node rm worker-3   # node must be drained first
```

---

## Swarm vs ECS Decision Matrix

| Factor | Docker Swarm | AWS ECS (Fargate) |
|---|---|---|
| Infrastructure management | You manage EC2 instances | AWS manages (Fargate) |
| Learning curve | Low (Compose-like) | Medium (ECS concepts) |
| AWS service integration | Manual setup | Native (ALB, IAM, CloudWatch) |
| Cost at low scale | Cheaper (reuse existing EC2) | Slightly higher (Fargate pricing) |
| Kubernetes migration path | Different architecture | EKS upgrade path |
| Operational overhead | High (OS patches, Docker updates) | Low (Fargate = serverless) |

**Recommendation for AWS teams:** Use ECS/Fargate for new projects. Swarm is valuable when you are on-premises, have an existing Swarm cluster, or want to minimize cloud vendor lock-in.

---

## Summary

- Docker Swarm is built into Docker — no additional installation, uses the same Compose file format for multi-host deployments via `docker stack deploy`
- A Swarm cluster has manager nodes (control plane, Raft consensus) and worker nodes (data plane, runs containers) — use an odd number of managers (1, 3, or 5) for high availability
- Services replace containers in Swarm — define desired replicas, update strategy, placement constraints, and resource limits in the `deploy:` block
- Overlay networks enable cross-host container DNS — services reach each other by name regardless of which node they run on
- The Swarm ingress routing mesh distributes published port traffic across all replicas automatically — any request to any node reaches the service
- Swarm secrets store credentials encrypted in the Raft log, mounted at `/run/secrets/` inside containers — significantly more secure than environment variables or bind-mounted files

This completes Module 10 — Docker Workflow. You now have a complete, production-grade understanding of Docker CI/CD pipelines, registry management, and production operations including logging, monitoring, security, and orchestration.
