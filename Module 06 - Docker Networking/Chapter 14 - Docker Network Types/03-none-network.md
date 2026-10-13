# The None Network

## Introduction

At the far end of the Docker networking spectrum is the `none` network driver. While the `host` driver provides maximum connectivity, the `none` driver provides absolute network isolation. 

When a container is attached to the `none` network, it is completely disconnected from all other containers, the host machine, and the outside internet.

This file explains how the `none` network functions, why you would intentionally break a container's network access, and the highly secure use cases it supports.

---

## How the None Network Works

When you run a container with `--network none`, Docker creates a dedicated network namespace for the container to ensure isolation, but it intentionally does not configure any network interfaces within that namespace (except for the standard loopback interface).

```bash
# Run an Alpine container with no network
docker run -it --network none alpine sh
```

If we inspect the networking stack from inside the container:

```bash
# Inside the container
ip addr show

# Output:
# 1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
#     link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
#     inet 127.0.0.1/8 scope host lo
#        valid_lft forever preferred_lft forever
```

Notice there is no `eth0`. The container only has `lo` (127.0.0.1). 

If you try to reach the outside world:

```bash
# Try to ping Google
ping 8.8.8.8
# ping: sendto: Network is unreachable

# Try to ping the host machine
ping 192.168.1.50
# ping: sendto: Network is unreachable
```

The container is a digital island. It cannot make outbound network requests, and it cannot receive inbound network requests. Port mapping (`-p`) is ignored because there is no interface for the traffic to route to.

---

## Why Use the None Network?

It might seem counterintuitive to run a container that cannot communicate. However, network isolation is the ultimate form of security. If a process does not need network access, removing the network stack entirely eliminates entire classes of vulnerabilities (e.g., data exfiltration, remote command execution payloads downloading additional malware).

### Use Case 1: Secure Data Processing
Imagine a container whose job is to take a batch of unencrypted credit card data, encrypt it using a public key, and write the encrypted data back to disk. 

If this container is compromised, the attacker might try to transmit the unencrypted data to an external server. If the container is on the `none` network, data exfiltration via the network is impossible.

How does the container get the data without a network? Via **bind mounts** or **volumes**.

```bash
docker run --rm \
  --network none \
  -v /data/input:/input:ro \
  -v /data/output:/output:rw \
  secure-encryptor:1.0 \
  encrypt_data --in /input/cards.csv --out /output/encrypted.dat
```
The container reads from a local directory, processes the data, writes to a local directory, and exits. Zero network exposure.

### Use Case 2: Generating Cryptographic Keys
When generating root SSL certificates, PGP keys, or cryptocurrency wallets, maximum paranoia is required. These tasks are often performed on "air-gapped" machines. The `none` network provides a software-defined air-gap.

### Use Case 3: Running Untrusted Code
If you are building a platform like a coding challenge website (e.g., LeetCode, HackerRank), users submit raw code to be executed. You must assume this code is malicious.

Running the untrusted code in a container with `--network none` ensures the code cannot launch DDoS attacks, mine cryptocurrency via network pools, or scan your internal network infrastructure.

---

## Custom Networking (Advanced)

The `none` driver is also used by advanced networking engineers who want to build entirely custom network topologies. 

When you use `--network none`, Docker leaves the network namespace completely empty. This allows an orchestrator or a custom script on the host to manually inject virtual interfaces (veth pairs), assign IPs using custom DHCP logic, and write custom routing tables using standard Linux `ip netns` commands.

```bash
# Start container with no network
docker run -d --name custom-net-container --network none alpine sleep 3600

# Get the container PID
PID=$(docker inspect -f '{{.State.Pid}}' custom-net-container)

# ... Proceed to manually wire interfaces using `ip link` and `ip netns` ...
```
This is how third-party networking plugins (like Calico or Cilium in Kubernetes) take over networking from the container runtime.

---

## AWS ECS Context

In AWS Elastic Container Service (ECS), you can set the `networkMode` in a task definition to `none`.

```json
{
  "family": "secure-processor-task",
  "networkMode": "none",
  "containerDefinitions": [
    {
      "name": "data-processor",
      "image": "my-secure-app:1.0"
    }
  ]
}
```

If you use `none` in ECS, you must rely entirely on EFS (Elastic File System) volumes or EC2 instance bind mounts to move data in and out of the task, as it will not be able to pull data from S3, reach RDS databases, or communicate with AWS APIs. 

*(Note: AWS Fargate does not support the `none` network mode. Fargate tasks must use `awsvpc` mode so AWS can route traffic to them).*

---

## Summary

- The `none` network driver provides total, impenetrable network isolation.
- The container has no network interfaces other than localhost. Port mapping (`-p`) is ignored.
- It is the ultimate security posture for tasks that do not require external connectivity.
- Use it for highly secure data processing, cryptographic operations, or executing untrusted user code.
- Data must be passed in and out of the container using volumes or bind mounts.
- It serves as a blank canvas for engineers who want to manually configure custom Linux networking namespaces.

In the next file, we will explore the `overlay` network, which solves the problem of networking containers across multiple physical machines.
