# Service Inventory

Status: Implemented
Purpose: Map each deployed or planned component to its documentation home.
Depends on: [Current state](./current-state.md)
Related docs: [Services](../services/README.md), [Platform](../platform/README.md), [Roadmaps](../roadmaps/README.md)

## Implemented Platform Components

| Component | Role | Documentation |
| --- | --- | --- |
| Ubuntu Server | Host operating system | [Base server setup](../platform/base-server-setup.md) |
| Docker and Compose | Container runtime | [Docker and Portainer](../platform/docker-and-portainer.md) |
| Portainer | Container management | [Docker and Portainer](../platform/docker-and-portainer.md) |
| AdGuard Home | DNS and filtering | [Networking and reverse proxy](../platform/networking-and-reverse-proxy.md) |
| Nginx Proxy Manager | Reverse proxy and HTTPS ingress | [Networking and reverse proxy](../platform/networking-and-reverse-proxy.md) |
| Tailscale remote access | VPN subnet routing for trusted remote clients | [Tailscale remote access runbook](../operations/tailscale-remote-access-runbook.md) |
| Storage mounts and Samba | Shared storage and LAN file access | [Storage and Samba](../platform/storage-and-samba.md) |

## Implemented Services And Workloads

| Service | Role | Documentation |
| --- | --- | --- |
| Homepage | Central homelab dashboard with Docker status and native service widgets | [Homepage](../services/homepage/README.md) |
| Plex | Media server | [Plex](../services/plex/README.md) |
| Nextcloud | Files, calendar, contacts, and sync | [Nextcloud](../services/nextcloud/README.md) |
| qBittorrent | Torrent downloader and seedbox | [qBittorrent](../services/qbittorrent/README.md) |
| Bitmagnet | Torrent search engine | [Bitmagnet](../services/bitmagnet/README.md) |
| Audiobookshelf | Audiobook and podcast server | [Audiobookshelf](../services/audiobookshelf/README.md) |
| Immich | Photo and video management | [Immich](../services/immich/README.md) |
| Kavita | Reading server and library manager | [Kavita](../services/kavita/README.md) |
| Server Health (Glances + PeaNUT) | Glances host metrics and PowerWalker UPS status in Homepage; six filesystem-capacity tiles now sourced from Disk Health v0.7 | [Reproducible deployment](../services/server-health/README.md) |
| SMART + Disk Space (host-native Python v0.7 + smartctl + statvfs) | Six-drive SMART diagnostics and six stable-ID filesystem capacity tiles; locked baseline, 10-minute timer, two Homepage Storage cards (v0.7 manual production acceptance 2026-10-16 CEST) | [Phase-2 deployed runbook](../services/disk-health/README.md) |
| UPS monitoring (host NUT) | Independent PowerWalker monitoring and safe shutdown | [UPS / NUT operations](../services/ups-monitoring/README.md) |
| ShadowBroker | Intelligence and monitoring platform | [ShadowBroker](../services/shadowbroker/README.md) |
| NetAlertX | LAN device inventory and presence monitoring | [NetAlertX](../services/netalertx/README.md) |
| AIOStreams | Self-hosted Stremio stream aggregation/control layer | [AIOStreams](../services/aiostreams/README.md) |
| stremio-libtorrent-server | Central BitTorrent playback, read-ahead, cache, and Stremio streaming server | [stremio-libtorrent-server](../services/stremio-libtorrent-server/README.md) |

Homepage is deployed under `/srv/docker/homepage` using `ghcr.io/gethomepage/homepage:v2.4.0` plus `ghcr.io/tecnativa/docker-socket-proxy:v0.4.2`. The canonical URL `https://home.pirocorp.com` is validated. The apex/public-DNS validation for `pirocorp.com` remains an explicit follow-up after the new DNS-only apex A record was created.

The combined deployed architecture and implementation status for the Stremio stack is recorded in [Stremio + AIOStreams implemented architecture](./stremio-streaming-architecture.md). The original seven-phase baseline and the explicit `6882/TCP+UDP` router-forward follow-up are complete.

## Planned

| Service | Status | Documentation |
| --- | --- | --- |
| AIOStreams multi-source P2P expansion | Planned follow-up; preserves Torrentio and evaluates STorz with Bitmagnet + DMM coverage | [Multi-source P2P expansion](../roadmaps/stremio-aiostreams/multi-source-p2p-expansion.md) |
| Usenet stack | Planned only, not deployed | [Usenet roadmap](../roadmaps/usenet/README.md) |
| ShadowBroker OpenClaw integration | Planned only, not deployed | [Integration roadmap](../roadmaps/shadowbroker-openclaw-integration.md) |
