# Audiobookshelf

Status: Implemented
Purpose: Deployment reference for the Audiobookshelf workload.
Depends on: [Docker and Portainer](../../platform/docker-and-portainer.md), [Storage and Samba](../../platform/storage-and-samba.md)
Related docs: [Services index](../README.md), [Current state](../../overview/current-state.md)

## Overview

Audiobookshelf is a simple single-container workload used to serve the audiobook library from shared storage.

## Deployment Summary

| Item | Value |
| --- | --- |
| Stack directory | `/srv/docker/audiobookshelf` |
| Container name | `audiobookshelf` |
| Image | `ghcr.io/advplyr/audiobookshelf:2.35.1` |
| Public web UI | `https://audiobookshelf.pirocorp.com` |
| Direct web UI | `http://192.168.0.10:13378` |
| Port mapping | `13378 -> 80` |
| Time zone | `Europe/Brussels` |
| Restart policy | `unless-stopped` |
| Reverse proxy | Nginx Proxy Manager |

## Storage Layout

| Host path | Container path | Purpose |
| --- | --- | --- |
| `./config` | `/config` | Audiobookshelf configuration |
| `./metadata` | `/metadata` | App metadata and state |
| `/mnt/comp/audiobooks` | `/audiobooks` | Only audiobook library path |

## Runtime Configuration

Environment values currently documented for the stack:

```env
AUDIOBOOKSHELF_VERSION=2.35.1
TZ=Europe/Brussels
```

Compose characteristics:

- single service: `audiobookshelf`
- explicit container name: `audiobookshelf`
- one published HTTP port
- no sidecar containers in the documented stack
- published externally through Nginx Proxy Manager

## Operations

Standard workflow:

```bash
cd /srv/docker/audiobookshelf
docker compose config
docker compose up -d
```

Update workflow:

```bash
cd /srv/docker/audiobookshelf
docker compose pull
docker compose up -d
```

Stop the stack:

```bash
cd /srv/docker/audiobookshelf
docker compose down
```

## Notes

- This is a straightforward workload with persistent app state kept inside the stack directory and media stored on `/mnt/comp`.
- `/mnt/comp/audiobooks` is the only documented library path for this service.
- There are no special update caveats beyond the standard compose pull and recreate workflow.
