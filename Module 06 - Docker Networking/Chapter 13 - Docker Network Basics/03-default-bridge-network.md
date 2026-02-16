# The Default Bridge Network

## Introduction

When you run a container without specifying a network, Docker automatically attaches it to the default bridge network. This network, managed by a virtual interface on the host named `docker0`, is the entry point to Docker networking.

While it is convenient for quick tests, the default bridge has significant limitations. In production DevOps environments, you should **never** rely on the default bridge network. 

This file explains how the default bridge works, how to inspect it, and why it is considered an anti-pattern for real-world applications.

---

## The docker0 Interface

When the Docker daemon starts, it creates a virtual ethernet bridge on the host machine called `docker0`. 

You can view this interface on a Linux host using standard networking commands:

```bash
# View the docker0 interface on the host
ip addr show docker0

# Output snippet:
# 3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
#     inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
```

Notice that the bridge itself has an IP address: `172.17.0.1`. This acts as the **default gateway** for all containers connected to this network. When a container tries to reach the internet, the traffic flows through `172.17.0.1` and is NAT-routed out through the host's physical interface.

---

## Connecting Containers to the Default Bridge

Any container started without a `--network` flag is attached here.

```bash
# Start two background containers
docker run -d --name container-a alpine sleep 3600
docker run -d --name container-b alpine sleep 3600
```

Let's inspect the default bridge network to see them:

```bash
docker network inspect bridge
```

In the JSON output, look for the `Containers` block:

```json
"Containers": {
    "a1b2c3d4...": {
        "Name": "container-a",
        "EndpointID": "...",
        "MacAddress": "02:42:ac:11:00:02",
        "IPv4Address": "172.17.0.2/16",
        "IPv6Address": ""
    },
    "e5f6g7h8...": {
        "Name": "container-b",
        "EndpointID": "...",
        "MacAddress": "02:42:ac:11:00:03",
        "IPv4Address": "172.17.0.3/16",
        "IPv6Address": ""
    }
}
```

Docker assigned `172.17.0.2` to `container-a` and `172.17.0.3` to `container-b`.

---

## IP Communication Works

Because both containers are on the same bridge, they can ping each other using their IP addresses.

```bash
# Ping container-b from container-a using its IP
docker exec container-a ping -c 3 172.17.0.3

# Output:
# PING 172.17.0.3 (172.17.0.3): 56 data bytes
# 64 bytes from 172.17.0.3: seq=0 ttl=64 time=0.088 ms
# 64 bytes from 172.17.0.3: seq=1 ttl=64 time=0.074 ms
```

Communication is successful. So why is the default bridge bad for production?

---

## The Fatal Flaw: No Automatic DNS Resolution

In a distributed application, containers are ephemeral. They crash, restart, or get replaced during deployments. When a container restarts, Docker assigns it a **new IP address**. 

If `container-a` is an API server and `container-b` is a database, hardcoding `172.17.0.3` in the API's configuration will break the moment the database container restarts and gets `172.17.0.4`.

We need DNS to resolve container names (e.g., `ping database`) to whatever IP address they currently have.

**The default bridge network DOES NOT provide automatic DNS resolution between containers.**

```bash
# Try to ping container-b by its name
docker exec container-a ping -c 3 container-b

# Output:
# ping: bad address 'container-b'
```

It fails. Containers on the default bridge can only communicate via raw IP addresses or by using deprecated, legacy features like `--link` (which you should never use).

---

## The Solution: User-Defined Bridge Networks

To get automatic DNS resolution, you must create a custom (user-defined) bridge network.

```bash
# 1. Create a custom network
docker network create my-app-net

# 2. Run containers on the custom network
docker run -d --name api --network my-app-net alpine sleep 3600
docker run -d --name db --network my-app-net alpine sleep 3600

# 3. Ping by name
docker exec api ping -c 3 db

# Output:
# PING db (172.18.0.3): 56 data bytes
# 64 bytes from 172.18.0.3: seq=0 ttl=64 time=0.088 ms
```

It works instantly. Docker runs an embedded DNS server at `127.0.0.11` inside custom networks that automatically resolves container names to their current IP addresses.

---

## Additional Limitations of the Default Bridge

Aside from the lack of DNS, the default bridge has other operational drawbacks:

1. **Lack of Isolation:** Every container started without a network flag lands on the default bridge. If you run a database, a web app, and an unrelated CI runner on the same host, they can all reach each other via IP on the default bridge. Custom networks isolate application stacks from each other.
2. **Difficult Configuration:** Changing the subnet, gateway, or MTU of the default bridge requires modifying the Docker daemon configuration (`/etc/docker/daemon.json`) and restarting the Docker service. Custom networks can be configured individually via the `docker network create` command without affecting the daemon.
3. **Implicit Dependencies:** Relying on the default bridge hides network dependencies. Explicitly defining a network (e.g., in a `docker-compose.yml`) makes the architectural boundaries of your application clear.

---

## Summary

- The default bridge network (`docker0`) is where containers land if you do not specify a `--network`.
- The host acts as the gateway (`172.17.0.1`) and provides NAT outbound access.
- Containers on the default bridge can ping each other by IP address, but **cannot** resolve each other by container name.
- Because IP addresses change when containers restart, communication via IP is fragile and unusable in production.
- **Best Practice:** Never use the default bridge in production. Always create user-defined networks to benefit from automatic DNS resolution and better security isolation.

In the next chapter, we will look at how to properly implement and manage user-defined bridge networks.
