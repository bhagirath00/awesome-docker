# Network Drivers

## Introduction

Docker networking is not a single, monolithic system. It is built on a pluggable architecture. The core Docker engine relies on **network drivers** to handle the actual creation, configuration, and routing of container networks. 

When you start a container or create a new network, you choose a driver. The driver determines whether the container is isolated, whether it shares the host's IP address, or whether it spans across multiple physical servers.

This file provides a comprehensive breakdown of the core Docker network drivers and when to use each in a DevOps workflow.

---

## The Default Network Drivers

When you install Docker, it automatically creates three default networks, each using a different driver. You can see these with the `docker network ls` command:

```bash
$ docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
1a2b3c4d5e6f   bridge    bridge    local
2b3c4d5e6f7a   host      host      local
3c4d5e6f7a8b   none      null      local
```

You cannot remove these default networks. They represent the three most fundamental ways a container can interact with the network.

---

## Driver 1: bridge

The `bridge` driver is the most common network driver. It is the default driver used when you do not specify one. 

A bridge network creates a private, isolated software bridge (like a virtual network switch) on the host machine. Containers connected to the same bridge network can communicate with each other, but they are isolated from containers on different bridge networks.

### Characteristics:
- **Scope:** Single host (containers must be on the same physical server/VM)
- **Isolation:** High (containers have their own IP addresses and network namespaces)
- **Port Mapping:** Required (must use `-p` to expose services to the outside world)
- **DNS:** Automatic DNS resolution between containers (only on *user-defined* bridges, not the default `bridge`)

### When to use:
- Running single-host application stacks (e.g., a web server and a database on the same VM)
- Local development environments
- Docker Compose deployments (Compose creates a custom bridge network by default)

```bash
# Explicitly use the bridge driver (though it is the default)
docker network create --driver bridge my-app-net
docker run --network my-app-net nginx
```

---

## Driver 2: host

The `host` driver removes network isolation between the container and the Docker host. The container does not get its own IP address or network namespace. Instead, it uses the host's networking stack directly.

If you run a web server container on port 80 using the `host` network, the web server is instantly available on port 80 of the host machine's IP address.

### Characteristics:
- **Scope:** Single host
- **Isolation:** None (shares the host's network interfaces, IP, and routing tables)
- **Port Mapping:** Ignored (`-p` does nothing because the container already has direct access to the host's ports)
- **Performance:** Very high (bypasses the overhead of network address translation (NAT) and virtual bridges)

### When to use:
- High-performance applications that handle massive network throughput and need to avoid NAT overhead
- Applications that need to bind to a large range of ports (where mapping them individually with `-p` is impractical)
- Network monitoring tools that need to observe the host's actual network traffic

```bash
# Run a container using the host network
docker run -d --network host nginx

# Accessible immediately on the host's port 80 without -p
curl http://localhost:80
```

**Warning:** Because the container uses the host's network, port conflicts are possible. You cannot run two containers on the `host` network that both try to listen on port 80.

---

## Driver 3: none (null)

The `none` driver (internally called the `null` driver) provides total network isolation. A container attached to the `none` network gets its own network namespace, but Docker does not configure any network interfaces for it (other than the standard `loopback` interface, i.e., `localhost`).

The container cannot reach the internet, it cannot reach other containers, and nothing can reach it.

### Characteristics:
- **Scope:** Single container
- **Isolation:** Absolute
- **Interfaces:** Only `lo` (127.0.0.1)

### When to use:
- Maximum security environments where a container performs offline processing (e.g., encrypting a file, generating a password hash) and strictly must not transmit data anywhere
- Scenarios where you want to manually configure a custom network stack (using `ip netns` commands from the host)

```bash
# Run a container with no network access
docker run -it --network none alpine sh

# Inside the container, try to ping out
ping 8.8.8.8
# ping: sendto: Network is unreachable

# Check interfaces
ip a
# Shows only the loopback (lo) interface
```

---

## Driver 4: overlay

The `overlay` driver is used for multi-host networking. It creates a distributed network that sits on top of the host-specific networks, allowing containers running on entirely different physical machines to communicate securely as if they were on the same local switch.

Docker routes the traffic seamlessly between the hosts using VXLAN encapsulation.

### Characteristics:
- **Scope:** Multi-host cluster
- **Isolation:** High (encrypted communication is possible)
- **Prerequisites:** Requires a clustered environment (Docker Swarm)

### When to use:
- Docker Swarm clusters where services need to communicate across multiple EC2 instances
- *Note:* In AWS ECS, you typically use `awsvpc` mode instead of Docker overlay networks. Kubernetes uses its own CNI (Container Network Interface) plugins (like Calico or Flannel) instead of Docker overlay networks.

```bash
# Creating an overlay network (requires Docker Swarm to be initialized)
docker network create --driver overlay my-distributed-net
```

---

## Driver 5: macvlan

The `macvlan` driver allows you to assign a physical MAC address to a container, making it appear as a physical device directly connected to your local network. 

The Docker engine routes traffic to containers by their MAC addresses.

### Characteristics:
- **Scope:** Local network
- **Isolation:** Appears as a physical peer on the network
- **Complexity:** High (requires specific subnet routing and host interface configuration)

### When to use:
- Legacy applications that expect to be directly connected to the physical network and cannot function behind a bridge or NAT
- Very specific network monitoring or routing appliance containers

```bash
# Example creation (requires specifying parent physical interface and subnet)
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 pub_net
```

---

## Summary

- **bridge** is the default. It provides an isolated, NAT-routed network on a single host. Use it for most standard deployments and local development.
- **host** removes isolation. The container uses the host's IP and ports directly. Use it for high-performance networking or when port mapping is too complex.
- **none** provides total isolation. The container has no network access. Use it for highly secure, offline processing tasks.
- **overlay** connects containers across multiple physical machines. Used primarily in Docker Swarm.
- **macvlan** assigns a MAC address to a container, making it look like a physical device on the network. Used for legacy apps requiring direct network attachment.

In the next file, we will explore the default bridge network (`docker0`) in detail and explain why you should almost never use it in production.
