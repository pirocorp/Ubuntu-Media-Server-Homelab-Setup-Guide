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
| `https://netalertx.pirocorp.com` | NetAlertX network inventory |

## DNS Model

AdGuard Home provides explicit DNS rewrites for the published service names. The current design intentionally keeps individual entries rather than replacing them with a wildcard rewrite.

NetAlertX uses:

```text
netalertx.pirocorp.com -> 192.168.0.10
```

This keeps unknown names such as unconfigured `*.pirocorp.com` hosts from automatically resolving to the homelab server.

## NetAlertX Reverse Proxy

NetAlertX runs directly on the host with `network_mode: host` and listens on `20211/tcp`.

Nginx Proxy Manager configuration:

```text
Domain:               netalertx.pirocorp.com
Scheme:               http
Forward Hostname/IP:  192.168.0.10
Forward Port:         20211
Block Common Exploits: ON
Websockets Support:    ON
Cache Assets:          OFF
```

SSL configuration:

```text
Certificate:    pirocorp.com, *.pirocorp.com
Force SSL:      ON
HTTP/2 Support: ON
HSTS:           OFF
```

NPM itself is currently on Docker subnet `172.20.0.0/16`. Because NetAlertX uses host networking and UFW restricts `20211/tcp`, the NPM subnet is explicitly allowed to that single destination port:

```text
172.20.0.0/16 -> 192.168.0.10:20211/tcp
```

The service runbook contains the exact validation and troubleshooting commands: [NetAlertX](../services/netalertx/README.md).

## Notes

- AdGuard Home provides the DNS layer for local service discovery.
- Nginx Proxy Manager is the central ingress path for published web services.
- The active naming scheme uses `*.pirocorp.com` with Let's Encrypt certificates.
- DNS rewrites remain explicit per service even though the certificate itself is wildcard-capable.
- All currently documented published services have named URLs in the repo.
- Public-domain TLS migration guidance is documented separately in the [Let's Encrypt guide](./lets-encrypt-public-domain.md).
