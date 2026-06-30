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
| ShadowBroker | `https://shadowbroker.pirocorp.com` |

## Direct Admin Ports

| Service | Address |
| --- | --- |
| AdGuard Home setup | `192.168.0.10:3000` |
| Cockpit | `192.168.0.10:9090` |
| Portainer | `192.168.0.10:9443` |
| Netdata | `192.168.0.10:19999` |
| Plex | `192.168.0.10:32400` |

## Scope

### Implemented

- Base Ubuntu Server host and SSH administration
- Docker and Docker Compose workloads
- Cockpit server administration
- Portainer container management
- AdGuard Home local DNS and filtering
- Nginx Proxy Manager internal ingress
- Storage mounts and Samba shares
- Plex media server
- Nextcloud
- qBittorrent
- Bitmagnet
- UPS monitoring with NUT and Netdata
- ShadowBroker

### Planned

- [Usenet stack and architecture roadmap](../roadmaps/usenet/README.md)
- [ShadowBroker OpenClaw integration roadmap](../roadmaps/shadowbroker-openclaw-integration.md)
