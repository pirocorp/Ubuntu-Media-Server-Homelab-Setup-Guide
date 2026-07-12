# Architecture

Status: Implemented
Purpose: Summarize how the homelab is structured across host, networking, storage, and deployed workloads.
Depends on: [Current state](./current-state.md)
Related docs: [Platform](../platform/README.md), [Service inventory](./service-inventory.md)

## High-Level View

```text
Clients
  |
  v
AdGuard Home DNS
  |
  v
Nginx Proxy Manager
  |
  +--> Netdata
  +--> Portainer
  +--> AdGuard Home
  +--> Nextcloud
  +--> Bitmagnet
  +--> qBittorrent
  +--> Plex
  +--> ShadowBroker
  +--> other published services

Ubuntu Server host (192.168.0.10)
  |
  +--> Tailscale subnet router for trusted remote clients
  +--> Docker / Docker Compose
  +--> Portainer
  +--> service stacks under /srv/docker
  +--> storage mounts under /mnt/*
```

## Core Layers

| Layer | Current role |
| --- | --- |
| Ubuntu Server | Base operating system for the homelab |
| SSH | Remote administration path |
| Docker and Compose | Runtime for service stacks |
| Portainer | Container visibility and management |
| AdGuard Home | DNS filtering and local hostname resolution |
| Nginx Proxy Manager | Reverse proxy and HTTPS entry point |
| Tailscale | Private VPN access and subnet routing for trusted clients |
| Netdata and NUT | Host and UPS monitoring |
| NTFS and Linux mounts | Shared data, media, and application storage |

## Storage Model

| Storage area | Primary use |
| --- | --- |
| NVMe SSD | OS, Docker state, databases, configs, metadata |
| `/srv/docker/<app>` | Compose files, env files, app-specific persistent data |
| `/mnt/data` and other mounted drives | Media, downloads, shared files, large application data |

## Operational Model

- The repo is the source of truth for what is currently deployed.
- Platform concerns are documented under `docs/platform/`.
- Each application or workload gets one clear home under `docs/services/`.
- The live Docker stack inventory currently includes 12 stack roots under `/srv/docker`.
- Remote access is provided by Tailscale on the host, advertising `192.168.0.0/24` to trusted clients.
- Historical build walkthrough material is retained in `docs/archive/`.
