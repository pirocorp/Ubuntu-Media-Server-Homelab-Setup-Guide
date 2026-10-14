# Bitmagnet Runbook

Status: Implemented
Purpose: Operating guide for the Bitmagnet deployment.
Depends on: [Docker and Portainer](../../platform/docker-and-portainer.md), [Networking and reverse proxy](../../platform/networking-and-reverse-proxy.md)
Related docs: [Services index](../README.md), [Service inventory](../../overview/service-inventory.md)

## Overview

Bitmagnet is a self-hosted BitTorrent DHT search engine running in Docker.

### Architecture

```text
AdGuard Home
      ↓
bitmagnet.pirocorp.com
      ↓
Nginx Proxy Manager
      ↓
Bitmagnet Web UI (3333)
      ↓
Bitmagnet Services
├── http_server
├── queue_server
└── dht_crawler
      ↓
PostgreSQL
```

### Storage Layout

```text
/srv/docker/bitmagnet
├── compose.yml
├── .env
├── config
└── postgres
```

| Path                                | Purpose                       |
| ----------------------------------- | ----------------------------- |
| `/srv/docker/bitmagnet/config`      | Bitmagnet configuration files |
| `/srv/docker/bitmagnet/postgres`    | PostgreSQL database           |
| `/srv/docker/bitmagnet/compose.yml` | Docker Compose stack          |
| `/srv/docker/bitmagnet/.env`        | Environment variables         |

---

## Access

### Web UI

```text
https://bitmagnet.pirocorp.com
```

### Direct Access

```text
http://192.168.0.10:3333
```

---

## Docker Operations

### Navigate to Stack

```bash
cd /srv/docker/bitmagnet
```

### Validate Configuration

```bash
docker compose config
```

Validates:

* YAML syntax
* Environment variables
* Volume mappings
* Port mappings
* Docker Compose configuration

### Start Stack

```bash
docker compose up -d
```

### Stop Stack

```bash
docker compose down
```

### Restart Stack

```bash
docker compose down
docker compose up -d
```

### View Running Containers

```bash
docker compose ps
```

### View Logs

```bash
docker compose logs -f bitmagnet
```

### View PostgreSQL Logs

```bash
docker compose logs -f postgres
```

---

## Crawler Management

Bitmagnet worker configuration is controlled through:

```text
BITMAGNET_WORKERS
```

in the `.env` file.

### Current Setting

```env
BITMAGNET_WORKERS=http_server,queue_server,dht_crawler
```

---

### Full Crawling Mode

Continuously discovers new torrents.

```env
BITMAGNET_WORKERS=http_server,queue_server,dht_crawler
```

Apply changes:

```bash
docker compose up -d
```

Expected workers:

```text
http_server
queue_server
dht_crawler
```

---

### Search-Only Mode (Freeze Database Growth)

Stops discovering new torrents while preserving search functionality.

```env
BITMAGNET_WORKERS=http_server,queue_server
```

Apply changes:

```bash
docker compose up -d
```

Expected workers:

```text
http_server
queue_server
```

The existing database remains intact.

---

## Monitoring

### Container Status

```bash
docker compose ps
```

### Resource Usage

```bash
docker stats bitmagnet
```

### All Docker Resources

```bash
docker stats
```

### Monitor Bitmagnet Logs

```bash
docker compose logs -f bitmagnet
```

Look for:

```text
started worker {"key":"http_server"}
started worker {"key":"queue_server"}
started worker {"key":"dht_crawler"}
```

---

## Disk Usage Monitoring

### Analyze Entire Bitmagnet Stack

```bash
sudo ncdu -x /srv/docker/bitmagnet
```

### Analyze PostgreSQL Database Only

```bash
sudo ncdu -x /srv/docker/bitmagnet/postgres
```

### Quick Size Summary

```bash
du -sh /srv/docker/bitmagnet/*
```

Example:

```text
500M config
120G postgres
```

### Check Remaining SSD Space

```bash
df -h /
```

---

## Database Growth Monitoring

### PostgreSQL Directory Size

```bash
du -sh /srv/docker/bitmagnet/postgres
```

### Monthly Check

Record:

| Date       | Database Size | Notes              |
| ---------- | ------------- | ------------------ |
| YYYY-MM-DD | XX GB         | Initial deployment |

This provides visibility into crawler growth over time.

---

## Updating Bitmagnet

### Pull Latest Images

```bash
docker compose pull
```

### Recreate Containers

```bash
docker compose up -d
```

### Verify Running Version

```bash
docker compose ps
```

---

## Troubleshooting

### Validate Compose Configuration

```bash
docker compose config
```

### Check Logs

```bash
docker compose logs -f bitmagnet
```

### Verify Database Connectivity

```bash
docker compose logs postgres
```

### Verify Active Workers

```bash
docker compose logs bitmagnet | grep "started worker"
```

Expected output:

```text
started worker {"key":"http_server"}
started worker {"key":"queue_server"}
started worker {"key":"dht_crawler"}
```

---

## Backup

### Stop Stack

```bash
docker compose down
```

### Backup Database

```bash
tar -czf bitmagnet-postgres-backup.tar.gz postgres/
```

### Backup Entire Stack

```bash
tar -czf bitmagnet-backup.tar.gz \
compose.yml \
.env \
config \
postgres
```

---

## Useful Commands

### Docker Configuration Validation

```bash
docker compose config
```

### Show Final Environment Variable Resolution

```bash
docker compose config
```

Useful when troubleshooting `.env` changes.

### Display Open Ports

```bash
ss -tulpn | grep 333
```

Expected:

```text
3333/tcp
3334/tcp
3334/udp
```

### Display Stack Directory Size

```bash
du -sh /srv/docker/bitmagnet
```

### Display Largest Directories

```bash
sudo ncdu -x /srv/docker/bitmagnet
```

## PostgreSQL Administration

### Open PostgreSQL Shell

Connect to the PostgreSQL container:

```bash
docker exec -it bitmagnet-postgres psql -U bitmagnet -d bitmagnet
```

Expected prompt:

```text
bitmagnet=#
```

### List Tables

```sql
\dt
```

### Show Database Size

```sql
SELECT pg_size_pretty(pg_database_size('bitmagnet'));
```

Example:

```text
  pg_size_pretty
-----------------
 42 GB
```

### Show Largest Tables

```sql
SELECT
    relname AS table_name,
    pg_size_pretty(pg_total_relation_size(relid)) AS size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;
```

Useful when identifying which tables are responsible for database growth.

### Exit PostgreSQL

```sql
\q
```

---

## PostgreSQL Monitoring

### Database Directory Size

Filesystem perspective:

```bash
du -sh /srv/docker/bitmagnet/postgres
```

### Database Size (PostgreSQL)

Database perspective:

```bash
docker exec -it bitmagnet-postgres \
psql -U bitmagnet -d bitmagnet \
-c "SELECT pg_size_pretty(pg_database_size('bitmagnet'));"
```

### Container Resource Usage

```bash
docker stats bitmagnet-postgres
```

---

## Current Host and Container Monitoring (Homepage + Glances)

Netdata was retired on 2026-10-14, after the [Homepage / Glances / PeaNUT Server Health Dashboard](../server-health/README.md) was validated. Do not follow the former Netdata navigation instructions.

Bitmagnet may use substantial CPU while crawling, storage I/O while indexing and PostgreSQL disk space as its database grows. The current compact dashboard covers host CPU/RAM, root filesystem capacity, and existing Docker container status with on-demand CPU/RAM/network figures. **It is not a replacement for historical PostgreSQL or per-process I/O tracing.**

Check present conditions using:

```bash
cd /srv/docker/bitmagnet
docker compose ps
docker stats --no-stream bitmagnet bitmagnet-postgres
df -h /
du -sh /srv/docker/bitmagnet/postgres
```

The system SSD is a Samsung SATA SSD; do not describe it as the former 1.8 TB NVMe device. For deeper storage I/O troubleshooting use existing Ubuntu tools on demand rather than reinstalling Netdata merely to display one extra dashboard card.

---

## Operational Health Checklist

Weekly:

```bash
docker compose ps
docker compose logs --tail=50 bitmagnet
du -sh /srv/docker/bitmagnet/postgres
df -h /
```

Monthly:

```bash
sudo ncdu -x /srv/docker/bitmagnet
docker stats bitmagnet
docker stats bitmagnet-postgres
```

Quarterly:

```bash
docker compose pull
docker compose config
```

Review:

* Database growth
* SSD capacity
* Container resource consumption
* Available Bitmagnet updates
* If the personal TMDB API key is configured correctly, Bitmagnet starts without the default TMDB API key warning.











