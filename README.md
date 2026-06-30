# Ubuntu Media Server Homelab Documentation

This repository documents the current state of the homelab, why it is organized this way, how the deployed services are operated, and what is planned next.

The active service-publishing scheme now uses `*.pirocorp.com` with Let's Encrypt certificates. Older `.home` hostnames are retained only where the previous naming scheme or migration steps need to be documented.

## Current Status

### Implemented

- Ubuntu Server host with SSH and Cockpit administration
- Docker and Docker Compose runtime
- Portainer
- AdGuard Home
- Nginx Proxy Manager
- Storage mounts and Samba shares
- Plex
- Nextcloud
- qBittorrent
- Bitmagnet
- UPS monitoring with NUT and Netdata
- ShadowBroker

### Planned

- [Usenet architecture and deployment roadmap](./docs/roadmaps/usenet/README.md)
- [ShadowBroker OpenClaw integration roadmap](./docs/roadmaps/shadowbroker-openclaw-integration.md)

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

Cockpit is currently documented on its direct admin port: `https://192.168.0.10:9090`.

## Architecture Summary

```text
Clients
  -> AdGuard Home DNS
  -> Nginx Proxy Manager
  -> published homelab services

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

## Deployed Service Docs

- [Plex](./docs/services/plex/README.md)
- [Nextcloud](./docs/services/nextcloud/README.md)
- [qBittorrent](./docs/services/qbittorrent/README.md)
- [Bitmagnet](./docs/services/bitmagnet/README.md)
- [UPS monitoring](./docs/services/ups-monitoring/README.md)
- [ShadowBroker](./docs/services/shadowbroker/README.md)

## Platform Docs

- [Base server setup](./docs/platform/base-server-setup.md)
- [Docker and Portainer](./docs/platform/docker-and-portainer.md)
- [Storage and Samba](./docs/platform/storage-and-samba.md)
- [Networking and reverse proxy](./docs/platform/networking-and-reverse-proxy.md)
- [Let's Encrypt public-domain guide](./docs/platform/lets-encrypt-public-domain.md)

## Historical Material

The original large build walkthrough is preserved as historical reference:

- [Legacy root README snapshot](./docs/archive/legacy-root-readme.md)
