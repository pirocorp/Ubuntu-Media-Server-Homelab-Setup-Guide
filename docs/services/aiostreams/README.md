# AIOStreams Deployment And Operations Runbook

Status: Implemented  
Purpose: Document the deployed self-hosted AIOStreams control plane used to aggregate Stremio stream sources without paid debrid services.  
Depends on: [Docker and Portainer](../../platform/docker-and-portainer.md), [Networking and reverse proxy](../../platform/networking-and-reverse-proxy.md)  
Related docs: [Services index](../README.md), [Service inventory](../../overview/service-inventory.md), [Stremio + AIOStreams architecture roadmap](../../roadmaps/stremio-aiostreams/README.md), [stremio-libtorrent-server](../stremio-libtorrent-server/README.md)

## Current Deployment

AIOStreams is the stream aggregation and control layer for the Stremio architecture. It does not download or proxy media. Its job is to query configured Stremio stream addons, normalize and deduplicate results, apply filtering/sorting, and return torrent-capable stream entries to the client.

| Item | Current value |
| --- | --- |
| AIOStreams version | `v2.34.1` |
| Image | `ghcr.io/viren070/aiostreams:v2.34.1` |
| Host | `piroman-server` / `192.168.0.10` |
| Stack root | `/srv/docker/aiostreams` |
| Persistent state | `/srv/docker/aiostreams/data` |
| Container port | `3000/tcp` |
| Published host port | `192.168.0.10:3001/tcp` |
| Preferred URL | `https://aio.pirocorp.com` |
| Database | SQLite |
| Public DNS | no public `A` record; validated as NXDOMAIN from Cloudflare DNS |
| Phase 4 source configuration | not configured yet |

The deployed instance is healthy and has been validated through both the direct LAN port and the preferred HTTPS URL.

## Architecture

```text
Stremio client / browser
        |
        | https://aio.pirocorp.com
        v
AdGuard Home explicit DNS rewrite
        |
        | aio.pirocorp.com -> 192.168.0.10
        v
Nginx Proxy Manager :443
        |
        | HTTP -> 192.168.0.10:3001
        v
AIOStreams :3000
        |
        +--> Torrentio / other free P2P-capable addons  [Phase 4]
        |
        v
normalized Stremio stream results
```

AIOStreams is intentionally separated from the media path. After a user selects a torrent result, Stremio passes the torrent information to the configured `stremio-libtorrent-server`; AIOStreams is not in the video byte path.

## Storage Layout

```text
/srv/docker/aiostreams
├── .env
├── compose.yaml
└── data
    ├── db.sqlite
    ├── db.sqlite-shm
    ├── db.sqlite-wal
    ├── anime-database/
    ├── id-mappings/
    ├── scene-mappings/
    ├── seadex/
    └── instance-id
```

The bind mount is:

```text
/srv/docker/aiostreams/data -> /app/data
```

This keeps the SQLite database and synchronized datasets outside the container filesystem so container recreation or image upgrades do not erase application state.

## Installation

### 1. Create The Stack Root

```bash
sudo mkdir -p /srv/docker/aiostreams
sudo chown -R piroman:piroman /srv/docker/aiostreams
mkdir -p /srv/docker/aiostreams/data
```

### 2. Create `.env`

The real `SECRET_KEY` must be generated locally and must never be committed to Git.

Generate a secret:

```bash
openssl rand -hex 32
```

Create:

```text
/srv/docker/aiostreams/.env
```

Sanitized example:

```env
AIOSTREAMS_VERSION=v2.34.1
BASE_URL=https://aio.pirocorp.com
SECRET_KEY=<generate-with-openssl-rand-hex-32>
DATABASE_URI=sqlite://./data/db.sqlite
```

Restrict permissions:

```bash
chmod 600 /srv/docker/aiostreams/.env
```

Expected ownership and mode in the current deployment:

```text
600 piroman:piroman /srv/docker/aiostreams/.env
```

### 3. Create `compose.yaml`

```yaml
services:
  aiostreams:
    image: ghcr.io/viren070/aiostreams:${AIOSTREAMS_VERSION}
    container_name: aiostreams
    restart: unless-stopped
    env_file:
      - .env
    ports:
      - "192.168.0.10:3001:3000"
    volumes:
      - ./data:/app/data
```

The image version is pinned through `.env`; do not replace it with `latest` during routine operation.

### 4. Validate Before Start

```bash
cd /srv/docker/aiostreams
docker compose config -q && echo "Compose syntax OK"
docker compose config --images
```

Expected image:

```text
ghcr.io/viren070/aiostreams:v2.34.1
```

### 5. Pull And Start

```bash
docker compose pull
docker compose up -d
```

Validate:

```bash
docker compose ps
docker compose logs --tail=100 aiostreams
```

Expected runtime state:

```text
Up (...) (healthy)
192.168.0.10:3001->3000/tcp
```

The first start may apply database migrations and synchronize bundled datasets before the health status becomes healthy.

## Direct LAN Validation

Test the AIOStreams configuration route directly:

```bash
curl -sS -o /dev/null -w 'HTTP %{http_code}\n' \
  http://192.168.0.10:3001/stremio/configure
```

Expected:

```text
HTTP 200
```

The direct browser URL is:

```text
http://192.168.0.10:3001/stremio/configure
```

## DNS Publishing With AdGuard Home

The homelab uses explicit DNS rewrites rather than wildcard local DNS.

In AdGuard Home add:

```text
aio.pirocorp.com -> 192.168.0.10
```

Validate from the server or LAN client:

```bash
getent hosts aio.pirocorp.com
```

Expected:

```text
192.168.0.10 aio.pirocorp.com
```

The public DNS namespace was also checked separately through Cloudflare DNS-over-HTTPS and returned NXDOMAIN for `aio.pirocorp.com`. This confirms that the hostname is currently supplied by local AdGuard DNS rather than a public `A` record.

## Nginx Proxy Manager

Create a dedicated Proxy Host.

### Details

```text
Domain Names:          aio.pirocorp.com
Scheme:                http
Forward Hostname/IP:   192.168.0.10
Forward Port:          3001
Access List:           Publicly Accessible
Cache Assets:          OFF
Block Common Exploits: ON
Websockets Support:    ON
```

### SSL

Select the existing certificate covering:

```text
pirocorp.com
*.pirocorp.com
```

Enable:

```text
Force SSL:      ON
HTTP/2 Support: ON
HSTS:           OFF
```

Preferred URL:

```text
https://aio.pirocorp.com
```

### Validate NPM-To-AIOStreams Connectivity

NPM has `curl` available. Validate the backend path directly from the NPM container:

```bash
docker exec nginx-proxy-manager \
  curl -sS -o /dev/null -w 'HTTP %{http_code}\n' \
  --connect-timeout 5 \
  http://192.168.0.10:3001/stremio/configure
```

Expected:

```text
HTTP 200
```

No additional UFW rule was required in the validated deployment because this path already worked from the NPM container.

Validate the final HTTPS route:

```bash
curl -sS -o /dev/null -w 'HTTP %{http_code}\n' \
  https://aio.pirocorp.com/stremio/configure
```

Expected:

```text
HTTP 200
```

## Persistent State Validation

After first successful startup:

```bash
ls -lah /srv/docker/aiostreams/data
```

Expected state includes at least:

```text
db.sqlite
db.sqlite-shm
db.sqlite-wal
anime-database/
id-mappings/
scene-mappings/
seadex/
instance-id
```

These files may be created as `root:root` by the container. Do not change ownership while the application is working correctly unless a concrete permission problem is observed.

## Docker Operations

```bash
cd /srv/docker/aiostreams
```

Start:

```bash
docker compose up -d
```

Restart:

```bash
docker compose restart
```

Stop:

```bash
docker compose down
```

Status:

```bash
docker compose ps
```

Logs:

```bash
docker compose logs -f aiostreams
```

## Updating

AIOStreams is pinned through:

```text
AIOSTREAMS_VERSION=v2.34.1
```

Before changing versions:

1. verify the desired upstream release exists;
2. review release notes and configuration changes;
3. back up the persistent `data/` directory;
4. update only `AIOSTREAMS_VERSION` in `.env`;
5. validate the resolved image before pulling.

Commands:

```bash
cd /srv/docker/aiostreams
docker compose config -q
docker compose config --images
docker compose pull
docker compose up -d
docker compose ps
```

Then revalidate both the direct route and `https://aio.pirocorp.com/stremio/configure`.

## Backup

Persistent state is under:

```text
/srv/docker/aiostreams/data
```

A simple stopped-container backup is:

```bash
cd /srv/docker/aiostreams
docker compose down
tar -czf aiostreams-data-backup.tar.gz data/
docker compose up -d
```

Keep the real `.env` outside Git and include it only in a protected secrets backup if recovery of the same instance secret is required.

## Security Notes

- Never commit the real AIOStreams `SECRET_KEY`.
- Never commit generated Stremio configuration URLs containing private configuration tokens.
- The service binds its direct host port only to `192.168.0.10`, not to `0.0.0.0`.
- The preferred hostname is provided by local AdGuard DNS; the validated public DNS response for `aio.pirocorp.com` is NXDOMAIN.
- HTTPS terminates at the existing Nginx Proxy Manager wildcard certificate.
- AIOStreams is a control/discovery service and is not intended to be a public Internet management endpoint.
- Phase 4 must continue to use only free P2P-capable upstream sources; paid debrid credentials are outside the v1 architecture.

## Troubleshooting

### `Page not found` At The Direct URL

Verify the browser address is exactly:

```text
http://192.168.0.10:3001/stremio/configure
```

The application correctly returns a page-not-found response for invalid routes.

### Container Remains `health: starting`

Wait for first-run migrations and dataset synchronization, then inspect:

```bash
docker compose logs --tail=150 aiostreams
docker compose ps
```

### NPM Returns A Gateway Error

Check backend reachability from inside NPM:

```bash
docker exec nginx-proxy-manager \
  curl -sS -o /dev/null -w 'HTTP %{http_code}\n' \
  --connect-timeout 5 \
  http://192.168.0.10:3001/stremio/configure
```

If direct LAN access works but this test fails, inspect Docker networking and UFW before changing AIOStreams itself.

### DNS Does Not Resolve

Verify the explicit AdGuard rewrite and then run:

```bash
getent hosts aio.pirocorp.com
```

## Operational Health Checklist

```bash
cd /srv/docker/aiostreams
docker compose ps
docker compose logs --tail=100 aiostreams
getent hosts aio.pirocorp.com
curl -sS -o /dev/null -w 'HTTP %{http_code}\n' http://192.168.0.10:3001/stremio/configure
curl -sS -o /dev/null -w 'HTTP %{http_code}\n' https://aio.pirocorp.com/stremio/configure
ls -lah /srv/docker/aiostreams/data
```

## Implementation Status

Completed in Phase 2:

- self-hosted AIOStreams deployment;
- pinned `v2.34.1` image;
- persistent SQLite/application state;
- direct LAN validation;
- explicit AdGuard rewrite;
- Nginx Proxy Manager publishing with the existing wildcard certificate;
- HTTPS validation;
- public-DNS NXDOMAIN validation.

Still pending in later phases:

- Torrentio P2P/non-debrid source configuration;
- source-result validation for usable torrent/infohash data;
- deduplication/filter/sort tuning;
- client addon installation and end-to-end torrent playback validation.
