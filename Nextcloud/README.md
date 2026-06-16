# Nextcloud Homelab Deployment

Self-hosted Nextcloud instance running in Docker.

Purpose:

- Calendar replacement
- CalDAV/CardDAV server
- Mail integration
- File storage
- Contact synchronization

Designed for a local homelab environment using:

- Docker Compose
- PostgreSQL
- Redis
- Nginx Proxy Manager
- AdGuard local DNS


---

# Architecture

## Request flow

```text
Client
  |
  | HTTPS
  |
  v
nextcloud.home
  |
  |
AdGuard DNS
  |
  |
Ubuntu Docker Host
  |
  |
Nginx Proxy Manager :443
  |
  | HTTP
  |
Nextcloud Container :80
```

HTTPS termination happens at Nginx Proxy Manager.

Nextcloud receives HTTP traffic internally.

---

# Directory Structure

```text
/srv/docker/nextcloud/

├── README.md
├── compose.yml
├── .env
│
├── nextcloud/
│   ├── config/
│   ├── data/
│   └── apps/
│
├── postgres/
│   └── database files
│
└── redis/
    └── cache data
```

---

# Containers

## nextcloud-app

Main application container.

Image:

```text
nextcloud:32-apache
```

Internal port:

```text
80/tcp
```

Host mapping:

```text
8090 -> 80
```

URL:

```text
https://nextcloud.home
```


---

## nextcloud-db

PostgreSQL database.

Image:

```text
postgres:16-alpine
```

Storage:

```text
./postgres
```


---

## nextcloud-redis

Redis memory cache.

Image:

```text
redis:8-alpine
```

Used for:

- file locking
- cache
- performance improvement


---

## nextcloud-cron

Runs Nextcloud background jobs.

Uses:

```text
/cron.sh
```

Do not use AJAX cron.


---

# Environment Configuration

File:

```bash
.env
```

Important values:

```env
NEXTCLOUD_VERSION=32-apache

NEXTCLOUD_DOMAIN=nextcloud.home

NEXTCLOUD_HTTP_PORT=8090

OVERWRITEPROTOCOL=https

TRUSTED_PROXIES=172.16.0.0/12
```

---

# Reverse Proxy Configuration

Nginx Proxy Manager:

## Details

Domain:

```text
nextcloud.home
```

Scheme:

```text
http
```

Forward Host:

```text
Docker host IP
```

Forward Port:

```text
8090
```

Enable:

- Websocket Support
- Block Common Exploits
- Force SSL
- HTTP/2

---

# DNS

AdGuard DNS rewrite:

```text
nextcloud.home
        |
        v
Ubuntu Server IP
```

---

# Operations Runbook


# Start

```bash
cd /srv/docker/nextcloud

docker compose up -d
```


---

# Stop

```bash
docker compose down
```


---

# Restart

```bash
docker compose restart
```


---

# View Containers

```bash
docker ps
```


---

# View Logs

All:

```bash
docker compose logs -f
```

Nextcloud only:

```bash
docker logs -f nextcloud-app
```

Database:

```bash
docker logs -f nextcloud-db
```


---

# Enter Nextcloud Container

```bash
docker exec -it nextcloud-app bash
```


---

# Run OCC Commands

Always execute OCC as www-data.

Example:

```bash
docker exec \
-u www-data \
nextcloud-app \
php occ status
```


---

# Maintenance Mode

Enable:

```bash
docker exec \
-u www-data \
nextcloud-app \
php occ maintenance:mode --on
```

Disable:

```bash
docker exec \
-u www-data \
nextcloud-app \
php occ maintenance:mode --off
```


---

# Update Containers

Minor update inside pinned major version.

Example:

32.x -> newer 32.x

```bash
cd /srv/docker/nextcloud

docker compose pull

docker compose up -d
```

Run upgrade:

```bash
docker exec \
-u www-data \
nextcloud-app \
php occ upgrade
```


---

# Major Version Upgrade

Example:

32 -> 33

Edit:

```bash
nano .env
```

Change:

```env
NEXTCLOUD_VERSION=33-apache
```

Upgrade:

```bash
docker compose pull

docker compose up -d

docker exec \
-u www-data \
nextcloud-app \
php occ upgrade
```

Check:

```bash
docker exec \
-u www-data \
nextcloud-app \
php occ status
```


---

# Backup Procedure

Stop containers:

```bash
docker compose down
```

Backup folder:

```bash
tar czvf nextcloud-backup.tar.gz \
/srv/docker/nextcloud
```

Start:

```bash
docker compose up -d
```


---

# Restore Procedure

Stop:

```bash
docker compose down
```

Restore files:

```bash
tar xzvf nextcloud-backup.tar.gz -C /
```

Start:

```bash
cd /srv/docker/nextcloud

docker compose up -d
```


---

# Health Checks


## Nextcloud status

```bash
docker exec \
-u www-data \
nextcloud-app \
php occ status
```


Expected:

```text
installed: true
maintenance: false
version: 32.x
```


---

## Database check

```bash
docker exec \
nextcloud-db \
pg_isready
```


---

## Redis check

```bash
docker exec \
nextcloud-redis \
redis-cli ping
```

Expected:

```text
PONG
```


---

# Clients


## CalDAV

URL:

```text
https://nextcloud.home/remote.php/dav
```


Supported:

- iOS Calendar
- Thunderbird
- Android DAVx5
- Outlook CalDav Synchronizer


---

# Notes

- Docker images are pinned by major version
- Upgrades are manual
- HTTPS handled by reverse proxy
- All application data stored under `/srv/docker/nextcloud`
- Suitable for LAN/self-hosted usage
