# DNS in Docker

## Introduction

In modern microservices, containers must discover and communicate with each other dynamically. Because containers are ephemeral and IP addresses change on every restart, hardcoding IP addresses is impossible.

Docker solves this using an embedded DNS server. This server dynamically tracks the lifecycle of containers and ensures that resolving a container's name always returns its current IP address.

This file explains how Docker's internal DNS works, how it resolves external domains, and how to configure custom DNS behavior.

---

## The Embedded DNS Server

When you create a user-defined (custom) network, Docker provisions an embedded DNS server. This server listens on the IP address `127.0.0.11` inside the container's network namespace.

When a container makes a DNS request (e.g., trying to connect to `http://database:5432`), the request goes to `127.0.0.11`.

```bash
# Inside a container on a custom network
cat /etc/resolv.conf

# Output:
# search default.svc.cluster.local svc.cluster.local cluster.local
# nameserver 127.0.0.11
# options ndots:0
```

Notice that the `nameserver` is explicitly set to `127.0.0.11`. All DNS queries from the container hit Docker's internal DNS resolver first.

### How Resolution Works

1. **Internal Query:** The container queries `database`.
2. **Intercept:** The query goes to `127.0.0.11`.
3. **Lookup:** Docker checks if a container named `database` exists on the same custom network.
4. **Response:** If it does, Docker returns the internal IP (e.g., `172.18.0.2`).
5. **External Fallback:** If the name does not match any container, Docker forwards the query to the external DNS servers configured on the host machine (e.g., `8.8.8.8` or your corporate DNS).

*Note: As emphasized in previous chapters, the embedded DNS server is **not** active on the default `bridge` network. Containers on the default bridge fall back to external DNS immediately, which is why container-to-container name resolution fails there.*

---

## Network Aliases

A container's name is its primary DNS record. However, you can assign multiple DNS names to a single container using network aliases.

This is highly useful for refactoring or migrating services.

```bash
docker run -d \
  --name new-payment-processor \
  --network app-net \
  --network-alias payments-api \
  --network-alias legacy-billing \
  payment-processor:v2
```

Other containers on `app-net` can now resolve `new-payment-processor`, `payments-api`, and `legacy-billing`. All three DNS records will return the same container IP address.

### Aliases in Docker Compose

Docker Compose makes heavy use of aliases. By default, the name of the service block in the `docker-compose.yml` becomes the network alias.

```yaml
version: '3.8'
services:
  db:             # This becomes the DNS alias
    image: postgres
    container_name: dev-database-v1
```
Other services can connect to `postgres://user:pass@db:5432`. Even though the physical container name is `dev-database-v1`, Compose automatically provisions the alias `db`.

---

## Round-Robin DNS (Load Balancing)

If multiple containers are given the same network alias, Docker's embedded DNS server will return all their IP addresses in a randomized order. This provides basic Round-Robin DNS load balancing.

```bash
# Start three web workers with the same alias
docker run -d --name worker1 --network app-net --network-alias api-worker my-worker
docker run -d --name worker2 --network app-net --network-alias api-worker my-worker
docker run -d --name worker3 --network app-net --network-alias api-worker my-worker
```

If another container queries `api-worker`, Docker returns all three IP addresses. The client application will typically connect to the first IP in the list. Because the order is randomized on each request, the traffic is roughly distributed across the three containers.

*Warning: DNS round-robin is not a replacement for a true Layer 7 load balancer (like HAProxy, Nginx, or an AWS ALB). If `worker2` crashes, Docker removes it from DNS, but clients that have already cached the IP might still try to send traffic to the dead container.*

---

## Configuring Custom DNS

Sometimes, you need a container to use a specific external DNS server instead of relying on the host's configuration. You can pass DNS settings directly when running the container.

```bash
# Force the container to use Google's public DNS for external queries
docker run -d \
  --name web \
  --dns 8.8.8.8 \
  --dns 8.8.4.4 \
  nginx
```

**How this interacts with embedded DNS:**
Even if you pass `--dns 8.8.8.8`, Docker still injects `127.0.0.11` into `/etc/resolv.conf` to handle internal container-to-container resolution. However, if the query is for an external domain (e.g., `api.github.com`), the embedded DNS server will forward the query to `8.8.8.8` instead of the host's default DNS.

### Custom DNS Search Domains

You can also configure the search domain. If a container queries a bare hostname like `intranet`, the search domain is appended to it.

```bash
docker run -d \
  --name tools \
  --dns-search mycompany.local \
  alpine sleep 3600

# Inside the container, 'ping intranet' resolves to 'intranet.mycompany.local'
```

### DNS in Docker Compose

```yaml
version: '3.8'
services:
  web:
    image: nginx
    dns:
      - 8.8.8.8
      - 1.1.1.1
    dns_search:
      - mycompany.local
```

---

## AWS Context: Route 53 and Service Discovery

In AWS ECS (particularly with Fargate), Docker's embedded DNS is replaced by native AWS networking.

To achieve container-to-container resolution in ECS:
1. You enable **AWS Cloud Map (Service Discovery)** on your ECS Service.
2. You define a namespace, e.g., `myapp.internal`.
3. When ECS launches a task, it automatically registers the task's VPC IP address in Route 53 under `service-name.myapp.internal`.
4. The AWS VPC DNS Resolver (`169.254.169.253`) handles the queries.

While the backend technology changes from `127.0.0.11` to Route 53, the DevOps pattern remains identical: applications communicate using static service names, and the infrastructure dynamically updates the DNS records as containers scale up and down.

---

## Summary

- Docker provides an embedded DNS server (`127.0.0.11`) on all user-defined networks to enable automatic service discovery.
- The embedded DNS handles internal container names and forwards external queries to the host's DNS servers.
- `network-aliases` allow multiple DNS names to point to a single container, or multiple containers to share the same name (enabling simple round-robin DNS load balancing).
- Use the `--dns` flag to force a container to use specific external nameservers (like `8.8.8.8`), which overrides the host's default fallback but preserves internal container resolution.
- In AWS ECS, this pattern is replicated using AWS Cloud Map and Route 53 for native VPC service discovery.

This concludes Module 6 on Docker Networking. You now understand how containers communicate, how to secure them via segmentation, and how traffic is routed and resolved.
