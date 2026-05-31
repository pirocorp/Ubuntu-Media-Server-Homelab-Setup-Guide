# Bitmagnet Runbook

## Overview

Bitmagnet is a self-hosted BitTorrent DHT search engine running in Docker.

### Architecture

```text
AdGuard Home
      ↓
bitmagnet.home
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
https://bitmagnet.home
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

## Netdata Monitoring

Bitmagnet is primarily:

* CPU intensive during crawling
* Disk intensive during indexing
* PostgreSQL intensive as the database grows

### Recommended Netdata Checks

#### CPU Usage

Navigate to:

```text
Netdata
→ System Overview
→ CPU
```

Watch for:

* Sustained high CPU usage
* CPU spikes during crawling
* CPU saturation

Expected on Xeon E3-1231 v3:

```text
Occasional spikes are normal.
Sustained 100% usage is not.
```

---

#### Memory Usage

Navigate to:

```text
Netdata
→ System Overview
→ Memory
```

Watch for:

* RAM consumption
* Swap usage

Current baseline:

```text
30 GB RAM
8 GB Swap
```

Bitmagnet should not require large amounts of RAM for a personal homelab deployment.

---

#### Disk Usage

Navigate to:

```text
Netdata
→ Disks
→ Space Usage
```

Monitor:

```text
/
```

Expected:

```text
1.8 TB NVMe SSD
```

Recommended alert threshold:

```text
80% utilization
```

---

#### Disk I/O

Navigate to:

```text
Netdata
→ Disks
→ I/O
```

Watch for:

* High write activity
* PostgreSQL indexing activity
* SSD bottlenecks

---

#### Docker Containers

Navigate to:

```text
Netdata
→ Containers
```

Monitor:

```text
bitmagnet
bitmagnet-postgres
```

Key metrics:

* CPU
* Memory
* Network
* Disk I/O

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


# qBittorrent ↔ Plex Integration

## Overview

This setup downloads media directly into Plex library folders and automatically refreshes Plex when a torrent finishes downloading.

### Workflow

```text
Torrent added
      ↓
Downloaded directly to Plex library folder
      ↓
Torrent reaches 100%
      ↓
qBittorrent executes completion hook
      ↓
Plex library refresh is triggered
      ↓
New media appears automatically in Plex
      ↓
Torrent continues seeding
```

---

## Container Information

### Container Name

```bash
qbittorrent
```

### Docker Directory

```bash
/srv/docker/qbittorrent
```

### qBittorrent Version

```text
lscr.io/linuxserver/qbittorrent:5.2.1
```

### User Mapping

```text
PUID=1000
PGID=1000
```

Container files are owned by the `piroman` user.

---

## Plex Library Mount

### Host Path

```bash
/mnt/data/Plex
```

### Container Path

```bash
/plex
```

Current structure:

```text
/plex
├── Movies
├── Series
└── Transcode
```

---

## qBittorrent Categories

### Movies

```text
Category: plex-movies
Save Path: /plex/Movies
```

### Series

```text
Category: plex-series
Save Path: /plex/Series
```

---

## Plex Refresh Script

### Host Location

```bash
/srv/docker/qbittorrent/scripts/plex-refresh.sh
```

### Container Location

```bash
/scripts/plex-refresh.sh
```

### Script Contents

```bash
#!/bin/sh

echo "$(date) - Torrent completed, refreshing Plex" >> /config/plex-refresh.log

curl -s -X POST \
"http://192.168.0.10:32400/library/sections/all/refresh?X-Plex-Token=YOUR_PLEX_TOKEN" \
>/dev/null 2>&1
```

### Make Script Executable

```bash
chmod +x /srv/docker/qbittorrent/scripts/plex-refresh.sh
```

---

## Docker Volume Mapping

```yaml
volumes:
  - /srv/docker/qbittorrent/config:/config
  - /srv/docker/qbittorrent/scripts:/scripts:ro
  - /mnt/data/Plex:/plex
```

---

## qBittorrent Completion Hook

Navigate to:

```text
Options
→ Downloads
→ Run external program
```

Enable:

```text
Run on torrent finished
```

Command:

```bash
/scripts/plex-refresh.sh
```

---

## Logging

### Log File

```bash
/config/plex-refresh.log
```

### View Log

```bash
docker exec -it qbittorrent cat /config/plex-refresh.log
```

Example:

```text
Sun May 31 14:08:02 CEST 2026 - Torrent completed, refreshing Plex
Sun May 31 14:35:47 CEST 2026 - Torrent completed, refreshing Plex
```

---

## Testing

### Execute Script Manually

```bash
docker exec -it qbittorrent /scripts/plex-refresh.sh
```

### Verify Log Entry

```bash
docker exec -it qbittorrent cat /config/plex-refresh.log
```

### Verify Plex Refresh

Navigate to:

```text
Plex
→ Settings
→ Status
→ Alerts
```

Expected messages:

```text
Scanning the "Филми" section
Scanning the "ТВ сериали" section
Library scan complete
```

---

## Torrent Port Configuration

### qBittorrent Listening Port

```text
6881
```

### Docker Ports

```yaml
ports:
  - "8080:8080"
  - "6881:6881/tcp"
  - "6881:6881/udp"
```

### Router Port Forwarding

```text
External Port: 6881
Internal IP: 192.168.0.10
Internal Port: 6881
Protocol: TCP + UDP
```

### External Verification

```bash
curl -4 ifconfig.me
```

Port validation:

```text
https://canyouseeme.org/
```

Expected result:

```text
Success: I can see your service on <public-ip> on port 6881
```

---

## Useful Commands

### Start qBittorrent

```bash
cd /srv/docker/qbittorrent
docker compose up -d
```

### Stop qBittorrent

```bash
cd /srv/docker/qbittorrent
docker compose down
```

### View Logs

```bash
docker compose logs -f qbittorrent
```

### Enter Container

```bash
docker exec -it qbittorrent sh
```

### Verify Script Exists

```bash
docker exec -it qbittorrent ls -l /scripts
```

### Verify Plex Refresh Log

```bash
docker exec -it qbittorrent cat /config/plex-refresh.log
```

---

## Notes

* Downloads are stored directly in Plex library folders.
* No Radarr or Sonarr are required.
* Completed torrents remain available for seeding.
* Plex refreshes automatically when a torrent completes.
* Library refresh is triggered using the Plex HTTP API.
* Port 6881 is externally reachable and forwarded through the router.











