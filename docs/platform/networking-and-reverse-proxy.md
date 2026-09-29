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
| `https://aio.pirocorp.com` | AIOStreams control/discovery endpoint |
| `https://stremio.pirocorp.com` | Browser-facing Stremio Web UI |

The Stremio media path itself does not traverse Nginx Proxy Manager. `stremio-libtorrent-server` continues to use its generated trusted `*.stremio.rocks:12470` endpoint directly for media delivery.

## DNS Model

Normal LAN resolution uses explicit AdGuard Home rewrites for each configured service name. The homelab intentionally keeps those individual local entries instead of replacing them with a local wildcard rewrite.

For clients that ignore or bypass the local resolver, Cloudflare public DNS also contains a DNS-only compatibility wildcard:

```text
*.pirocorp.com -> 192.168.0.10
```

Because the target is an RFC1918/private address, this fallback does not make the services Internet-routable. It does mean that otherwise-unconfigured `*.pirocorp.com` names can resolve publicly to `192.168.0.10`; only explicitly configured Nginx Proxy Manager hosts provide an application route.

Example explicit local rewrite:

```text
netalertx.pirocorp.com -> 192.168.0.10
```

The same explicit-local plus public-wildcard-fallback model is used for AIOStreams and the Stremio Web UI.

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

- AdGuard Home provides the preferred DNS layer for local service discovery.
- Nginx Proxy Manager is the central ingress path for published web services.
- The active naming scheme uses `*.pirocorp.com` with Let's Encrypt certificates.
- Local DNS rewrites remain explicit per service even though the certificate and public compatibility DNS record are wildcard-capable.
- The public wildcard is DNS-only and points to the private LAN IP; it is not equivalent to public Internet exposure.
- AIOStreams and the Stremio Web UI follow the same NPM publishing model as the other named web services.
- Public-domain TLS migration guidance is documented separately in the [Let's Encrypt guide](./lets-encrypt-public-domain.md).
