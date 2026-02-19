# Volume Backup and Restore

## Introduction

A Docker volume that is not backed up is production data waiting to be lost. Disk failures, accidental `docker volume rm`, incorrect `docker compose down -v`, or a corrupted filesystem can permanently destroy months or years of data.

Backing up volumes is not optional in production. This file covers the practical techniques for backing up and restoring Docker volumes — from simple tar archives to production-grade backup pipelines using AWS S3.

---

## The Core Technique: Temporary Backup Container

The canonical way to back up a Docker volume is to run a temporary container that mounts the volume, creates an archive, and streams it somewhere safe. This approach works universally because the backup container uses standard Linux tools (`tar`, `gzip`) and makes no assumptions about what is inside the volume.

### Backup to a Local tar.gz File

```bash
# Backup the pgdata volume to a compressed archive on the host
docker run --rm \
  -v pgdata:/source:ro \
  -v $(pwd):/backup \
  alpine \
  tar czf /backup/pgdata-backup-$(date +%Y%m%d-%H%M%S).tar.gz -C /source .
```

What this does step by step:
1. `docker run --rm` — starts a temporary container, removes it when done
2. `-v pgdata:/source:ro` — mounts the volume to back up as read-only
3. `-v $(pwd):/backup` — mounts the current host directory for output
4. `alpine` — uses the small Alpine Linux image which has `tar`
5. `tar czf /backup/pgdata-backup-....tar.gz -C /source .` — creates a compressed archive of all volume contents

Result: a file like `pgdata-backup-20240115-092341.tar.gz` in your current directory.

### Restore from a Local tar.gz File

```bash
# Restore volume data from an archive
# (creates the volume if it does not exist)
docker run --rm \
  -v pgdata:/target \
  -v $(pwd):/backup \
  alpine \
  tar xzf /backup/pgdata-backup-20240115-092341.tar.gz -C /target
```

This extracts the archive into the `pgdata` volume, fully restoring its contents.

---

## Stopping the Application Before Backup

For application data (user uploads, generated files), running the backup while the application is writing may produce a consistent backup. For databases, it will not.

**Never back up a running database volume with tar.** A live PostgreSQL or MySQL data directory that is being written to during the backup will be internally inconsistent — the resulting backup may be completely unrestorable.

### Option 1: Pause the Container

```bash
# Pause the database (freezes all writes)
docker pause db

# Run the backup
docker run --rm \
  -v pgdata:/source:ro \
  -v /backups:/backup \
  alpine \
  tar czf /backup/pgdata-$(date +%Y%m%d-%H%M%S).tar.gz -C /source .

# Resume the database
docker unpause db
```

Pausing is fast (milliseconds) and keeps the database available — it just freezes I/O briefly during the backup.

### Option 2: Use Database-Native Backup Tools (Recommended)

For production databases, always use the database's own backup mechanism rather than backing up the raw data directory. Database-native backups are consistent by design:

**PostgreSQL:**

```bash
# pg_dump creates a consistent, portable backup without stopping the server
docker exec db pg_dump \
  -U apiuser \
  -d myapp \
  -F custom \
  -f /tmp/myapp-$(date +%Y%m%d-%H%M%S).dump

# Copy the dump out of the container
docker cp db:/tmp/myapp-20240115-092341.dump ./backups/
```

**MySQL/MariaDB:**

```bash
# mysqldump creates a consistent backup with table locks
docker exec db mysqldump \
  -u root \
  -psecretpassword \
  --all-databases \
  --single-transaction \
  > ./backups/mysql-$(date +%Y%m%d-%H%M%S).sql
```

**Redis:**

```bash
# Trigger a BGSAVE to create an RDB snapshot
docker exec redis redis-cli BGSAVE

# Wait for completion
docker exec redis redis-cli INFO persistence | grep rdb_bgsave_in_progress

# Copy the RDB file out
docker cp redis:/data/dump.rdb ./backups/redis-$(date +%Y%m%d-%H%M%S).rdb
```

---

## Backup to AWS S3

For production systems, backups should go to AWS S3 — not to the same host disk. If the host disk fails, you lose both the volume and the backup.

### Single-Step Backup: Volume → S3

```bash
# Backup directly to S3 using the AWS CLI image
docker run --rm \
  -v pgdata:/source:ro \
  -e AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID \
  -e AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY \
  -e AWS_DEFAULT_REGION=us-east-1 \
  amazon/aws-cli \
  s3 cp - s3://my-backups-bucket/pgdata/pgdata-$(date +%Y%m%d-%H%M%S).tar.gz \
  --stdin \
  --content-type application/gzip < <(docker run --rm -v pgdata:/source:ro alpine tar czf - -C /source .)
```

A cleaner approach — pipe the tar output directly to S3:

```bash
# Step 1: Create the archive
docker run --rm -v pgdata:/source:ro alpine tar czf - -C /source . | \
# Step 2: Pipe to S3
aws s3 cp - s3://my-backups-bucket/pgdata/pgdata-$(date +%Y%m%d-%H%M%S).tar.gz
```

### Automated Backup Script

```bash
#!/bin/bash
# /usr/local/bin/backup-volumes.sh

set -euo pipefail

BUCKET="my-backups-bucket"
DATE=$(date +%Y%m%d-%H%M%S)

volumes=(
  "pgdata"
  "redis-data"
  "api-uploads"
)

for volume in "${volumes[@]}"; do
  echo "Backing up volume: $volume"
  
  # Archive and stream to S3
  docker run --rm -v ${volume}:/source:ro alpine tar czf - -C /source . | \
    aws s3 cp - s3://${BUCKET}/${volume}/${volume}-${DATE}.tar.gz
  
  echo "Backed up ${volume} to s3://${BUCKET}/${volume}/${volume}-${DATE}.tar.gz"
done

# Apply a lifecycle policy to S3 to auto-delete backups older than 30 days
# (configure this in the AWS console or via aws s3api put-bucket-lifecycle-configuration)

echo "All backups complete."
```

Schedule this script with cron:

```bash
# Run every day at 2am UTC
0 2 * * * /usr/local/bin/backup-volumes.sh >> /var/log/docker-backups.log 2>&1
```

---

## Restoring from S3

```bash
# Download from S3 and restore to the volume in one pipeline
aws s3 cp s3://my-backups-bucket/pgdata/pgdata-20240115-092341.tar.gz - | \
  docker run --rm -i -v pgdata:/target alpine tar xzf - -C /target
```

---

## Migrating a Volume Between Hosts

To move a volume from one Docker host to another:

**On the source host:**

```bash
# Create the archive
docker run --rm -v pgdata:/source:ro alpine tar czf - -C /source . \
  > pgdata-export.tar.gz

# Transfer to destination host
scp pgdata-export.tar.gz user@destination-host:/tmp/
```

**On the destination host:**

```bash
# Create the volume on the new host
docker volume create pgdata

# Restore the data
cat /tmp/pgdata-export.tar.gz | \
  docker run --rm -i -v pgdata:/target alpine tar xzf - -C /target
```

The volume is now available on the new host with all its data intact.

---

## Verifying a Backup

After creating a backup, always verify it is restorable. A corrupt backup is worse than no backup — it gives false confidence.

```bash
# Create a temporary verification volume
docker volume create verify-pgdata

# Restore the backup into the verification volume
cat pgdata-backup-20240115.tar.gz | \
  docker run --rm -i -v verify-pgdata:/target alpine tar xzf - -C /target

# Verify the contents look correct
docker run --rm -v verify-pgdata:/data alpine ls -la /data/

# For PostgreSQL: check if the data directory is valid
docker run --rm \
  -v verify-pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=test \
  postgres:16 \
  postgres --check   # exits 0 if data directory is valid

# Clean up
docker volume rm verify-pgdata
```

---

## Summary

- The core backup technique is a temporary `alpine` container that mounts the volume read-only and runs `tar czf` to create a compressed archive
- Always stop or pause a database container before running a tar backup of its raw data directory — live database files are internally inconsistent during writes
- For production databases, use database-native backup tools (pg_dump, mysqldump, redis BGSAVE) which produce consistent backups without stopping the server
- In production, stream backups directly to AWS S3 rather than local disk — if the host fails, local backups fail with it
- Schedule backups with cron and monitor backup log output — an unmonitored backup job that silently fails is as dangerous as no backup at all
- Always verify backups by restoring to a temporary volume and checking that the data is valid and complete
- Volume migration between hosts is a three-step process: tar the source, transfer the archive, restore on the destination

In the next file, we cover volume drivers — how to use third-party drivers to store volume data in cloud storage, NFS servers, and distributed filesystems.
