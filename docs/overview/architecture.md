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
  +--> Homepage
  +--> Glances
  +--> PeaNUT
  +--> Portainer
  +--> AdGuard Home
  +--> Nextcloud
  +--> Bitmagnet
  +--> qBittorrent
  +--> Plex
  +--> ShadowBroker
  +--> other published services

Homepage
  |
  +--> native service APIs / HTTP health
  +--> Glances:61208 (shared Docker bridge; host + FS metrics)
  +--> PeaNUT 192.168.0.10:61210 (host network; NUT loopback:3493)
  +--> private Docker Socket Proxy
        |
        +--> Docker daemon read path

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
| Homepage | Central operational landing page and compact service health/status surface |
| AdGuard Home | DNS filtering and local hostname resolution |
| Nginx Proxy Manager | Reverse proxy and HTTPS entry point |
| Tailscale | Private VPN access and subnet routing for trusted clients |
| Homepage + Glances | Current CPU, RAM, host temperature, uptime and six filesystem capacity tiles |
| PeaNUT + host NUT | Authenticated UPS visibility; native Ubuntu NUT retains safe shutdown |
| NTFS and Linux mounts | Shared data, media, and application storage |

## Storage Model

| Storage area | Primary use |
| --- | --- |
| Samsung 870 SATA system SSD | OS, Docker state, databases, configs, metadata |
| `/srv/docker/<app>` | Compose files, env files, app-specific persistent data |
| `/mnt/data` and other mounted drives | Media, downloads, shared files, large application data |

## Operational Model

- The repo is the source of truth for what is currently deployed.
- Platform concerns are documented under `docs/platform/`.
- Each application or workload gets one clear home under `docs/services/`.
- Docker Compose stacks are independently managed under `/srv/docker`, including `/srv/docker/glances` and `/srv/docker/peanut`; verify the actual stack count on the host.
- Homepage uses a dedicated Docker Socket Proxy instead of a direct `/var/run/docker.sock` mount; the proxy is private to Docker and mutation is disabled with `POST=0`.
- Remote access is provided by Tailscale on the host, advertising `192.168.0.0/24` to trusted clients.
- Historical build walkthrough material is retained in `docs/archive/`.
