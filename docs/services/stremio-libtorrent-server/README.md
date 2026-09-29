# stremio-libtorrent-server Deployment And Operations Runbook

Status: Implemented  
Purpose: Document the deployed central Stremio BitTorrent streaming engine, persistent cache, trusted HTTPS endpoint, and operational validation.  
Depends on: [Docker and Portainer](../../platform/docker-and-portainer.md), [Storage and Samba](../../platform/storage-and-samba.md), [Networking and reverse proxy](../../platform/networking-and-reverse-proxy.md)  
Related docs: [Services index](../README.md), [Service inventory](../../overview/service-inventory.md), [Stremio + AIOStreams architecture roadmap](../../roadmaps/stremio-aiostreams/README.md), [Phase 4 completion](../../roadmaps/stremio-aiostreams/phase-4-completion.md), [AIOStreams](../aiostreams/README.md), [qBittorrent](../qbittorrent/README.md)

## Current Deployment

`stremio-libtorrent-server` is the central torrent playback engine for the Stremio architecture. It joins BitTorrent swarms, prioritizes torrent pieces around the playhead, maintains server-side read-ahead and persistent cache data, and serves the resulting media stream to Stremio clients.

| Item | Current value |
| --- | --- |
| Version | `1.6.15` |
| Image | `androshack/stremio-libtorrent-server:1.6.15` |
| Host | `piroman-server` / `192.168.0.10` |
| Stack root | `/srv/docker/stremio-libtorrent-server` |
| Persistent data/cache | `/mnt/data/stremio-libtorrent-server` |
| Cache filesystem | `/mnt/data` on NTFS/fuseblk |
| Cache budget | `100GiB` |
| Read-ahead target | `10GiB` |
| Cache eviction grace | upstream default `1800s` / 30 minutes |
| Download rate limit | `0` / unlimited |
| Adaptive picking | disabled |
| BitTorrent listen port | `6882/TCP+UDP` |
| Web UI host port | `192.168.0.10:8081 -> 8080/tcp` |
| Streaming API | `192.168.0.10:11470 -> 11470/tcp` |
| Trusted HTTPS | `192.168.0.10:12470 -> 12470/tcp` |
| Client TLS method | upstream trusted `*.stremio.rocks` certificate generated from `IPADDRESS=192.168.0.10` |
| Transcoding | not part of the normal v1 path; no GPU overlay configured |
| Router forwarding for `6882` | enabled to `192.168.0.10:6882`; TCP and UDP operationally validated |

The container is healthy and the trusted HTTPS endpoint has been validated with normal certificate verification, without `curl -k`.

## Architecture

```text
Stremio client
      |
      | selected torrent infoHash / fileIdx
      v
stremio-libtorrent-server
      |
      +--> DHT / trackers / PEX / peers
      |       |
      |       +--> Internet TCP+UDP :6882
      |              |
      |              +--> router -> 192.168.0.10:6882
      |
      +--> 10 GiB playhead read-ahead target
      +--> 100 GiB persistent cache budget
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

AIOStreams remains separate from this path. It provides source discovery and result normalization; it does not proxy selected media bytes.

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

The application itself is configured to listen on `6882`; do not map host `6882` to container `6881`.

## Storage

`/mnt/data` was selected for the persistent cache. At preflight it had approximately 4.1 TiB available, providing substantial headroom for the current `100GiB` cache budget.

Layout:

```text
/srv/docker/stremio-libtorrent-server
├── .env
└── compose.yaml

/mnt/data/stremio-libtorrent-server
├── .resume/
├── .evictor-owner
├── certificates.pem
├── httpsCert.json
├── .<infohash>.parts
└── <torrent-name>/
```

The persistent bind mount is:

```text
/mnt/data/stremio-libtorrent-server -> /root/.stremio-server
```

`certificates.pem` contains private key material. Never commit it to Git or copy it into unprotected documentation artifacts.

## Installation

### Create Directories

```bash
sudo mkdir -p /srv/docker/stremio-libtorrent-server
sudo chown -R piroman:piroman /srv/docker/stremio-libtorrent-server
sudo mkdir -p /mnt/data/stremio-libtorrent-server
```

### `.env`

Current sanitized configuration:

```env
STREMIO_LIBTORRENT_VERSION=1.6.15
STREMIOSRV_BT_LISTEN_PORT=6882
STREMIOSRV_CACHE_SIZE=100GiB
STREMIOSRV_READAHEAD_BYTES=10GiB
STREMIOSRV_DOWNLOAD_RATE_LIMIT=0
STREMIOSRV_ADAPTIVE_PICKING=false
STREMIO_DATA_DIR=/mnt/data/stremio-libtorrent-server
IPADDRESS=192.168.0.10
```

`STREMIOSRV_CACHE_EVICT_GRACE` is not explicitly set, so upstream default `1800` seconds / 30 minutes is used.

Restrict the environment file:

```bash
chmod 600 /srv/docker/stremio-libtorrent-server/.env
```

### `compose.yaml`

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

## Start And Validate

```bash
cd /srv/docker/stremio-libtorrent-server
docker compose config -q
docker compose config --images
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

Docker may also display `6881/tcp` as exposed image metadata. That is not a published host binding and does not conflict with qBittorrent.

## Runtime Configuration Check

Do not rely only on `.env`; verify the running container:

```bash
docker compose exec stremio-libtorrent-server env | \
  grep -E '^STREMIOSRV_(BT_LISTEN_PORT|CACHE_SIZE|READAHEAD_BYTES|DOWNLOAD_RATE_LIMIT|ADAPTIVE_PICKING)='
```

Expected:

```text
STREMIOSRV_READAHEAD_BYTES=10GiB
STREMIOSRV_DOWNLOAD_RATE_LIMIT=0
STREMIOSRV_ADAPTIVE_PICKING=false
STREMIOSRV_BT_LISTEN_PORT=6882
STREMIOSRV_CACHE_SIZE=100GiB
```

## HTTP And Health Validation

Port `11470` is the streaming-server API. The root may correctly return `{"detail":"Not Found"}`.

Use:

```bash
curl -fsS http://192.168.0.10:11470/health
```

The Docker healthcheck uses the same endpoint internally at `127.0.0.1:11470/health`.

## BitTorrent Port Validation

Port `6882` is BitTorrent peer traffic, not HTTP.

Validate host sockets:

```bash
sudo ss -lntup | grep -E ':6882\b'
```

Both TCP and UDP should be present.

The router explicitly forwards `6882/TCP+UDP` to `192.168.0.10:6882`. Docker DNAT/forwarding rules for both protocols are validated, public inbound TCP was independently confirmed open, and live bidirectional Internet UDP peer traffic has been observed. qBittorrent remains unchanged on `6881/TCP+UDP`.

## Trusted HTTPS Client Endpoint

The deployment sets:

```text
IPADDRESS=192.168.0.10
```

On startup the image obtains a trusted certificate and generates an HTTPS hostname under:

```text
*.stremio.rocks:12470
```

The exact hostname is stored in logs and `httpsCert.json`. Treat it as instance state rather than hard-coding the suffix into automation.

Validate TLS without disabling certificate verification:

```bash
curl -sS -o /dev/null -w 'HTTP %{http_code}\n' \
  https://<generated-host>.stremio.rocks:12470/
```

Validated response:

```text
HTTP 200
```

## End-To-End Playback Proof

Phase 4 established that normal AIOStreams/Torrentio selections reach this server.

During desktop playback of `Mayday`, the following changed under `/mnt/data/stremio-libtorrent-server`:

- the selected media file;
- `.resume/<infohash>.fastresume`;
- `.resume/index.json`;
- the matching `.<infohash>.parts` file;
- `.evictor-owner`.

This proves that the Ubuntu server, rather than the desktop Stremio client, downloaded and cached the selected torrent.

Initial primary-TV playback through the same central path was also successful.

## Cache Behaviour

### `10GiB` Is Read-Ahead, Not A Download Cap

`STREMIOSRV_READAHEAD_BYTES=10GiB` is the high-priority region around/ahead of the playhead. It does not limit total downloaded bytes.

Upstream continues filling the wanted file after playback closes. Therefore a partially watched 20-60+ GiB movie can continue downloading until complete.

A custom fork to pause-on-close was rejected to avoid maintaining patched images across upstream releases.

### Current Upstream-Only Policy

```text
Read-ahead:       10GiB
Cache budget:     100GiB
Eviction grace:   1800s / 30 min upstream default
Download rate:    unlimited
Adaptive picking: false
```

The cache budget is not a hard quota. Active/protected data can temporarily push actual disk usage above `100GiB` before a background eviction pass runs.

`CACHE_EVICT_GRACE` is not a TTL. It prevents recently served/modified entries from being selected; cleanup is still triggered by the cache exceeding its size budget.

### Live Eviction Validation

During real 4K playback, observed media usage reached approximately:

```text
The Whisper Man       21G
The End Of Oak Street 22G
Mayday                24G
In The Grey           36G  (active)
-----------------------------------
Total                 ~101G
```

The cache then evicted the older `Mayday` entry while keeping the active `In The Grey` entry, reducing total usage to approximately `78G`.

This validates the current `100GiB` upstream-only cache policy.

If routine use begins to include single REMUX files larger than `100GiB`, consider increasing the budget because upstream cache guidance expects the budget to exceed the largest normal file.

## Cache Monitoring

Total real disk usage:

```bash
du -sh /mnt/data/stremio-libtorrent-server
```

Real allocated size file-by-file, including sparse media files:

```bash
find /mnt/data/stremio-libtorrent-server -type f -exec du -h {} + | sort -h
```

Watch media files and total cache every 10 seconds:

```bash
watch -n 10 'du -sh /mnt/data/stremio-libtorrent-server; echo; find /mnt/data/stremio-libtorrent-server -type f \( -name "*.mkv" -o -name "*.mp4" \) -exec du -h {} + | sort -h'
```

Recent cache/eviction logs:

```bash
docker logs --since 5m stremio-libtorrent-server 2>&1 | grep -Ei 'evict|cache'
```

Do not use apparent file length alone for partially downloaded torrent files: libtorrent can create sparse files whose `st_size` reflects final size while `du` reflects actual allocated disk blocks.

## Docker Operations

```bash
cd /srv/docker/stremio-libtorrent-server
```

Start/recreate:

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
3. back up `/mnt/data/stremio-libtorrent-server` as appropriate;
4. update only the version variable;
5. validate the resolved image;
6. pull and recreate;
7. revalidate runtime settings, trusted HTTPS, cache policy, and playback.

The deployment intentionally avoids a custom fork/image so normal upstream upgrades do not require carrying local code patches.

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

- The only Stremio port intentionally forwarded for public inbound peer traffic is `6882/TCP+UDP`.
- `8081`, `11470`, and `12470` are bound to the LAN IP rather than `0.0.0.0` in Compose and are not part of the general public-forward scope.
- The trusted `stremio.rocks` HTTPS endpoint resolves to the LAN IP used during certificate setup; it is intended for trusted client access, not general public service publishing.
- `certificates.pem` contains a private key and must never be committed.
- `/mnt/data` is NTFS/fuse; review Samba exposure of `/mnt/data/stremio-libtorrent-server` before final acceptance.
- qBittorrent remains unchanged on `6881/TCP+UDP`.
- No debrid service is configured.
- No GPU/transcoding overlay is configured in v1.

## Troubleshooting

### `6882` Shows An Empty Browser Response

Expected. It is not an HTTP service. Validate sockets with `ss`.

### `11470/` Returns `{"detail":"Not Found"}`

Expected for the API root. Test `/health` instead.

### Container Is Healthy But Cache Is Not Growing

Cache usage grows only after actual torrent playback starts. Confirm that AIOStreams returns P2P torrent entries and inspect media/cache files during playback.

### Cache Exceeds `100GiB`

Expected temporarily when active/protected data pushes usage above the configured budget. The background evictor removes eligible older entries on subsequent passes.

### Trusted HTTPS Fails

Check startup logs and inspect:

```bash
ls -lah /mnt/data/stremio-libtorrent-server
cat /mnt/data/stremio-libtorrent-server/httpsCert.json
```

Do not paste private-key contents from `certificates.pem` into tickets, chats, or Git.

### qBittorrent Port Conflict

Confirm the new service uses `6882` both inside and outside the container:

```bash
docker compose exec stremio-libtorrent-server env | grep STREMIOSRV_BT_LISTEN_PORT
sudo ss -lntup | grep -E ':(6881|6882)\b'
```

qBittorrent must remain on `6881`; Stremio must remain on `6882`.

## Operational Health Checklist

```bash
cd /srv/docker/stremio-libtorrent-server
docker compose ps
docker compose logs --tail=100 stremio-libtorrent-server
docker compose exec stremio-libtorrent-server env | grep -E '^STREMIOSRV_(BT_LISTEN_PORT|CACHE_SIZE|READAHEAD_BYTES|DOWNLOAD_RATE_LIMIT|ADAPTIVE_PICKING)='
curl -fsS http://192.168.0.10:11470/health
sudo ss -lntup | grep -E ':6882\b'
du -sh /mnt/data/stremio-libtorrent-server
```

For the client-facing TLS check, use the generated `*.stremio.rocks:12470` hostname reported by the current instance and verify it without `-k`.

## Implementation Status

The original seven-phase Stremio/AIOStreams baseline is complete.

Completed:

- pinned `androshack/stremio-libtorrent-server:1.6.15` deployment;
- persistent state/cache under `/mnt/data/stremio-libtorrent-server`;
- `10GiB` read-ahead;
- `100GiB` cache budget;
- upstream default 30-minute eviction grace;
- unlimited torrent download rate;
- experimental adaptive picking disabled;
- `6882/TCP+UDP` peer port without disturbing qBittorrent `6881`;
- LAN-only web/API bindings;
- trusted `*.stremio.rocks:12470` HTTPS method;
- healthy-container, runtime-environment, persistence, and TLS validation;
- end-to-end proof that Stremio-selected torrents are downloaded by the Ubuntu server;
- real 4K cache growth and LRU eviction validation;
- full primary-TV/client acceptance including seek/resume and sustained playback;
- explicit router forwarding for `6882/TCP+UDP`, with TCP and UDP operationally validated;
- formal large-file read-ahead, seek, cache/LRU, and resilience acceptance.

No baseline implementation phases remain pending. The next planned Stremio work is the separate [AIOStreams multi-source P2P expansion](../../roadmaps/stremio-aiostreams/multi-source-p2p-expansion.md), which changes only the source layer and preserves this playback engine.
