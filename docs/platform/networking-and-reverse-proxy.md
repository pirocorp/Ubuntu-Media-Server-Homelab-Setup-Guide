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
| `https://home.pirocorp.com` | Homepage central dashboard |
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

For clients that ignore or bypass the local resolver, Cloudflare public DNS contains DNS-only private-address compatibility records:

```text
*.pirocorp.com -> 192.168.0.10
pirocorp.com   -> 192.168.0.10
```

The explicit apex A record was added on 2026-09-29 because `*.pirocorp.com` does not cover `pirocorp.com` itself. At the last validation point, the Cloudflare UI showed the apex record but direct A queries to both authoritative Cloudflare nameservers still returned no A address; authoritative/public propagation remains pending validation.

Because the targets are RFC1918/private addresses, these records do not make the services Internet-routable. They do mean that otherwise-unconfigured wildcard-covered names can resolve publicly to `192.168.0.10`; only explicitly configured Nginx Proxy Manager hosts provide an application route.

Homepage has explicit AdGuard rewrites for all three landing/redirect names:

```text
home.pirocorp.com -> 192.168.0.10
pirocorp.com      -> 192.168.0.10
www.pirocorp.com  -> 192.168.0.10
```

The same explicit-local plus public-wildcard-fallback model is used for AIOStreams and the Stremio Web UI.

## Homepage Reverse Proxy And Redirects

The canonical Homepage route is:

```text
https://home.pirocorp.com
  -> Nginx Proxy Manager
  -> http://192.168.0.10:3002
  -> Homepage container :3000
```

Nginx Proxy Manager proxy-host settings:

```text
Domain:                home.pirocorp.com
Scheme:                http
Forward Hostname/IP:   192.168.0.10
Forward Port:          3002
Block Common Exploits: ON
Websockets Support:    ON
Cache Assets:          OFF
Certificate:           pirocorp.com, *.pirocorp.com
Force SSL:             ON
HTTP/2 Support:        ON
HSTS:                  OFF
```

`https://home.pirocorp.com` was validated successfully with the complete dashboard and live widgets.

Nginx Proxy Manager also has one online redirection host containing both:

```text
pirocorp.com
www.pirocorp.com
```

with:

```text
HTTP code:    301
Scheme:       https
Destination:  home.pirocorp.com
```

`www.pirocorp.com` resolves through the wildcard compatibility record. End-to-end apex redirect validation remains pending until the new public `pirocorp.com` record is returned by the authoritative/public DNS path.

## NetAlertX Reverse Proxy

NetAlertX runs directly on the host with `network_mode: host` and listens on `20211/tcp` for the web UI. Its v2 API/metrics backend uses `20212/tcp`.

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

Homepage's native NetAlertX v2 widget needs the backend API port. Homepage's default Docker network is `172.31.0.0/16`, so UFW allows only that subnet to the API destination:

```text
172.31.0.0/16 -> 192.168.0.10:20212/tcp
```

The service runbooks contain the validation and troubleshooting commands: [NetAlertX](../services/netalertx/README.md) and [Homepage](../services/homepage/README.md).

## Notes

- AdGuard Home provides the preferred DNS layer for local service discovery.
- Nginx Proxy Manager is the central ingress path for published web services.
- The active naming scheme uses `pirocorp.com` / `*.pirocorp.com` with Let's Encrypt certificates.
- Local DNS rewrites remain explicit per service even though the certificate and public compatibility DNS records are wildcard-capable.
- Public DNS-only records point to the private LAN IP; they are not equivalent to public Internet exposure.
- Homepage, AIOStreams, and the Stremio Web UI follow the standard NPM publishing model.
- `home.pirocorp.com` is the canonical dashboard URL; apex and `www` are redirect aliases.
- Public-domain TLS migration guidance is documented separately in the [Let's Encrypt guide](./lets-encrypt-public-domain.md).
