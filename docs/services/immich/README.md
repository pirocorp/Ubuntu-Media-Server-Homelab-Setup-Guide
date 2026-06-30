# Immich

Status: Implemented
Purpose: Deployment and workflow reference for the Immich stack.
Depends on: [Docker and Portainer](../../platform/docker-and-portainer.md), [Storage and Samba](../../platform/storage-and-samba.md), [Networking and reverse proxy](../../platform/networking-and-reverse-proxy.md)
Related docs: [Update and backup runbook](./update-runbook.md), [Services index](../README.md), [Current state](../../overview/current-state.md)

## Overview

Immich is the homelab photo and video platform. It is treated as the indexing, search, AI, and organization layer first, with filesystem-level organization happening later through exports and optional external libraries.

## Deployment Summary

| Item | Value |
| --- | --- |
| Stack directory | `/srv/docker/immich` |
| Main containers | `immich_server`, `immich_machine_learning`, `immich_postgres`, `immich_redis` |
| Application version | `v2.7.5` |
| App image | `ghcr.io/immich-app/immich-server:v2.7.5` |
| Public web UI | `https://immich.pirocorp.com` |
| Direct app URL | `http://192.168.0.10:2283` |
| Port mapping | `2283 -> 2283` |
| Reverse proxy | Nginx Proxy Manager |
| DNS | AdGuard Home |
| Restart policy | `unless-stopped` |

## Stack Architecture

The deployment follows the standard Immich multi-container layout:

- `immich_server`
- `immich_machine_learning`
- `immich_redis`
- `immich_postgres`

Images are version-pinned rather than using floating `latest` tags.

Current image definitions:

- Immich app: `ghcr.io/immich-app/immich-server:${IMMICH_VERSION}`
- Machine learning: `ghcr.io/immich-app/immich-machine-learning:${IMMICH_VERSION}`
- Valkey: `docker.io/valkey/valkey:9@sha256:3b55fbaa0cd93cf0d9d961f405e4dfcc70efe325e2d84da207a0a8e6d8fde4f9`
- PostgreSQL: `ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0@sha256:bcf63357191b76a916ae5eb93464d65c07511da41e3bf7a8416db519b40b1c23`

## Runtime Configuration

Verified `.env` values:

```env
UPLOAD_LOCATION=/mnt/data/Immich/Library
DB_DATA_LOCATION=/srv/docker/immich/postgres

IMMICH_VERSION=v2.7.5

DB_USERNAME=postgres
DB_DATABASE_NAME=immich
```

`DB_PASSWORD` is configured in `.env` but is intentionally not documented here.

Application container environment:

```env
DB_HOSTNAME=immich_postgres
DB_USERNAME=${DB_USERNAME}
DB_PASSWORD=${DB_PASSWORD}
DB_DATABASE_NAME=${DB_DATABASE_NAME}
REDIS_HOSTNAME=immich_redis
```

## Storage Layout

Root storage:

```text
/mnt/data/Immich
```

Structure:

```text
Immich/
|
+-- Library/
|
+-- Backups/
|
+-- PhotoAlbums/
```

Additional persistent paths used by the stack:

| Host path | Container path | Purpose |
| --- | --- | --- |
| `/mnt/data/Immich/Library` | `/data` | Immich-managed library |
| `/srv/docker/immich/postgres` | `/var/lib/postgresql/data` | PostgreSQL data |
| `/srv/docker/immich/model-cache` | `/cache` | Machine learning model cache |
| `/etc/localtime` | `/etc/localtime:ro` | Host time synchronization |

### Library

Immich-managed storage mounted into the application as:

```text
/data
```

Immich creates and manages directories such as:

- `upload/`
- `library/`
- `thumbs/`
- `encoded-video/`
- `profile/`
- `backups/`

These managed folders should not be manually reorganized.

### PhotoAlbums

`PhotoAlbums` is the planned export area for album-based filesystem organization.

The intended model is:

- create and curate albums inside Immich first
- export finished albums into `PhotoAlbums/<Album Name>/`
- optionally add `PhotoAlbums` back into Immich later as an External Library

### Backups

`Backups` is also still in the planning phase as the intended location for manual exports and backup archives.

## Reverse Proxy

Immich is published through Nginx Proxy Manager and AdGuard Home.

Current routing:

```text
immich.pirocorp.com -> 192.168.0.10
```

Proxy target:

```text
http://192.168.0.10:2283
```

Enabled options:

- SSL enabled
- Force HTTPS
- HTTP/2

Historical note:

- earlier implementation notes referenced `immich.home`
- the current repo tracks the active homelab naming scheme as `immich.pirocorp.com`

### Custom Nginx Configuration

```nginx
client_max_body_size 0;

proxy_request_buffering off;
proxy_buffering off;

proxy_read_timeout 3600s;
proxy_send_timeout 3600s;
send_timeout 3600s;
```

This was added because large video uploads over the reverse proxy were failing.

## Known Issue History

### Large Upload Failures

Large video uploads, especially files over 2 GB, previously failed because Nginx timed out before the upload completed.

The fix was to increase the relevant timeout values to `3600` seconds and disable request buffering.

### Immich Marker File Crash

Immich later started restarting because expected marker files were missing inside the managed library.

The missing path reported in logs was:

```text
/data/encoded-video/.immich
```

A validation command used during troubleshooting:

```bash
find /mnt/data/Immich/Library -maxdepth 2 -name .immich
```

When that returned nothing, it confirmed the hidden `.immich` marker files were gone. After restoring them, the stack started successfully again.

## User Workflow

The intended operating model is:

### Phase 1

Import every photo and video into Immich.

Immich is responsible for:

- duplicate detection
- metadata handling
- AI indexing
- search
- albums
- people recognition
- maps

### Phase 2

Organize content into logical albums inside Immich.

Examples:

- `Vacation 2026`
- `Family`
- `Germany`
- `Drone`
- `Kids`

### Phase 3

Export albums into filesystem folders under `PhotoAlbums/`.

### Phase 4

Optionally re-add `PhotoAlbums` to Immich as an External Library so Immich indexes the files without owning or moving them.

## External Library Model

Immich Managed Library:

- handles uploads
- generates thumbnails
- performs transcoding
- runs AI features
- owns managed storage

External Library:

- indexes existing folders
- does not move files
- leaves folder management to the user

## TV Setup

A dedicated Immich user was created for TV usage:

- username/purpose: `TV`
- email: `tv@piroman-server`

The TV account is intended to have viewer-only access to specifically shared albums.

Preferred TV client:

- `Immich TV` non-official app

## Notes

- Reverse proxy access is considered mandatory for normal day-to-day use.
- HTTPS is used internally through Nginx Proxy Manager.
- Storage lives on `/mnt/data`, separate from the OS disk.
- This document combines the implementation summary with the verified `compose.yml`, `.env`, and host-state data.
