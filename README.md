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
- UPS monitoring with NUT and Netdata
- ShadowBroker
- Audiobookshelf
- Immich
- Kavita
- NetAlertX network inventory and presence monitoring
- AIOStreams self-hosted Stremio stream aggregation/control layer
- `stremio-libtorrent-server` central BitTorrent streaming engine with persistent read-ahead/cache

### Partially Implemented

- [Stremio + AIOStreams self-hosted streaming architecture](./docs/overview/stremio-streaming-architecture.md) — Phases 1-3 are deployed; source configuration, client cutover, peer forwarding, and resilience testing remain pending.

### Planned

- [Homepage dashboard architecture roadmap](./docs/roadmaps/homepage/README.md)
- [Usenet architecture and deployment roadmap](./docs/roadmaps/usenet/README.md)
- [ShadowBroker OpenClaw integration roadmap](./docs/roadmaps/shadowbroker-openclaw-integration.md)

## Current Access URLs

These names are intended for local DNS resolution and trusted Tailscale VPN clients. They are not documented here as public internet endpoints.

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
| Audiobookshelf | `https://audiobookshelf.pirocorp.com` |
| Immich | `https://immich.pirocorp.com` |
| Kavita | `https://kavita.pirocorp.com` |
| ShadowBroker | `https://shadowbroker.pirocorp.com` |
| NetAlertX | `https://netalertx.pirocorp.com` |
| AIOStreams | `https://aio.pirocorp.com` |

`stremio-libtorrent-server` uses a generated trusted `*.stremio.rocks:12470` client endpoint. The instance-specific hostname is recorded by the running server and documented in the service runbook rather than treated as a permanent `pirocorp.com` service name.

## Architecture Summary

```text
General homelab services
  -> AdGuard Home DNS
  -> Nginx Proxy Manager
  -> published homelab services

Stremio discovery/control
  -> aio.pirocorp.com
  -> AdGuard Home
  -> Nginx Proxy Manager
  -> AIOStreams

Stremio media path
  -> stremio-libtorrent-server
  -> 10 GiB read-ahead
  -> 300 GiB persistent cache
  -> trusted *.stremio.rocks HTTPS

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
- [Tailscale remote access runbook](./docs/operations/tailscale-remote-access-runbook.md)
- [IPv6 leak validation and mitigation runbook](./docs/operations/ipv6-leak-validation-and-mitigation-runbook.md)
- [Let's Encrypt public-domain guide](./docs/platform/lets-encrypt-public-domain.md)
- [Nextcloud update runbook](./docs/services/nextcloud/update-runbook.md)
- [Immich update and backup runbook](./docs/services/immich/update-runbook.md)
- [qBittorrent seedbox runbook](./docs/services/qbittorrent/README.md)
- [Bitmagnet runbook](./docs/services/bitmagnet/README.md)
- [ShadowBroker operations runbook](./docs/services/shadowbroker/README.md)
- [NetAlertX deployment and operations runbook](./docs/services/netalertx/README.md)
- [AIOStreams deployment and operations runbook](./docs/services/aiostreams/README.md)
- [stremio-libtorrent-server deployment and operations runbook](./docs/services/stremio-libtorrent-server/README.md)

Start with the infrastructure guide when rebuilding the base environment. Use the workload runbooks only after the platform itself is ready.

## Deployed Service Docs

- [Plex](./docs/services/plex/README.md)
- [Nextcloud](./docs/services/nextcloud/README.md)
- [qBittorrent](./docs/services/qbittorrent/README.md)
- [Bitmagnet](./docs/services/bitmagnet/README.md)
- [Audiobookshelf](./docs/services/audiobookshelf/README.md)
- [Immich](./docs/services/immich/README.md)
- [Kavita](./docs/services/kavita/README.md)
- [UPS monitoring](./docs/services/ups-monitoring/README.md)
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
