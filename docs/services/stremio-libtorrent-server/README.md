# stremio-libtorrent-server Deployment And Operations Runbook

Status: Implemented  
Purpose: Document the deployed central Stremio BitTorrent streaming engine, persistent cache, trusted HTTPS endpoint, and operational validation.  
Depends on: [Docker and Portainer](../../platform/docker-and-portainer.md), [Storage and Samba](../../platform/storage-and-samba.md), [Networking and reverse proxy](../../platform/networking-and-reverse-proxy.md)  
Related docs: [Services index](../README.md), [Service inventory](../../overview/service-inventory.md), [Stremio + AIOStreams architecture roadmap](../../roadmaps/stremio-aiostreams/README.md), [AIOStreams](../aiostreams/README.md), [qBittorrent](../qbittorrent/README.md)

## Current Deployment

`stremio-libtorrent-server` is the central torrent playback engine for the Stremio architecture. It is responsible for joining BitTorrent swarms, downloading and prioritizing torrent pieces around the playhead, building server-side read-ahead, retaining a persistent disk cache, and serving the resulting media stream to Stremio clients.

| Item | Current value |
| --- | --- |
| Version | `1.6.15` |
| Image | `androshack/stremio-libtorrent-server:1.6.15` |
| Host | `piroman-server` / `192.168.0.10` |
| Stack root | `/srv/docker/stremio-libtorrent-server` |
| Persistent data/cache | `/mnt/data/stremio-libtorrent-server` |
| Cache filesystem | `/mnt/data` on NTFS/fuseblk |
| Cache budget | `300GiB` |
| Read-ahead target | `10GiB` |
| Download rate limit | `0` / unlimited |
| Adaptive picking | disabled |
| BitTorrent listen port | `6882/TCP+UDP` |
| Web UI host port | `192.168.0.10:8081 -> 8080/tcp` |
| Streaming API | `192.168.0.10:11470 -> 11470/tcp` |
| Trusted HTTPS | `192.168.0.10:12470 -> 12470/tcp` |
| Client TLS method | upstream trusted `*.stremio.rocks` certificate generated from `IPADDRESS=192.168.0.10` |
| Transcoding | not part of the normal v1 path; no GPU overlay configured |
| Router forwarding for `6882` | not completed yet; Phase 6 |

The container is healthy and the trusted HTTPS endpoint has been validated with normal certificate verification, without `curl -k`.

## Why Port `6882`

qBittorrent already owns:

```text
6881/TCP
6881/UDP
```

The Stremio torrent engine therefore uses:

```text
6882/TCP
6882/UDP
```

The BitTorrent listen port inside the container and the published host port must match. Do not map host `6882` to container `6881`; the application itself is configured to listen on `6882`.

## Architecture

```text
Stremio client
      |
      | selected torrent infohash / file index
      v
stremio-libtorrent-server
      |
      +--> DHT / trackers / PEX / peers
      |       |
      |       +--> 6882/TCP+UDP
      |
      +--> playhead-first torrent download
      +--> 10 GiB read-ahead target
      +--> 300 GiB persistent cache
      |       |
      |       +--> /mnt/data/stremio-libtorrent-server
      |
      v
trusted HTTPS stream :12470
      |
      v
Stremio player
```

The media path intentionally does not traverse Nginx Proxy Manager. The upstream trusted `*.stremio.rocks` HTTPS endpoint is used directly so large video transfers do not add an unnecessary reverse-proxy hop.

AIOStreams remains separate from this path. It provides source discovery and result normalization; it does not proxy the selected media stream.

## Preflight Decisions

The deployment was validated against the live host before installation.

### Host Resources

```text
Architecture: x86_64
CPU: Intel Xeon E3-1231 v3 @ 3.40 GHz
CPU topology: 4 cores / 8 threads
RAM: 30 GiB total
GPU: NVIDIA GeForce GT 730
```

Transcoding was explicitly excluded from the v1 design, so the GPU is not passed into the container.

### Storage

`/mnt/data` was selected for the persistent cache. At preflight it had approximately:

```text
Filesystem: fuseblk / NTFS
Size:       7.3 TiB
Available:  4.1 TiB
```

This provides ample headroom for the locked `300GiB` cache budget.

### Ports Verified Free Before Deployment

```text
8081/tcp
11470/tcp
12470/tcp
6882/tcp
6882/udp
```

Host `8080` was deliberately not used because qBittorrent already publishes its web UI there.

## Storage Layout

```text
/srv/docker/stremio-libtorrent-server
├── .env
└── compose.yaml

/mnt/data/stremio-libtorrent-server
├── .resume/
├── .evictor-owner
├── certificates.pem
└── httpsCert.json
```

The persistent bind mount is:

```text
/mnt/data/stremio-libtorrent-server -> /root/.stremio-server
```

Torrent cache contents are added under this same persistent root as playback begins.

`certificates.pem` contains private key material. Never commit it to Git or copy it into unprotected documentation artifacts.

## Installation

### 1. Create Directories

```bash
sudo mkdir -p /srv/docker/stremio-libtorrent-server
sudo chown -R piroman:piroman /srv/docker/stremio-libtorrent-server
sudo mkdir -p /mnt/data/stremio-libtorrent-server
```

### 2. Create `.env`

```env
STREMIO_LIBTORRENT_VERSION=1.6.15
STREMIOSRV_BT_LISTEN_PORT=6882
STREMIOSRV_CACHE_SIZE=300GiB
STREMIOSRV_READAHEAD_BYTES=10GiB
STREMIOSRV_DOWNLOAD_RATE_LIMIT=0
STREMIOSRV_ADAPTIVE_PICKING=false
STREMIO_DATA_DIR=/mnt/data/stremio-libtorrent-server
IPADDRESS=192.168.0.10
```

Restrict the file:

```bash
chmod 600 /srv/docker/stremio-libtorrent-server/.env
```

Validated deployment mode:

```text
600 piroman:piroman /srv/docker/stremio-libtorrent-server/.env
```

### 3. Create `compose.yaml`

```yaml
services:
  stremio-libtorrent-server:
    image: androshack/stremio-libtorrent-server:${STREMIO_LIBTORRENT_VERSION}
    container_name: stremio-libtorrent-server
    restart: unless-stopped

    environment:
      STREMIOSRV_BT_LISTEN_PORT: "${STREMIOSRV_BT_LISTEN_PORT}"
      STREMIOSRV_CACHE_ROOT: /root/.stremio-server
      STREMIOSRV_CACHE_SIZE: "${STREMIOSRV_CACHE_SIZE}"
      STREMIOSRV_READAHEAD_BYTES: "${STREMIOSRV_READAHEAD_BYTES}"
      STREMIOSRV_DOWNLOAD_RATE_LIMIT: "${STREMIOSRV_DOWNLOAD_RATE_LIMIT}"
      STREMIOSRV_ADAPTIVE_PICKING: "${STREMIOSRV_ADAPTIVE_PICKING}"
      IPADDRESS: "${IPADDRESS}"

    ports:
      - "192.168.0.10:8081:8080/tcp"
      - "192.168.0.10:11470:11470/tcp"
      - "192.168.0.10:12470:12470/tcp"
      - "${STREMIOSRV_BT_LISTEN_PORT}:${STREMIOSRV_BT_LISTEN_PORT}/tcp"
      - "${STREMIOSRV_BT_LISTEN_PORT}:${STREMIOSRV_BT_LISTEN_PORT}/udp"

    volumes:
      - "${STREMIO_DATA_DIR}:/root/.stremio-server"

    healthcheck:
      test: ["CMD", "curl", "-fsS", "http://127.0.0.1:11470/health"]
      interval: 30s
      timeout: 5s
      retries: 3
```

There is intentionally no NVIDIA, VAAPI, or other GPU/transcoding overlay in v1.

### 4. Validate Before Start

```bash
cd /srv/docker/stremio-libtorrent-server
docker compose config -q && echo "Compose syntax OK"
docker compose config --images
```

Expected image:

```text
androshack/stremio-libtorrent-server:1.6.15
```

Verify the upstream tag exists before first deployment or a future version change:

```bash
docker manifest inspect \
  androshack/stremio-libtorrent-server:1.6.15 \
  >/dev/null && echo "Image 1.6.15 available"
```

### 5. Pull And Start

```bash
docker compose pull
docker compose up -d
docker compose ps
```

Expected state:

```text
Up (...) (healthy)
```

Expected published ports include:

```text
192.168.0.10:8081->8080/tcp
192.168.0.10:11470->11470/tcp
192.168.0.10:12470->12470/tcp
0.0.0.0:6882->6882/tcp
0.0.0.0:6882->6882/udp
```

Docker may also display `6881/tcp` as exposed image metadata. That is not a published host binding and does not conflict with qBittorrent. The actual host peer port remains `6882`.

## Startup Validation

Inspect logs:

```bash
docker compose logs --tail=150 stremio-libtorrent-server
```

Healthy first startup should show:

- trusted `stremio.rocks` certificate acquisition;
- a generated `A` record for the LAN-IP-based hostname;
- HTTPS metadata saved under `/root/.stremio-server`;
- cache evictor started with `budget=300.0 GiB`;
- application startup complete;
- Uvicorn listening on `0.0.0.0:11470`.

## Validate Runtime Configuration

Do not rely only on `.env`; verify the actual container environment:

```bash
docker exec stremio-libtorrent-server env | \
  grep -E '^STREMIOSRV_(BT_LISTEN_PORT|CACHE_SIZE|READAHEAD_BYTES|DOWNLOAD_RATE_LIMIT|ADAPTIVE_PICKING)='
```

Expected:

```text
STREMIOSRV_READAHEAD_BYTES=10GiB
STREMIOSRV_DOWNLOAD_RATE_LIMIT=0
STREMIOSRV_ADAPTIVE_PICKING=false
STREMIOSRV_BT_LISTEN_PORT=6882
STREMIOSRV_CACHE_SIZE=300GiB
```

## HTTP API Validation

Port `11470` is the streaming-server API, not a normal website root. Opening:

```text
http://192.168.0.10:11470/
```

may correctly return:

```json
{"detail":"Not Found"}
```

The health endpoint is:

```bash
curl -fsS http://192.168.0.10:11470/health
```

The Docker healthcheck uses the same endpoint internally at `127.0.0.1:11470/health`.

## BitTorrent Port Validation

Port `6882` is a BitTorrent peer port, not HTTP. Opening it in a browser can produce `ERR_EMPTY_RESPONSE`; that does not indicate a failure.

Validate the published sockets on the host instead:

```bash
sudo ss -lntup | grep -E ':6882\b'
```

Both TCP and UDP should be present after startup.

Inbound router forwarding is deliberately deferred to Phase 6. Do not disturb the existing qBittorrent `6881/TCP+UDP` forwarding when `6882` is added later.

## Trusted HTTPS Client Endpoint

The deployment sets:

```text
IPADDRESS=192.168.0.10
```

On startup the image obtains a trusted certificate and generates an HTTPS hostname under:

```text
*.stremio.rocks:12470
```

The exact generated hostname is stored in logs and `httpsCert.json`. The currently validated deployment generated:

```text
https://192-168-0-10.519b6502d940.stremio.rocks:12470/
```

Treat the generated hostname as instance state rather than hard-coding it into automation. If the certificate identity changes after a rebuild, use the hostname reported by the running server.

Validate TLS without disabling certificate verification:

```bash
curl -sS -o /dev/null -w 'HTTP %{http_code}\n' \
  https://192-168-0-10.519b6502d940.stremio.rocks:12470/
```

Validated response:

```text
HTTP 200
```

The same endpoint successfully loaded the Stremio web interface in a browser.

## Why The Media Path Does Not Use Nginx Proxy Manager

AIOStreams uses the normal internal AdGuard + NPM pattern because it is a control/configuration service.

The streaming server uses the upstream trusted `*.stremio.rocks:12470` endpoint instead because:

- native Stremio clients require trusted HTTPS in some environments;
- the upstream service already supplies a trusted certificate flow;
- direct streaming avoids an unnecessary reverse-proxy hop for large media transfers;
- Range requests and seeking can be validated directly against the streaming server.

Do not add NPM to the media path unless a later requirement demonstrates a concrete need and Range/seek behavior is retested.

## Persistent State Validation

After startup:

```bash
ls -lah /mnt/data/stremio-libtorrent-server
```

Validated persistent state includes:

```text
.resume/
.evictor-owner
certificates.pem
httpsCert.json
```

The torrent cache will populate this root during playback.

Because `/mnt/data` is an NTFS/fuse mount, displayed Unix mode bits may appear permissive. The certificate PEM includes private-key material, so Samba/share exposure of this path must be reviewed before final architecture acceptance.

## Docker Operations

```bash
cd /srv/docker/stremio-libtorrent-server
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
docker compose logs -f stremio-libtorrent-server
```

Resources:

```bash
docker stats stremio-libtorrent-server
```

## Updating

The image is pinned through:

```text
STREMIO_LIBTORRENT_VERSION=1.6.15
```

Before changing versions:

1. check the current stable upstream release;
2. review release notes for environment or API changes;
3. back up `/mnt/data/stremio-libtorrent-server`;
4. update only the version variable;
5. validate the resolved image;
6. pull and recreate;
7. revalidate runtime settings and trusted HTTPS.

Commands:

```bash
cd /srv/docker/stremio-libtorrent-server
docker compose config -q
docker compose config --images
docker compose pull
docker compose up -d
docker compose ps
docker compose logs --tail=150 stremio-libtorrent-server
```

After every upgrade, re-run the runtime environment check and verify that the generated trusted HTTPS endpoint still returns `HTTP 200`.

## Backup

Persistent state and cache live under:

```text
/mnt/data/stremio-libtorrent-server
```

For a consistent stopped-container backup:

```bash
cd /srv/docker/stremio-libtorrent-server
docker compose down
sudo tar -czf stremio-libtorrent-server-backup.tar.gz \
  /mnt/data/stremio-libtorrent-server
docker compose up -d
```

The backup contains `certificates.pem` and therefore private-key material. Store it as a secret-bearing backup, not in Git or a public share.

## Security Notes

- Only BitTorrent peer port `6882/TCP+UDP` is a candidate for public inbound forwarding.
- `8081`, `11470`, and `12470` are bound to the LAN IP rather than `0.0.0.0` in Compose.
- The trusted `stremio.rocks` HTTPS endpoint resolves to the LAN IP used during certificate setup; it is intended for trusted client access, not general public service publishing.
- `certificates.pem` contains a private key and must never be committed.
- `/mnt/data` is NTFS/fuse; review Samba exposure of `/mnt/data/stremio-libtorrent-server` before final acceptance.
- qBittorrent remains unchanged on `6881/TCP+UDP`.
- No debrid service is configured.
- No GPU/transcoding overlay is configured in v1.

## Troubleshooting

### `6882` Shows An Empty Browser Response

Expected. It is not an HTTP service. Validate sockets with `ss` and later validate peer connectivity using BitTorrent-aware tests.

### `11470/` Returns `{"detail":"Not Found"}`

Expected for the API root. Test `/health` instead.

### Container Is Healthy But Cache Is Not Growing

Cache usage will not grow until actual torrent playback begins. Phase 7 validates growth using a legitimate high-bitrate source.

### Trusted HTTPS Fails

Check startup logs for certificate acquisition and inspect:

```bash
ls -lah /mnt/data/stremio-libtorrent-server
cat /mnt/data/stremio-libtorrent-server/httpsCert.json
```

Do not paste private key contents from `certificates.pem` into tickets, chats, or Git.

### qBittorrent Port Conflict

Confirm the new service is configured for `6882` both inside and outside the container:

```bash
docker exec stremio-libtorrent-server env | grep STREMIOSRV_BT_LISTEN_PORT
sudo ss -lntup | grep -E ':(6881|6882)\b'
```

qBittorrent must remain on `6881`; Stremio must remain on `6882`.

## Operational Health Checklist

```bash
cd /srv/docker/stremio-libtorrent-server
docker compose ps
docker compose logs --tail=100 stremio-libtorrent-server
docker exec stremio-libtorrent-server env | grep -E '^STREMIOSRV_(BT_LISTEN_PORT|CACHE_SIZE|READAHEAD_BYTES|DOWNLOAD_RATE_LIMIT|ADAPTIVE_PICKING)='
curl -fsS http://192.168.0.10:11470/health
sudo ss -lntup | grep -E ':6882\b'
ls -lah /mnt/data/stremio-libtorrent-server
```

For the client-facing TLS check, use the generated `*.stremio.rocks:12470` hostname reported by the current instance and verify it without `-k`.

## Implementation Status

Completed in Phase 3:

- pinned `androshack/stremio-libtorrent-server:1.6.15` deployment;
- persistent state/cache under `/mnt/data/stremio-libtorrent-server`;
- `10GiB` read-ahead;
- `300GiB` cache budget;
- unlimited torrent download rate;
- experimental adaptive picking disabled;
- `6882/TCP+UDP` peer port without disturbing qBittorrent `6881`;
- LAN-only web/API bindings;
- trusted `*.stremio.rocks:12470` HTTPS method;
- healthy-container, runtime-environment, persistence, and TLS validation.

Still pending in later phases:

- AIOStreams Torrentio/source configuration;
- primary TV client streaming-server configuration;
- end-to-end proof that selected torrents are downloaded by the Ubuntu server;
- router forwarding and inbound-peer validation for `6882/TCP+UDP`;
- large-file read-ahead, seek, and resilience testing.
