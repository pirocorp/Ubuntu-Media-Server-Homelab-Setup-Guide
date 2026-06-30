# Kavita

Status: Implemented
Purpose: Deployment reference for the Kavita workload.
Depends on: [Docker and Portainer](../../platform/docker-and-portainer.md), [Storage and Samba](../../platform/storage-and-samba.md)
Related docs: [Services index](../README.md), [Current state](../../overview/current-state.md)

## Overview

Kavita is a straightforward single-container reading server used for the books and comics libraries stored on shared storage.

## Deployment Summary

| Item | Value |
| --- | --- |
| Stack directory | `/srv/docker/kavita` |
| Container name | `kavita` |
| Application version | `0.9.0.2` |
| Image | `jvmilazzo/kavita:0.9.0.2` |
| Public web UI | `https://kavita.pirocorp.com` |
| Direct web UI | `http://192.168.0.10:5000` |
| Port mapping | `5000 -> 5000` |
| Time zone | `Europe/Brussels` |
| Restart policy | `unless-stopped` |
| Reverse proxy | Nginx Proxy Manager |

## Storage Layout

| Host path | Container path | Purpose |
| --- | --- | --- |
| `./config` | `/kavita/config` | Kavita configuration and app state |
| `/mnt/comp/comics` | `/comics` | Only comics library path |
| `/mnt/comp/books` | `/books` | Only books library path |

## Runtime Configuration

Verified `.env` values:

```env
KAVITA_VERSION=0.9.0.2
TZ=Europe/Brussels
```

Compose characteristics:

- single service: `kavita`
- explicit container name: `kavita`
- one published HTTP port
- no sidecar containers in the documented stack
- published externally through Nginx Proxy Manager

## Operations

Standard workflow:

```bash
cd /srv/docker/kavita
docker compose config
docker compose up -d
```

Update workflow:

```bash
cd /srv/docker/kavita
docker compose pull
docker compose up -d
```

Stop the stack:

```bash
cd /srv/docker/kavita
docker compose down
```

## Notes

- This is a simple workload with configuration stored in the stack directory and media mounted from `/mnt/comp`.
- `/mnt/comp/comics` and `/mnt/comp/books` are the only documented library paths for this service.
- There are no documented update caveats beyond the standard compose pull and recreate workflow.
