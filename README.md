# Ubuntu Media Server Homelab Documentation

This repository documents the current state of the homelab, why it is organized this way, how the deployed services are operated, and what is planned next.

The active service-publishing scheme uses `*.pirocorp.com` with Let's Encrypt certificates plus AdGuard local DNS rewrites to private LAN addresses. Older `.home` hostnames are retained only where the previous naming scheme or migration steps need to be documented.

## Current Status

### Implemented

- Ubuntu Server host with SSH administration
- Docker and Docker Compose runtime
- Portainer
- AdGuard Home
- Nginx Proxy Manager
- Tailscale remote access
- Storage mounts and Samba shares
- Plex
- Nextcloud
- qBittorrent
- Bitmagnet
- [OnTheSpot self-hosted music downloader](./docs/services/onthespot/README.md) — pinned local image, NPM/AdGuard integration and Homepage card; Spotify download acceptance pending
- Host-native NUT safe shutdown with PeaNUT UPS visibility
- ShadowBroker
- Audiobookshelf
- Immich
- Kavita
- NetAlertX network inventory and presence monitoring
- Homepage central homelab dashboard at `https://home.pirocorp.com` with Docker Socket Proxy, native service widgets, and file-backed secrets
- [Phase-1 Server Health Dashboard](./docs/services/server-health/README.md): Glances 4.5.6 (host info strip and its own authenticated FS API), PeaNUT 6.0.0 (PowerWalker UPS), and independent host NUT; Netdata fully decommissioned
- [Phase-2 SMART + Disk Space](./docs/services/disk-health/README.md): host-native Python **v0.8** collects SMART and stable-ID filesystem usage (`os.statvfs`) on the existing 10-minute timer; a **single native Homepage Storage card** displays six combined percentage/temperature/SMART tiles (3×2) plus a separate LAST UPDATED tile, refreshing every 30 seconds. v0.8 production and desktop acceptance verified 2026-10-16 CEST; [unified Storage as-built](./docs/services/disk-health/storage-v08-unified.md)
- AIOStreams self-hosted Stremio stream aggregation/control layer
- `stremio-libtorrent-server` central BitTorrent streaming engine with persistent read-ahead/cache
- Stremio Web UI published internally through AdGuard Home + Nginx Proxy Manager
- [Stremio + AIOStreams self-hosted streaming architecture](./docs/overview/stremio-streaming-architecture.md) — seven-phase baseline complete; client and large-file acceptance passed; explicit `6882/TCP+UDP` router forwarding is enabled and both peer protocols are operationally validated

### Planned

- [Home Energy Monitoring — Home Assistant Container (approved architecture; not deployed)](./docs/roadmaps/home-energy-monitoring/README.md)
- [Taiwan–Russia Risk Watch Discord Bridge](./docs/roadmaps/riskwatch-discord-bridge/README.md)
- [AIOStreams multi-source P2P expansion](./docs/roadmaps/stremio-aiostreams/multi-source-p2p-expansion.md)
- [Usenet architecture and deployment roadmap](./docs/roadmaps/usenet/README.md)
- [ShadowBroker OpenClaw integration roadmap](./docs/roadmaps/shadowbroker-openclaw-integration.md)

## Current Access URLs

These names are intended for local DNS resolution and trusted Tailscale VPN clients. They are not documented here as public internet endpoints.

| Service | URL |
| --- | --- |
| Homepage dashboard | `https://home.pirocorp.com` |
| Glances server monitoring | `https://glances.pirocorp.com` |
| PeaNUT / PowerWalker UPS | `https://peanut.pirocorp.com` |
| Portainer | `https://portainer.pirocorp.com` |
| AdGuard Home | `https://adguard.pirocorp.com` |
| Nginx Proxy Manager admin | `http://npm.pirocorp.com:81` |
| Nextcloud | `https://nextcloud.pirocorp.com` |
| Bitmagnet | `https://bitmagnet.pirocorp.com` |
| OnTheSpot | `https://onthespot.pirocorp.com` |
| qBittorrent | `https://qbittorrent.pirocorp.com` |
| Plex | `https://plex.pirocorp.com` |
| Audiobookshelf | `https://audiobookshelf.pirocorp.com` |
| Immich | `https://immich.pirocorp.com` |
| Kavita | `https://kavita.pirocorp.com` |
| ShadowBroker | `https://shadowbroker.pirocorp.com` |
| NetAlertX | `https://netalertx.pirocorp.com` |
| AIOStreams | `https://aio.pirocorp.com` |
| Stremio Web UI | `https://stremio.pirocorp.com` |

`stremio.pirocorp.com` is the browser-facing Stremio Web UI only. The `stremio-libtorrent-server` media path continues to use the generated trusted `*.stremio.rocks:12470` client endpoint directly, so large media transfers do not traverse Nginx Proxy Manager.

Homepage is the canonical homelab landing page. `www.pirocorp.com` and `pirocorp.com` are configured in Nginx Proxy Manager as `301` redirects to `https://home.pirocorp.com`; final authoritative/public DNS validation for the newly-created apex `pirocorp.com` record is still pending.

## Architecture Summary

```text
General homelab services
  -> AdGuard Home DNS
  -> Nginx Proxy Manager
  -> published homelab services

Homepage dashboard
  -> home.pirocorp.com
  -> AdGuard Home / DNS-only private-address fallback
  -> Nginx Proxy Manager
  -> 192.168.0.10:3002
  -> Homepage
  -> private Docker Socket Proxy -> Docker daemon
  -> Glances :61208 on homepage_default -> authenticated host info strip; Glances Web FS view remains available
  -> PeaNUT :61210 on Ubuntu host -> loopback NUT :3493 -> PowerWalker
  -> read-only /app/public/disk-health/status.json
       <- host-native smartctl + os.statvfs / Python v0.8 collector (10-minute timer)
       <- private baseline and last-observed state under /srv/docker/disk-health/state

Stremio discovery/control
  -> aio.pirocorp.com
  -> AdGuard Home
  -> Nginx Proxy Manager
  -> AIOStreams

Stremio browser UI
  -> stremio.pirocorp.com
  -> AdGuard Home
  -> Nginx Proxy Manager
  -> stremio-libtorrent-server Web UI :8081

Stremio media path
  -> stremio-libtorrent-server
  -> 10 GiB read-ahead
  -> 100 GiB persistent cache
  -> trusted *.stremio.rocks HTTPS :12470

Stremio peer path
  Internet TCP+UDP :6882
  -> router forward
  -> 192.168.0.10:6882
  -> stremio-libtorrent-server

Ubuntu Server host (192.168.0.10)
  -> Docker / Docker Compose
  -> Portainer
  -> /srv/docker stacks
  -> /mnt/* shared storage
```

## Documentation Map

- [Overview](./docs/overview/README.md)
- [Platform](./docs/platform/README.md)
- [Services](./docs/services/README.md)
- [Operations](./docs/operations/README.md)
- [Roadmaps](./docs/roadmaps/README.md)
- [Archive](./docs/archive/README.md)

## How-To And Runbooks

- [Infrastructure HowTo](./docs/platform/infrastructure-howto.md)
- [Operations and common commands](./docs/operations/README.md)
- [Ubuntu swap expansion runbook (8 → 16 GiB)](./docs/operations/swap-expansion-runbook.md)
- [Tailscale remote access runbook](./docs/operations/tailscale-remote-access-runbook.md)
- [IPv6 leak validation and mitigation runbook](./docs/operations/ipv6-leak-validation-and-mitigation-runbook.md)
- [Let's Encrypt public-domain guide](./docs/platform/lets-encrypt-public-domain.md)
- [Homepage deployment and operations runbook](./docs/services/homepage/README.md)
- [Server Health Dashboard: reproducible Glances + PeaNUT runbook](./docs/services/server-health/README.md)
- [SMART Disk Health: Phase-2 deployed runbook](./docs/services/disk-health/README.md)
- [Nextcloud update runbook](./docs/services/nextcloud/update-runbook.md)
- [Immich update and backup runbook](./docs/services/immich/update-runbook.md)
- [qBittorrent seedbox runbook](./docs/services/qbittorrent/README.md)
- [Bitmagnet runbook](./docs/services/bitmagnet/README.md)
- [OnTheSpot infrastructure runbook](./docs/services/onthespot/README.md)
- [ShadowBroker operations runbook](./docs/services/shadowbroker/README.md)
- [NetAlertX deployment and operations runbook](./docs/services/netalertx/README.md)
- [AIOStreams deployment and operations runbook](./docs/services/aiostreams/README.md)
- [stremio-libtorrent-server deployment and operations runbook](./docs/services/stremio-libtorrent-server/README.md)
- [Stremio Web UI publishing runbook](./docs/services/stremio-libtorrent-server/web-ui-publishing.md)

Start with the infrastructure guide when rebuilding the base environment. Use the workload runbooks only after the platform itself is ready.

## Deployed Service Docs

- [Homepage](./docs/services/homepage/README.md)
- [Plex](./docs/services/plex/README.md)
- [Nextcloud](./docs/services/nextcloud/README.md)
- [qBittorrent](./docs/services/qbittorrent/README.md)
- [Bitmagnet](./docs/services/bitmagnet/README.md)
- [OnTheSpot](./docs/services/onthespot/README.md)
- [Audiobookshelf](./docs/services/audiobookshelf/README.md)
- [Immich](./docs/services/immich/README.md)
- [Kavita](./docs/services/kavita/README.md)
- [UPS / NUT shutdown monitoring](./docs/services/ups-monitoring/README.md)
- [Server Health Dashboard](./docs/services/server-health/README.md)
- [SMART Disk Health](./docs/services/disk-health/README.md)
- [ShadowBroker](./docs/services/shadowbroker/README.md)
- [NetAlertX](./docs/services/netalertx/README.md)
- [AIOStreams](./docs/services/aiostreams/README.md)
- [stremio-libtorrent-server](./docs/services/stremio-libtorrent-server/README.md)

## Platform Docs

- [Infrastructure HowTo](./docs/platform/infrastructure-howto.md)
- [Base server setup](./docs/platform/base-server-setup.md)
- [Docker and Portainer](./docs/platform/docker-and-portainer.md)
- [Storage and Samba](./docs/platform/storage-and-samba.md)
- [Networking and reverse proxy](./docs/platform/networking-and-reverse-proxy.md)
- [Let's Encrypt public-domain guide](./docs/platform/lets-encrypt-public-domain.md)

## Historical Material

The original large build walkthrough is preserved as historical reference:

- [Legacy root README snapshot](./docs/archive/legacy-root-readme.md)
