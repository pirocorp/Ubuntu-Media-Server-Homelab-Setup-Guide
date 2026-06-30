# Current State

Status: Implemented
Purpose: Record the live homelab baseline, key endpoints, and current implementation scope.
Depends on: [Platform docs](../platform/README.md)
Related docs: [Architecture](./architecture.md), [Service inventory](./service-inventory.md), [Services](../services/README.md)

## Host Baseline

| Property | Value |
| --- | --- |
| Hostname | `piroman-server` |
| LAN IP | `192.168.0.10` |
| OS | `Ubuntu 26.04 LTS` |
| Kernel | `Linux 7.0.0-22-generic` |
| Architecture | `x86-64` |
| Main admin user | `piroman` |
| Primary service domain | `*.pirocorp.com` |
| Container root | `/srv/docker` |
| Main shared data mount | `/mnt/data` |

## Current Access URLs

| Service | URL |
| --- | --- |
| Server monitoring (Netdata) | `https://server.pirocorp.com` |
| Portainer | `https://portainer.pirocorp.com` |
| AdGuard Home | `https://adguard.pirocorp.com` |
| Nginx Proxy Manager admin | `http://npm.pirocorp.com:81` |
| Nextcloud | `https://nextcloud.pirocorp.com` |
| Bitmagnet | `https://bitmagnet.pirocorp.com` |
| qBittorrent | `https://qbittorrent.pirocorp.com` |
| Plex | `https://plex.pirocorp.com` |
| ShadowBroker | `https://shadowbroker.pirocorp.com` |

## Direct Ports

| Service | Address |
| --- | --- |
| AdGuard Home setup | `192.168.0.10:3000` |
| Nginx Proxy Manager admin | `192.168.0.10:81` |
| Portainer | `192.168.0.10:9443` |
| Netdata | `192.168.0.10:19999` |
| Plex | `192.168.0.10:32400` |
| qBittorrent web UI | `192.168.0.10:8080` |
| Bitmagnet web/API | `192.168.0.10:3333-3334` |
| Nextcloud app | `192.168.0.10:8090` |
| ShadowBroker frontend | `192.168.0.10:3010` |
| ShadowBroker backend | `192.168.0.10:8010` |
| Audiobookshelf | `192.168.0.10:13378` |
| Immich | `192.168.0.10:2283` |
| Kavita | `192.168.0.10:5000` |

## Active Docker Stack Roots

- `/srv/docker/adguard-home`
- `/srv/docker/audiobookshelf`
- `/srv/docker/bitmagnet`
- `/srv/docker/immich`
- `/srv/docker/kavita`
- `/srv/docker/nextcloud`
- `/srv/docker/nginx-proxy-manager`
- `/srv/docker/plex`
- `/srv/docker/portainer`
- `/srv/docker/qbittorrent`
- `/srv/docker/shadowbroker`

## Mounted Storage Snapshot

| Mount | Label | Filesystem | Size | Used | Available | Use |
| --- | --- | --- | --- | --- | --- | --- |
| `/` | system | `ext4` | `1.8T` | `125G` | `1.6T` | `8%` |
| `/mnt/iac` | `IAC` | `ntfs` | `3.7T` | `1.8T` | `1.9T` | `50%` |
| `/mnt/lp` | `LP` | `ntfs` | `7.3T` | `3.7T` | `3.7T` | `51%` |
| `/mnt/data` | `DATA` | `ntfs` | `7.3T` | `3.3T` | `4.1T` | `45%` |
| `/mnt/ia` | `IA` | `ntfs` | `3.7T` | `2.8T` | `894G` | `77%` |
| `/mnt/comp` | `COMP` | `ntfs` | `3.7T` | `1.1T` | `2.7T` | `28%` |

## Scope

### Implemented

- Base Ubuntu Server host and SSH administration
- Docker and Docker Compose workloads
- Portainer container management
- AdGuard Home local DNS and filtering
- Nginx Proxy Manager internal ingress
- Storage mounts and Samba shares
- Plex media server
- Nextcloud
- qBittorrent
- Bitmagnet
- Audiobookshelf
- Immich
- Kavita
- UPS monitoring with NUT and Netdata
- ShadowBroker

### Planned

- [Usenet stack and architecture roadmap](../roadmaps/usenet/README.md)
- [ShadowBroker OpenClaw integration roadmap](../roadmaps/shadowbroker-openclaw-integration.md)
