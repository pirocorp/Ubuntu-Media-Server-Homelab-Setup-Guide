# AIOStreams Deployment And Operations Runbook

Status: Implemented  
Purpose: Document the deployed self-hosted AIOStreams control plane used to aggregate Stremio stream sources without paid debrid services.  
Depends on: [Docker and Portainer](../../platform/docker-and-portainer.md), [Networking and reverse proxy](../../platform/networking-and-reverse-proxy.md)  
Related docs: [Services index](../README.md), [Service inventory](../../overview/service-inventory.md), [Stremio + AIOStreams architecture roadmap](../../roadmaps/stremio-aiostreams/README.md), [Phase 4 completion](../../roadmaps/stremio-aiostreams/phase-4-completion.md), [stremio-libtorrent-server](../stremio-libtorrent-server/README.md)

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
| Local DNS | explicit AdGuard rewrite to `192.168.0.10` |
| Public DNS fallback | DNS-only `*.pirocorp.com -> 192.168.0.10` |
| Phase 4 source configuration | Torrentio P2P/non-debrid configured and validated |

The deployed instance is healthy and has been validated through both the direct LAN port and the preferred HTTPS URL.

## Architecture

```text
Stremio client / browser
        |
        | https://aio.pirocorp.com
        v
DNS resolution
        |
        +--> normal LAN: AdGuard explicit rewrite
        |                aio.pirocorp.com -> 192.168.0.10
        |
        +--> DNS-bypassing clients: Cloudflare DNS-only wildcard
                         *.pirocorp.com -> 192.168.0.10
        |
        v
Nginx Proxy Manager :443
        |
        | HTTP -> 192.168.0.10:3001
        v
AIOStreams :3000
        |
        +--> Torrentio P2P / non-debrid
        |
        v
normalized Stremio torrent stream results
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

## DNS Publishing

### Normal LAN Resolution — AdGuard Home

The homelab continues to use explicit AdGuard rewrites for normal LAN DNS.

In AdGuard Home:

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

### TV / DNS-Bypass Compatibility — Public Wildcard

The primary TV did not reliably resolve `aio.pirocorp.com` through the local AdGuard resolver, even after manual DNS configuration.

To avoid hard-coded service IPs on devices that bypass local DNS, Cloudflare public DNS now contains:

```text
Type:          A
Name:          *
IPv4 address:  192.168.0.10
Proxy status:  DNS only
TTL:           Auto
```

This produces:

```text
*.pirocorp.com -> 192.168.0.10
```

The record points only to an RFC1918/private address. It does not make the homelab services Internet-routable.

Validation through Google DNS:

```powershell
nslookup aio.pirocorp.com 8.8.8.8
```

Validated response:

```text
Name:    aio.pirocorp.com
Address: 192.168.0.10
```

The explicit AdGuard rewrites remain the preferred LAN DNS model; the public wildcard is a compatibility fallback for clients that ignore or bypass it.

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

## Phase 4 Source Configuration

Torrentio is configured as the initial stream source in P2P/non-debrid mode.

Current baseline:

```text
Resources:          Stream only
URL:                default/blank
Providers:          unrestricted
Services:           unrestricted
Media Types:        unrestricted
Multiple Instances: OFF
```

Filtering baseline:

- no debrid/cache requirement;
- no stream-type inclusion/exclusion requirement;
- CAM, SCR, TS, and TC releases excluded;
- 3D visual tag excluded;
- preferred resolution order includes 2160p, 1440p, 1080p and lower fallbacks;
- preferred quality order favors BluRay REMUX, BluRay, WEB-DL, WEBRip, HDRip and related normal releases;
- existing deduplication remains enabled;
- no seeders sort criterion was added during initial validation.

Proxy/background policy:

```text
AIOStreams built-in media proxy: OFF in practice
proxiedServices: []
proxiedAddons:   []
precacheNextEpisode: false
preloadStreams.enabled: false
```

Autoplay matching uses:

```text
matchingFile
attributes: resolution, quality
```

Statistics output remains enabled for troubleshooting.

The generated configuration URL contains private identifiers/tokens and must not be committed or copied into documentation.

## Stremio Addon Baseline

AIOStreams was installed into the Stremio account so it synchronizes to account clients.

During validation, optional third-party addons were removed because one or more polluted the normal movie/detail flow with deep-link style entries. The stable baseline retained:

- Cinemeta;
- Local Files;
- AIOStreams.

After the cleanup, normal movie navigation invoked AIOStreams and returned expected Torrentio P2P streams.

## Stream Result Validation

Direct AIOStreams stream API testing confirmed that normal stream results contain the data required by the central torrent engine, including:

- `infoHash`;
- `fileIdx`;
- filename;
- video size;
- seeder information.

Multiple titles returned valid P2P results, including Guardians of the Galaxy, Mayday, The Whisper Man, Project Hail Mary, Back Roads, and others used during troubleshooting.

Desktop playback of `Mayday` subsequently produced matching cache and `.fastresume` activity under `/mnt/data/stremio-libtorrent-server`, proving the end-to-end path.

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
- Local AdGuard rewrites remain the preferred LAN DNS model.
- The public DNS-only wildcard maps only to the private RFC1918 address `192.168.0.10`; it does not expose the services directly to the Internet.
- HTTPS terminates at the existing Nginx Proxy Manager wildcard certificate.
- AIOStreams is a control/discovery service and is not intended to be a public Internet management endpoint.
- Phase 4 uses only free P2P-capable upstream sources; paid debrid credentials remain outside the v1 architecture.

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

### TV Does Not Resolve `aio.pirocorp.com`

First verify the local AdGuard rewrite. If the TV still bypasses local DNS, verify the public wildcard through an external resolver:

```powershell
nslookup aio.pirocorp.com 8.8.8.8
```

Expected:

```text
Address: 192.168.0.10
```

### Stremio Shows Unexpected Deep-Link Entries Instead Of P2P Streams

Temporarily reduce the account addon baseline to Cinemeta, Local Files, and AIOStreams, then restart the Stremio client and retest. During Phase 4 this removed conflicting third-party result pollution and restored normal AIOStreams invocation.

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

For a DNS-bypassing client path, also verify `aio.pirocorp.com` against an external resolver.

## Implementation Status

The original seven-phase Stremio/AIOStreams baseline is complete.

Completed:

- self-hosted AIOStreams deployment;
- pinned `v2.34.1` image;
- persistent SQLite/application state;
- direct LAN validation;
- explicit AdGuard rewrite;
- Nginx Proxy Manager publishing with the existing wildcard certificate;
- HTTPS validation;
- Cloudflare DNS-only wildcard fallback for clients that bypass local DNS;
- Torrentio P2P/non-debrid source configuration;
- source-result validation for usable torrent/infoHash/fileIdx data;
- AIOStreams Stremio-account installation;
- end-to-end proof that selected P2P streams reach the central torrent engine;
- full primary-TV/client acceptance including seek/resume and sustained playback;
- explicit `6882/TCP+UDP` router forwarding and peer-path validation;
- large-file read-ahead, seek, cache/LRU, and resilience acceptance.

The next planned AIOStreams work is the [multi-source P2P expansion](../../roadmaps/stremio-aiostreams/multi-source-p2p-expansion.md). It is a source-layer follow-up and does not reopen the completed seven-phase baseline.
