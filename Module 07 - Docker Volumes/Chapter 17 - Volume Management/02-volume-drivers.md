# Volume Drivers

## Introduction

Docker's default volume driver, `local`, stores data on the host machine's filesystem. This works well for single-host deployments, but it has a fundamental limitation: the data only exists on the specific host where the volume was created. If a container is scheduled to a different host — during auto-scaling, instance replacement, or cluster rebalancing — it cannot access the volume.

Volume drivers solve this by extending where Docker volumes can store data. Instead of the host's local disk, a volume driver can store data on an NFS server, a distributed storage cluster, or a cloud-native filesystem like AWS EFS or Azure Files.

This file covers how volume drivers work, the most relevant drivers for AWS environments, and how Docker Compose uses them.

---

## How Volume Drivers Work

A volume driver is a plugin that intercepts Docker volume operations (create, mount, unmount, remove) and implements them against a specific storage backend.

From Docker's perspective, all volumes behave the same way. The container mounts a volume, reads and writes files — Docker does not care whether those files are on local disk or streamed from a cloud storage system. The driver handles the implementation.

```bash
# List installed volume drivers (plugins)
docker plugin ls

# Install a third-party volume driver (example: rexray)
docker plugin install rexray/ebs

# Create a volume using a specific driver
docker volume create \
  --driver rexray/ebs \
  --opt size=20 \
  ebs-volume

# Mount it into a container — Docker calls the driver to handle the mount
docker run -v ebs-volume:/data myapp:1.0
```

---

## The local Driver — Default and Advanced Usage

The `local` driver is not just for simple host directories. With the right options, it can mount NFS shares, tmpfs, or specific host paths, all while presenting as a standard named volume.

### NFS Mount as a Named Volume

```bash
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=10.0.0.50,rw,noatime \
  --opt device=:/exports/myapp-data \
  nfs-appdata
```

Now `nfs-appdata` works like any other named volume, but the data is stored on an NFS server at `10.0.0.50`. Any container that mounts `nfs-appdata` accesses the same NFS share, even across multiple Docker hosts.

### CIFS/SMB Mount (Windows File Server)

```bash
docker volume create \
  --driver local \
  --opt type=cifs \
  --opt o=username=smbuser,password=smbpassword,domain=MYDOMAIN \
  --opt device=//fileserver.internal/share \
  cifs-share
```

---

## AWS EFS as a Docker Volume

AWS Elastic File System (EFS) is a managed NFS service. Multiple EC2 instances can mount the same EFS filesystem simultaneously, making it the production-grade shared storage solution for ECS clusters and Docker Swarm deployments.

### Mounting EFS Directly via the local Driver

If the EFS filesystem is already mounted on the host (via the EFS mount helper or NFS client), you can expose it as a named volume:

```bash
# First, mount EFS on the host using the EFS mount helper
sudo mount -t efs -o tls fs-0a1b2c3d4e5f6789:/ /mnt/efs

# Create a volume that points to the EFS mount path
docker volume create \
  --driver local \
  --opt type=none \
  --opt o=bind \
  --opt device=/mnt/efs/myapp-data \
  efs-appdata

# Use it in containers
docker run -v efs-appdata:/app/data myapp:1.0
```

Every Docker host in your cluster with EFS mounted can mount the `efs-appdata` volume and access the same shared data.

### EFS in ECS Task Definitions (Recommended for ECS)

When using AWS ECS (especially Fargate), you do not manage volume drivers manually. EFS is a first-class volume type in ECS task definitions:

```json
{
  "family": "myapp-task",
  "volumes": [
    {
      "name": "app-data",
      "efsVolumeConfiguration": {
        "fileSystemId": "fs-0a1b2c3d4e5f6789",
        "rootDirectory": "/myapp",
        "transitEncryption": "ENABLED",
        "transitEncryptionPort": 2049,
        "authorizationConfig": {
          "accessPointId": "fsap-0987fedcba654321",
          "iam": "ENABLED"
        }
      }
    }
  ],
  "containerDefinitions": [
    {
      "name": "api",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2",
      "mountPoints": [
        {
          "containerPath": "/app/data",
          "sourceVolume": "app-data",
          "readOnly": false
        }
      ]
    }
  ]
}
```

ECS mounts the EFS filesystem directly into the container's network namespace (using the EFS CSI driver). The task IAM role controls access to the EFS access point. Traffic between the container and EFS is encrypted in transit by default.

---

## The rexray Driver Suite

REX-Ray is an open-source storage orchestration engine that provides Docker volume drivers for many cloud and on-premises storage backends. It translates Docker volume API calls into calls to storage APIs like AWS EBS, S3, Azure Disk, or GCE Persistent Disk.

### rexray/ebs — AWS EBS Volumes

```bash
# Install the EBS driver plugin
docker plugin install rexray/ebs \
  EBS_ACCESSKEY=AKIAIOSFODNN7EXAMPLE \
  EBS_SECRETKEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY \
  EBS_REGION=us-east-1

# Create a 20GB EBS volume via Docker
docker volume create \
  --driver rexray/ebs \
  --opt size=20 \
  --opt volumetype=gp3 \
  myapp-database

# Mount it in a container
docker run -d \
  --name db \
  -v myapp-database:/var/lib/postgresql/data \
  postgres:16
```

When the container is removed, the EBS volume persists. When a new container is started on any EC2 instance in the same region, rexray detaches the EBS volume from the old instance and attaches it to the new one before mounting.

**Limitation:** An EBS volume can only be attached to one EC2 instance at a time. This makes rexray/ebs suitable for single-writer scenarios (like a database) but not for shared multi-writer storage.

---

## The convoy Driver

Convoy is an older but still-used Docker volume driver that supports snapshotting via device mapper or NFS.

```bash
# Install convoy
docker plugin install rancher/convoy

# Create a volume with snapshot capability
docker volume create --driver convoy myapp-db
```

Convoy's main advantage is point-in-time snapshots — you can snapshot a running volume and restore from it without stopping the container.

---

## Volume Drivers in Docker Compose

```yaml
version: "3.8"

services:
  db:
    image: postgres:16
    volumes:
      - postgres-data:/var/lib/postgresql/data

  api:
    image: myapp:1.0
    volumes:
      - app-uploads:/app/uploads

volumes:
  postgres-data:
    driver: local
    driver_opts:
      type: nfs
      o: "addr=10.0.0.50,rw"
      device: ":/exports/postgres"

  app-uploads:
    driver: local
    driver_opts:
      type: nfs
      o: "addr=10.0.0.50,rw"
      device: ":/exports/uploads"
```

Both containers in this Compose stack use NFS-backed volumes. When deployed to any host in the cluster, they all access the same shared NFS storage.

---

## Choosing the Right Storage for AWS

| Use Case | Recommended Storage | Reason |
|---|---|---|
| Database on single host | `local` named volume | Simple, fast, host-local SSD |
| Database on ECS Fargate | EFS task definition volume | EFS is managed, encrypted, persists across task replacements |
| Shared user uploads (ECS) | EFS task definition volume | Multiple tasks can read/write simultaneously |
| Shared user uploads (S3) | Application reads/writes S3 directly | Best practice — application code uses AWS SDK, not Docker volumes |
| Build cache on CI | `local` named volume | Ephemeral but fast; acceptable to lose on runner replacement |
| Cross-EC2 shared data | EFS via NFS driver | Only managed NFS that persists across instance replacements |

**AWS Best Practice Note:** For user uploads and large file storage in production on ECS, the most scalable pattern is to avoid Docker volumes entirely. Application containers write files directly to S3 using the AWS SDK. This removes the storage layer from Docker completely and scales to any number of containers without a shared mount.

---

## Summary

- Volume drivers extend where Docker volumes store data — beyond the local host filesystem to NFS, cloud storage, and distributed systems
- The `local` driver with `--opt type=nfs` options allows named volumes to back a standard NFS share, enabling shared storage across Docker hosts
- AWS EFS is the recommended persistent storage for ECS/Fargate — it is managed, encrypted, HA, and mounts directly into containers via ECS task definition volume configuration
- The `rexray/ebs` driver provides automatic EBS volume attachment across EC2 instances but is limited to single-writer access (one instance at a time)
- In Docker Compose, `driver_opts` in the `volumes:` block configures the driver transparently — services use the volume by name without caring about the underlying storage
- For user uploads and large files in AWS, the most scalable approach is writing directly to S3 via the AWS SDK rather than using Docker volumes — remove the filesystem dependency entirely

This completes Module 7 on Docker Volumes. You now have the full picture of Docker's storage system, from ephemeral writable layers to cloud-native persistent storage.
