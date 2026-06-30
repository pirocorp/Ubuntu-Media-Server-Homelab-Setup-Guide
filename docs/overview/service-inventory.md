# Service Inventory

Status: Implemented
Purpose: Map each deployed or planned component to its documentation home.
Depends on: [Current state](./current-state.md)
Related docs: [Services](../services/README.md), [Platform](../platform/README.md), [Roadmaps](../roadmaps/README.md)

## Implemented Platform Components

| Component | Role | Documentation |
| --- | --- | --- |
| Ubuntu Server | Host operating system | [Base server setup](../platform/base-server-setup.md) |
| Cockpit | Server administration | [Base server setup](../platform/base-server-setup.md) |
| Docker and Compose | Container runtime | [Docker and Portainer](../platform/docker-and-portainer.md) |
| Portainer | Container management | [Docker and Portainer](../platform/docker-and-portainer.md) |
| AdGuard Home | DNS and filtering | [Networking and reverse proxy](../platform/networking-and-reverse-proxy.md) |
| Nginx Proxy Manager | Reverse proxy and HTTPS ingress | [Networking and reverse proxy](../platform/networking-and-reverse-proxy.md) |
| Storage mounts and Samba | Shared storage and LAN file access | [Storage and Samba](../platform/storage-and-samba.md) |

## Implemented Services And Workloads

| Service | Role | Documentation |
| --- | --- | --- |
| Plex | Media server | [Plex](../services/plex/README.md) |
| Nextcloud | Files, calendar, contacts, and sync | [Nextcloud](../services/nextcloud/README.md) |
| qBittorrent | Torrent downloader and seedbox | [qBittorrent](../services/qbittorrent/README.md) |
| Bitmagnet | Torrent search engine | [Bitmagnet](../services/bitmagnet/README.md) |
| UPS monitoring | Power visibility and monitoring | [UPS monitoring](../services/ups-monitoring/README.md) |
| ShadowBroker | Intelligence and monitoring platform | [ShadowBroker](../services/shadowbroker/README.md) |

## Planned

| Service | Status | Documentation |
| --- | --- | --- |
| Usenet stack | Planned only, not deployed | [Usenet roadmap](../roadmaps/usenet/README.md) |
| ShadowBroker OpenClaw integration | Planned only, not deployed | [Integration roadmap](../roadmaps/shadowbroker-openclaw-integration.md) |
