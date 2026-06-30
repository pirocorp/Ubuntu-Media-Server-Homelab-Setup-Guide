# Networking And Reverse Proxy

Status: Implemented
Purpose: Document local DNS, ingress routing, and HTTPS publishing across the homelab.
Depends on: [Base server setup](./base-server-setup.md)
Related docs: [Infrastructure HowTo](./infrastructure-howto.md), [Current state](../overview/current-state.md), [Let's Encrypt public-domain guide](./lets-encrypt-public-domain.md), [Legacy build history](../archive/legacy-root-readme.md)

Use the [Infrastructure HowTo](./infrastructure-howto.md) for the ordered setup path. This page captures the steady-state DNS and reverse-proxy model after the platform is live.

## Core Components

| Component | Role |
| --- | --- |
| AdGuard Home | Local DNS resolution, filtering, and rewrites |
| Nginx Proxy Manager | Reverse proxy routing and HTTPS entry point |
| `pirocorp.com` subdomains | Human-readable HTTPS service access |

## Current Publishing Model

| Route | Purpose |
| --- | --- |
| `https://server.pirocorp.com` | Netdata and UPS monitoring |
| `https://portainer.pirocorp.com` | Portainer |
| `https://adguard.pirocorp.com` | AdGuard Home |
| `http://npm.pirocorp.com:81` | Nginx Proxy Manager admin |
| `https://nextcloud.pirocorp.com` | Nextcloud |
| `https://bitmagnet.pirocorp.com` | Bitmagnet |
| `https://qbittorrent.pirocorp.com` | qBittorrent |
| `https://plex.pirocorp.com` | Plex |
| `https://audiobookshelf.pirocorp.com` | Audiobookshelf |
| `https://immich.pirocorp.com` | Immich |
| `https://kavita.pirocorp.com` | Kavita |
| `https://shadowbroker.pirocorp.com` | ShadowBroker |

## Notes

- AdGuard Home provides the DNS layer for local service discovery.
- Nginx Proxy Manager is the central ingress path for published web services.
- The active naming scheme uses `*.pirocorp.com` with Let's Encrypt certificates.
- All currently documented published services now have named URLs in the repo.
- Public-domain TLS migration guidance is documented separately in the [Let's Encrypt guide](./lets-encrypt-public-domain.md).
