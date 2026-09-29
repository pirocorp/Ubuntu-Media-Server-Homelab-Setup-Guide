# Homepage Dashboard

Status: Implemented baseline; final apex-DNS and trusted-Tailscale validation still pending
Purpose: Document the deployed PIROCORP HOMELAB Homepage dashboard, Docker integration, publishing model, native widgets, secret handling, and operational checks.
Depends on: [Docker and Portainer](../../platform/docker-and-portainer.md), [Networking and reverse proxy](../../platform/networking-and-reverse-proxy.md)
Related docs: [Current state](../../overview/current-state.md), [Service inventory](../../overview/service-inventory.md), [Homepage architecture decision record](../../roadmaps/homepage/README.md)

## Deployment Snapshot

| Property | Deployed value |
| --- | --- |
| Stack root | `/srv/docker/homepage` |
| Homepage image | `ghcr.io/gethomepage/homepage:v2.4.0` |
| Homepage container | `homepage` |
| Docker Socket Proxy image | `ghcr.io/tecnativa/docker-socket-proxy:v0.4.2` |
| Docker Socket Proxy container | `homepage-dockerproxy` |
| Homepage host endpoint | `192.168.0.10:3002` |
| Homepage container port | `3000/tcp` |
| Canonical URL | `https://home.pirocorp.com` |
| Homepage config | `/srv/docker/homepage/config` |
| Local secrets | `/srv/docker/homepage/secrets` |
| Homepage UID/GID | `1000:1000` (`piroman`) |
| Docker status server name | `piroman-server` |

The original architecture reserved host port `3000`, but deployment preflight showed that AdGuard Home already publishes `192.168.0.10:3000 -> container:80`. Homepage therefore uses `192.168.0.10:3002 -> container:3000`. Port `3001` remains assigned to AIOStreams.

## Docker Access Security Model

Homepage does **not** mount `/var/run/docker.sock` directly.

```text
Homepage
  -> private Docker API network
  -> homepage-dockerproxy:2375
  -> /var/run/docker.sock:ro
  -> Docker daemon
```

The socket proxy is the only container in this stack with the Docker socket mounted. Its `2375/tcp` port is exposed only inside Docker and is not published on the Ubuntu host or LAN.

Validated proxy settings:

```text
CONTAINERS=1
EVENTS=1
PING=1
VERSION=1
POST=0
LOG_LEVEL=warning
```

`POST=0` keeps the proxy read-oriented. Homepage container discovery, status, and on-demand CPU/RAM/RX/TX statistics were validated through the proxy.

Current network names:

```text
homepage_default
homepage_docker_api   # internal Docker API network
```

The runtime check confirmed that host port `2375` is not listening.

## Compose Shape

The deployed Compose file follows this structure. Secret values are never stored in the file.

```yaml
services:
  homepage:
    image: ghcr.io/gethomepage/homepage:v2.4.0
    container_name: homepage
    restart: unless-stopped
    environment:
      HOMEPAGE_ALLOWED_HOSTS: home.pirocorp.com,192.168.0.10:3002
      HOMEPAGE_FILE_PORTAINER_KEY: /app/secrets/portainer_key
      HOMEPAGE_FILE_NPM_USER: /app/secrets/npm_user
      HOMEPAGE_FILE_NPM_PASSWORD: /app/secrets/npm_password
      HOMEPAGE_FILE_ADGUARD_USER: /app/secrets/adguard_user
      HOMEPAGE_FILE_ADGUARD_PASSWORD: /app/secrets/adguard_password
      HOMEPAGE_FILE_NETALERTX_KEY: /app/secrets/netalertx_token
      HOMEPAGE_FILE_PLEX_TOKEN: /app/secrets/plex_token
      HOMEPAGE_FILE_IMMICH_KEY: /app/secrets/immich_key
      HOMEPAGE_FILE_AUDIOBOOKSHELF_KEY: /app/secrets/audiobookshelf_key
      HOMEPAGE_FILE_KAVITA_KEY: /app/secrets/kavita_key
      HOMEPAGE_FILE_QBITTORRENT_KEY: /app/secrets/qbittorrent_key
      HOMEPAGE_FILE_NEXTCLOUD_TOKEN: /app/secrets/nextcloud_token
      PUID: 1000
      PGID: 1000
    ports:
      - "192.168.0.10:3002:3000"
    volumes:
      - ./config:/app/config
      - ./secrets:/app/secrets:ro
    networks:
      - default
      - docker-api
    depends_on:
      - dockerproxy
    security_opt:
      - no-new-privileges:true

  dockerproxy:
    image: ghcr.io/tecnativa/docker-socket-proxy:v0.4.2
    container_name: homepage-dockerproxy
    restart: unless-stopped
    environment:
      CONTAINERS: 1
      POST: 0
      EVENTS: 1
      PING: 1
      VERSION: 1
      LOG_LEVEL: warning
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - docker-api
    security_opt:
      - no-new-privileges:true

networks:
  default:
    name: homepage_default
  docker-api:
    name: homepage_docker_api
    internal: true
```

## Docker Integration

`config/docker.yaml` uses the socket proxy rather than the host socket:

```yaml
piroman-server:
  host: dockerproxy
  port: 2375
```

Validated checks:

- Homepage health: healthy
- `http://192.168.0.10:3002`: HTTP `200`
- `homepage -> dockerproxy:2375/_ping`: `OK`
- Docker API `/version`: `29.1.3`
- Docker `/containers/json`: successful
- Docker CPU, memory, receive, and transmit statistics expand on demand from the green status indicator

## Dashboard Layout

The deployed dashboard has four groups:

```text
Infrastructure
  Server / Netdata
  Portainer
  Nginx Proxy Manager
  AdGuard Home
  NetAlertX

Media & Libraries
  Plex
  Immich
  Audiobookshelf
  Kavita
  Stremio
  AIOStreams

Downloads & Discovery
  qBittorrent
  Bitmagnet

Cloud & Applications
  Nextcloud
  ShadowBroker
```

The initial layout contains 15 top-level cards. Implementation-only database, Redis, worker, machine-learning, I2P, and sidecar containers remain hidden from the top-level dashboard.

The separately deployed `stash` container is intentionally not added to Homepage because it is not yet part of the documented service inventory.

## Native Widgets

| Card | Integration | Deployed fields / notes |
| --- | --- | --- |
| Server / Netdata | Netdata native widget + HTTP monitor | `warnings`, `criticals` |
| Portainer | Portainer native widget + Docker | `running`, `stopped`, `total`; environment ID `3` |
| Nginx Proxy Manager | NPM native widget + Docker | `enabled`, `disabled`, `total` |
| AdGuard Home | AdGuard native widget + Docker | `queries`, `blocked`, `filtered`, `latency` |
| NetAlertX | NetAlertX widget `version: 2` + Docker | `total`, `connected`, `new_devices`, `down_alerts`; backend `192.168.0.10:20212` |
| Plex | Plex native widget + Docker | `streams`, `movies`, `tv` |
| Immich | Immich widget `version: 2` + Docker | `users`, `photos`, `videos`, `storage` |
| Audiobookshelf | Audiobookshelf native widget + Docker | `books`, `booksDuration` |
| Kavita | Kavita native widget + Docker | `seriesCount`, `totalFiles` |
| Stremio | Docker status only | no custom/native widget added |
| AIOStreams | Docker status only | no custom/native widget added |
| qBittorrent | qBittorrent native widget + Docker | `leech`, `download`, `seed`, `upload` |
| Bitmagnet | Docker + HTTP monitor | no custom widget |
| Nextcloud | Nextcloud native widget + Docker | `activeusers`, `numfiles`, `numshares`, `freespace` |
| ShadowBroker | Docker + HTTP monitor | one logical card; no custom widget |

The dashboard keeps Docker resource statistics collapsed by default (`showStats: false`) and uses dot-style status indicators.

## Credentials And Secrets

Secrets are stored only on the Ubuntu host under `/srv/docker/homepage/secrets`, mounted read-only at `/app/secrets`, and referenced through `HOMEPAGE_FILE_*` substitutions.

Current secret files:

```text
portainer_key
npm_user
npm_password
adguard_user
adguard_password
netalertx_token
plex_token
immich_key
audiobookshelf_key
kavita_key
qbittorrent_key
nextcloud_token
```

The files use mode `0600`. Real values must never be committed to this repository.

Credential choices validated during implementation:

- Portainer: API access key, environment ID `3`
- Nginx Proxy Manager: dedicated non-admin/view-oriented account credentials
- AdGuard Home: authenticated API credentials
- NetAlertX: regenerated API token; GraphQL/API backend port `20212`
- Plex: `PlexOnlineToken` extracted locally from the running server configuration
- Immich: dedicated API key restricted to `server.statistics`
- Audiobookshelf: dedicated API key acting as a non-admin `homepage` user
- Kavita: dedicated 32-character `Homepage` authorization key
- qBittorrent `5.2.1`: regenerated 32-character API key using Bearer authentication
- Nextcloud: dedicated 64-character `serverinfo` `NC-Token`

## NetAlertX Firewall Exception

Homepage initially could not reach the NetAlertX v2 backend even though the host API test succeeded. The deployed Homepage default network is `172.31.0.0/16`, so UFW now permits only that subnet to the NetAlertX API port:

```text
172.31.0.0/16 -> 192.168.0.10:20212/tcp
```

UFW comment:

```text
NetAlertX API from Homepage
```

After this rule was added, the Homepage container received HTTP `200` from `/metrics` and the native NetAlertX v2 widget became operational.

## DNS And Reverse Proxy

### Canonical Homepage route

AdGuard Home contains an explicit local rewrite:

```text
home.pirocorp.com -> 192.168.0.10
```

Nginx Proxy Manager has an enabled proxy host:

```text
Domain:                home.pirocorp.com
Scheme:                http
Forward Hostname/IP:   192.168.0.10
Forward Port:          3002
Block Common Exploits: ON
WebSockets Support:    ON
Cache Assets:          OFF
SSL certificate:       pirocorp.com, *.pirocorp.com
Force SSL:             ON
HTTP/2:                ON
HSTS:                  OFF
```

`https://home.pirocorp.com` was validated successfully with the complete dashboard and live widgets.

### Apex and www redirects

AdGuard Home also has explicit local rewrites:

```text
pirocorp.com     -> 192.168.0.10
www.pirocorp.com -> 192.168.0.10
```

Nginx Proxy Manager has one online `301` redirection host for both names:

```text
pirocorp.com
www.pirocorp.com
  -> https://home.pirocorp.com
```

`www.pirocorp.com` resolves through the existing public wildcard and reaches the redirect path.

Cloudflare DNS currently contains DNS-only records pointing to the private LAN address:

```text
*.pirocorp.com -> 192.168.0.10
pirocorp.com   -> 192.168.0.10
```

The explicit apex A record was added because a wildcard never covers the zone apex itself. At the last validation point, Cloudflare's UI showed the apex record but direct A queries to the authoritative nameservers still returned no A address. Therefore the NPM apex redirect is configured but final public-DNS resolution for `pirocorp.com` remains a pending validation item rather than a completed acceptance check.

These DNS-only RFC1918 records do not make the services routable from the public Internet; they are compatibility records for clients that bypass the local AdGuard resolver.

## Validation Snapshot

Completed:

- Homepage and socket-proxy images pulled and started
- Compose syntax validated
- Homepage container healthy
- direct `192.168.0.10:3002` access returns HTTP `200`
- Docker socket proxy read path validated
- `2375` confirmed not published on the host
- all 15 cards render
- Docker status indicators and expandable CPU/RAM/RX/TX statistics work
- Netdata native widget works
- Portainer native widget works
- Nginx Proxy Manager native widget works
- AdGuard Home native widget works
- NetAlertX v2 native widget works after narrow UFW rule
- Plex native widget works
- Immich v2 native widget works
- Audiobookshelf native widget works
- Kavita native widget works
- qBittorrent API-key widget works
- Nextcloud `NC-Token` widget works
- Bitmagnet and ShadowBroker Docker + HTTP health indicators work
- `https://home.pirocorp.com` works through Nginx Proxy Manager and TLS
- AdGuard rewrites exist for `home.pirocorp.com`, `pirocorp.com`, and `www.pirocorp.com`
- NPM `301` redirection host exists for `pirocorp.com` and `www.pirocorp.com`

Still pending at this documentation point:

- validate `https://home.pirocorp.com` from a trusted Tailscale client
- re-test authoritative/public resolution of the newly-created Cloudflare apex `pirocorp.com` A record and then validate the apex `301` redirect end-to-end

## Operations

```bash
cd /srv/docker/homepage

docker compose ps
docker compose logs --tail 100 homepage
docker compose logs --tail 100 dockerproxy
docker compose pull
docker compose up -d
```

Validate that the socket proxy remains private:

```bash
sudo ss -lntp | grep -E ':2375\b' || echo "2375 is not published on host"
```

Validate Homepage directly:

```bash
curl -sS -o /dev/null -w 'HTTP %{http_code}\n' http://192.168.0.10:3002
```

Validate the canonical HTTPS route:

```bash
curl -I https://home.pirocorp.com
```

## Security Boundaries

- Homepage has no direct Docker socket mount.
- Docker Socket Proxy has no host-published `2375` port.
- Docker API mutation is disabled with `POST=0`.
- Secret files are host-local, mode `0600`, and mounted read-only.
- No real API keys, passwords, or tokens belong in Git.
- NetAlertX API access from Docker is restricted to the Homepage default subnet instead of all Docker networks.
- Homepage is intended for LAN and trusted Tailscale use; Cloudflare DNS-only private-address records are not public application exposure.
