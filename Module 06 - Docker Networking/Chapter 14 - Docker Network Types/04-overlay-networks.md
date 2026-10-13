# Overlay Networks

## Introduction

So far, we have discussed networks that exist on a single host machine (bridge, host, none). But modern applications are distributed. If you have a web server container on Server A and a database container on Server B, how do they communicate securely without exposing the database to the public internet?

The answer is the `overlay` network driver. Overlay networks allow containers running on completely different physical hosts to communicate as if they were plugged into the same local switch.

This file explains the mechanics of overlay networks, how they encrypt traffic, and their role in orchestration systems.

---

## What is an Overlay Network?

An overlay network is a virtual network built *on top of* an existing physical network. 

Imagine two physical servers, Host A and Host B, connected by the internet or a corporate LAN. 
- Host A runs Container 1 (IP: 10.0.9.2)
- Host B runs Container 2 (IP: 10.0.9.3)

These IPs (`10.0.9.x`) only exist inside Docker. The physical routers between Host A and Host B do not know how to route `10.0.9.x` traffic. 

Docker solves this using **encapsulation** (specifically, VXLAN - Virtual eXtensible Local Area Network). 

When Container 1 sends a packet to Container 2:
1. Docker intercepts the packet on Host A.
2. Docker wraps (encapsulates) the container packet inside a standard UDP packet addressed to Host B's *physical* IP address.
3. The physical network delivers the UDP packet to Host B.
4. Docker on Host B unwraps the packet and delivers the original payload to Container 2.

To the containers, they are simply talking to each other on a local `10.0.9.x` subnet. They are completely unaware of the physical network routing happening underneath.

---

## Requirements for Overlay Networks

You cannot simply create an overlay network on a standalone Docker installation. Overlay networks require a distributed state store to keep track of which containers are on which hosts, and what their virtual MAC and IP addresses are.

In the Docker ecosystem, this means **Docker Swarm mode must be enabled**.

```bash
# Initialize Docker Swarm on Host A (becomes a Manager node)
docker swarm init --advertise-addr <HOST_A_IP>

# (Host A outputs a join token)

# On Host B, join the Swarm (becomes a Worker node)
docker swarm join --token <TOKEN> <HOST_A_IP>:2377
```

Once the Swarm cluster is active, you can create overlay networks.

---

## Creating and Using an Overlay Network

Creating an overlay network is identical to creating a bridge network, but you change the driver.

```bash
# On the Manager node (Host A)
docker network create -d overlay my-distributed-net
```

Now, you can deploy services to the Swarm that use this network:

```bash
# Deploy a Redis database service
docker service create \
  --name redis-db \
  --network my-distributed-net \
  redis:alpine

# Deploy a Web API service that connects to Redis
docker service create \
  --name web-api \
  --network my-distributed-net \
  --replicas 3 \
  -p 80:80 \
  my-api:1.0
```

Swarm might schedule the `redis-db` container on Host B, and the three `web-api` containers across Host A and Host B. Because they are all on `my-distributed-net`, the API containers can resolve `redis-db` via DNS and connect to it, regardless of which physical servers they are actually running on.

---

## Overlay Network Encryption

By default, the control plane traffic (the management data keeping the Swarm synchronized) is encrypted using TLS. However, the **application data** flowing between containers over the VXLAN overlay is *unencrypted* by default.

If Host A and Host B are communicating over an untrusted network (like the public internet), you must encrypt the overlay network to protect your application traffic.

Docker supports native IPSec encryption for overlay networks. You simply add a flag during creation:

```bash
docker network create -d overlay --opt encrypted secure-overlay-net
```

When `--opt encrypted` is set, Docker establishes IPSec tunnels between all hosts participating in the network. All container-to-container traffic is encrypted before it leaves the physical host. 

*Trade-off:* IPSec encryption requires CPU overhead. Expect a performance drop in network throughput (often 10-30%) when encryption is enabled.

---

## Ingress Networking

When you publish a port in a Docker Swarm (e.g., `-p 80:80`), Docker uses a special overlay network called the **ingress network**. 

The ingress network provides a routing mesh. If you have a 5-node cluster and you publish port 80, port 80 opens on **all 5 physical nodes**. If a user sends a request to Node 1, but the actual container is running on Node 4, the ingress network intercepts the traffic on Node 1, routes it over the overlay to Node 4, and returns the response.

This built-in load balancing makes managing external traffic into a distributed cluster incredibly simple.

---

## Overlay Networks in AWS ECS

If you are using AWS Elastic Container Service (ECS), you will rarely use Docker's native overlay networks. 

AWS solves the multi-host networking problem using its own infrastructure. In ECS, using the **`awsvpc` network mode** assigns a real AWS Elastic Network Interface (ENI) to every container task. 

Because the container gets a real IP address from your VPC subnet, the containers communicate across EC2 hosts using the native AWS VPC routing fabric, rather than relying on Docker's software-based VXLAN encapsulation. This provides higher performance and deeper integration with AWS Security Groups and VPC Flow Logs.

Kubernetes operates similarly, relying on CNI (Container Network Interface) plugins like Calico, Flannel, or the AWS VPC CNI to handle multi-host overlay routing.

---

## Summary

- The `overlay` network driver allows containers on different physical hosts to communicate securely.
- It uses VXLAN encapsulation to tunnel container traffic over the physical network.
- It requires a clustered environment (Docker Swarm) to manage the distributed state.
- Application traffic is unencrypted by default, but can be secured with native IPSec by adding `--opt encrypted`.
- The `ingress` routing mesh is a specialized overlay network that load-balances external traffic across the cluster.
- In modern cloud environments like AWS ECS, native cloud networking (like `awsvpc` mode) is preferred over Docker's software overlay networks due to better performance and security integration.

In the next chapter, we will shift focus from network types to how containers actually use these networks to communicate with each other.
