# Stremio + AIOStreams Implemented Architecture

Status: Partially implemented — Phases 1-3 complete  
Purpose: Record the as-built Stremio streaming architecture after host preflight, AIOStreams deployment, and central torrent-engine deployment.  
Depends on: [Current state](./current-state.md), [Docker and Portainer](../platform/docker-and-portainer.md), [Networking and reverse proxy](../platform/networking-and-reverse-proxy.md), [Storage and Samba](../platform/storage-and-samba.md)  
Related docs: [Original architecture roadmap](../roadmaps/stremio-aiostreams/README.md), [AIOStreams runbook](../services/aiostreams/README.md), [stremio-libtorrent-server runbook](../services/stremio-libtorrent-server/README.md), [qBittorrent](../services/qbittorrent/README.md)

## Implementation Scope

The original roadmap defined a seven-phase free/self-hosted Stremio rebuild. This document records the live server state after completion of:

1. Phase 1 — preflight;
2. Phase 2 — self-hosted AIOStreams deployment;
3. Phase 3 — `stremio-libtorrent-server` deployment.

Phases 4-7 remain pending. In particular, Torrentio/source configuration, TV client cutover, router forwarding for the new peer port, and large-file resilience testing have not yet been declared complete.

## Locked Architecture Preserved

The deployed state still follows the original locked decisions:

- free/self-hosted components only;
- AIOStreams self-hosted on the Ubuntu server;
- ordinary P2P BitTorrent, with no paid debrid backend;
- `stremio-libtorrent-server` as the central torrent/streaming engine;
- `10GiB` read-ahead target;
- `300GiB` persistent torrent cache;
- qBittorrent unchanged on `6881/TCP+UDP`;
- Stremio torrent engine on `6882/TCP+UDP`;
- no TorBox, Real-Debrid, Premiumize, StremThru, or MediaFlow in v1;
- no GPU/transcoding path in normal v1 operation;
- Docker stacks under `/srv/docker/<service>`;
- explicit AdGuard local DNS rewrites rather than wildcard local DNS;
- Nginx Proxy Manager for the AIOStreams control/configuration endpoint;
- direct trusted HTTPS from `stremio-libtorrent-server` for the media path.

## As-Built High-Level Architecture

```text
                         DISCOVERY / CONTROL PATH
                         ========================

Stremio client
     |
     | addon request
     v
https://aio.pirocorp.com
     |
     v
AdGuard Home explicit rewrite
     |
     | aio.pirocorp.com -> 192.168.0.10
     v
Nginx Proxy Manager :443
     |
     | HTTP -> 192.168.0.10:3001
     v
AIOStreams v2.34.1
     |
     +--> Torrentio P2P / additional verified free sources  [Phase 4 pending]
     |
     v
normalized torrent-capable stream results
     |
     v
Stremio client


                            MEDIA / DATA PATH
                            =================

selected torrent infohash / file
     |
     v
stremio-libtorrent-server 1.6.15
     |
     +--> DHT / trackers / PEX / peers
     |       |
     |       +--> 6882/TCP+UDP
     |
     +--> 10 GiB read-ahead
     +--> 300 GiB persistent cache
     |       |
     |       +--> /mnt/data/stremio-libtorrent-server
     |
     v
trusted *.stremio.rocks HTTPS :12470
     |
     v
Stremio player
```

The key separation is unchanged: **AIOStreams is not the media proxy**. It is the source aggregation/control plane. The selected torrent is handled by `stremio-libtorrent-server`, which joins the swarm and serves the media bytes.

## Phase 1 — Preflight Results

### Host

| Item | Validated value |
| --- | --- |
| Architecture | `x86_64` |
| CPU | Intel Xeon E3-1231 v3 @ 3.40 GHz |
| CPU topology | 4 cores / 8 threads |
| RAM | 30 GiB total |
| Swap | 8 GiB |
| GPU | NVIDIA GeForce GT 730 |
| LAN interface | `enp2s0` |
| LAN address | `192.168.0.10/24` |
| Transcoding | explicitly not required for v1 |

The published Docker images are compatible with the host architecture. No GPU passthrough was required because direct play is the intended path.

### Storage Selection

`/mnt/data` was selected for the torrent cache.

At preflight:

```text
Device:      /dev/sdc2
Mount:       /mnt/data
Filesystem:  fuseblk / NTFS
Size:        ~7.3 TiB
Available:   ~4.1 TiB
```

The `300GiB` cache budget therefore had substantial free-space headroom.

### Existing Torrent Port

qBittorrent was already publishing:

```text
6881/TCP
6881/UDP
```

This confirmed the architectural need for a separate peer port:

```text
stremio-libtorrent-server -> 6882/TCP+UDP
```

### Docker And Reverse Proxy Baseline

Nginx Proxy Manager was already running on:

```text
Network: nginx-proxy-manager_default
Subnet:  172.20.0.0/16
NPM IP:  172.20.0.2
Gateway: 172.20.0.1
```

The existing wildcard certificate covers:

```text
pirocorp.com
*.pirocorp.com
```

AdGuard Home already used explicit per-host DNS rewrites, so `aio.pirocorp.com` was added in the same pattern.

### Ports Reserved

The following were checked and free before deployment:

```text
3001/tcp       AIOStreams host port
8081/tcp       Stremio web UI host port
11470/tcp      streaming API
12470/tcp      trusted HTTPS
6882/tcp+udp   Stremio BitTorrent peer port
```

Host port `8080` was not reused because qBittorrent already owns it.

## Phase 2 — AIOStreams As Built

| Item | Deployed value |
| --- | --- |
| Version | `v2.34.1` |
| Image | `ghcr.io/viren070/aiostreams:v2.34.1` |
| Stack root | `/srv/docker/aiostreams` |
| Persistent state | `/srv/docker/aiostreams/data` |
| Host binding | `192.168.0.10:3001 -> 3000/tcp` |
| Preferred URL | `https://aio.pirocorp.com` |
| DNS | explicit AdGuard rewrite to `192.168.0.10` |
| HTTPS | existing wildcard cert through NPM |
| Database | SQLite under `/app/data` |
| Health | validated healthy |

Validated application state includes:

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

Validated HTTP paths:

```text
http://192.168.0.10:3001/stremio/configure      -> HTTP 200
https://aio.pirocorp.com/stremio/configure      -> HTTP 200
```

The NPM container was also tested directly against `192.168.0.10:3001` and returned HTTP 200, so no additional UFW rule was added for this service.

A public DNS check through Cloudflare DNS-over-HTTPS returned NXDOMAIN for `aio.pirocorp.com`, confirming that the current hostname is supplied by local AdGuard DNS rather than a public `A` record.

See the [AIOStreams runbook](../services/aiostreams/README.md) for the sanitized `.env`, Compose file, exact installation sequence, NPM settings, backup, and troubleshooting.

## Phase 3 — Central Torrent Engine As Built

| Item | Deployed value |
| --- | --- |
| Version | `1.6.15` |
| Image | `androshack/stremio-libtorrent-server:1.6.15` |
| Stack root | `/srv/docker/stremio-libtorrent-server` |
| Persistent state/cache | `/mnt/data/stremio-libtorrent-server` |
| Read-ahead | `10GiB` |
| Cache budget | `300GiB` |
| Download rate | unlimited (`0`) |
| Adaptive picking | `false` |
| Peer port | `6882/TCP+UDP` |
| Web UI | `192.168.0.10:8081 -> 8080` |
| API | `192.168.0.10:11470 -> 11470` |
| Trusted HTTPS | `192.168.0.10:12470 -> 12470` |
| Health | validated healthy |
| GPU/transcoding overlay | not configured |

The running container environment was checked directly and confirmed:

```text
STREMIOSRV_READAHEAD_BYTES=10GiB
STREMIOSRV_DOWNLOAD_RATE_LIMIT=0
STREMIOSRV_ADAPTIVE_PICKING=false
STREMIOSRV_BT_LISTEN_PORT=6882
STREMIOSRV_CACHE_SIZE=300GiB
```

Persistent state was confirmed on the host:

```text
/mnt/data/stremio-libtorrent-server/.resume/
/mnt/data/stremio-libtorrent-server/.evictor-owner
/mnt/data/stremio-libtorrent-server/certificates.pem
/mnt/data/stremio-libtorrent-server/httpsCert.json
```

The first startup successfully obtained a trusted `*.stremio.rocks` certificate for the LAN-IP-based endpoint. The generated HTTPS URL returned HTTP 200 with normal certificate verification and loaded the Stremio web interface.

The current generated endpoint is recorded in the service runbook, but clients should use the hostname reported by the running instance rather than assuming the instance-specific suffix can never change.

See the [stremio-libtorrent-server runbook](../services/stremio-libtorrent-server/README.md) for the exact Compose file, `.env`, validation commands, TLS flow, operations, backup, and troubleshooting.

## Port Allocation

| Service | Port | Exposure model |
| --- | --- | --- |
| qBittorrent peer traffic | `6881/TCP+UDP` | existing assignment, unchanged |
| AIOStreams | `192.168.0.10:3001/tcp` | LAN host binding; HTTPS via NPM |
| Stremio web UI | `192.168.0.10:8081/tcp` | LAN host binding |
| Stremio API | `192.168.0.10:11470/tcp` | LAN host binding |
| Stremio trusted HTTPS | `192.168.0.10:12470/tcp` | LAN host binding; trusted `stremio.rocks` hostname |
| Stremio BitTorrent | `6882/TCP+UDP` | peer port; router forwarding pending Phase 6 |

Docker may show `6881/tcp` in the Stremio image metadata. That is not a host binding. The live peer binding is `6882`.

## Why Two Different HTTPS Publishing Models

### AIOStreams

AIOStreams follows the normal homelab application pattern:

```text
AdGuard -> Nginx Proxy Manager -> AIOStreams
```

This is appropriate because AIOStreams is a configuration/control service with ordinary web traffic.

### Streaming Server

The media server uses:

```text
Stremio client -> trusted *.stremio.rocks:12470 -> stremio-libtorrent-server
```

This was selected because the upstream service provides trusted client HTTPS directly, and it avoids adding NPM as another hop for large video transfers. Range/seek behavior will be validated in the end-to-end client and resilience phases.

## Security Boundaries

### Do Not Commit

Never commit:

- the real AIOStreams `SECRET_KEY`;
- generated private AIOStreams configuration URLs/tokens;
- `certificates.pem` from the streaming-server data directory;
- Stremio account/session tokens;
- future private addon credentials.

### Network Exposure

Only the BitTorrent peer port is a candidate for explicit public inbound router forwarding:

```text
6882/TCP
6882/UDP
```

Do not intentionally forward the following as public management endpoints:

```text
3001/tcp
8081/tcp
11470/tcp
12470/tcp
```

### NTFS Permission Caveat

`/mnt/data` is mounted through NTFS/fuse. Unix mode bits shown by `ls` may appear broadly permissive because permissions are governed by mount options. Since `certificates.pem` includes a private key, verify that `/mnt/data/stremio-libtorrent-server` is not unintentionally available through Samba or another share before final acceptance.

## Rebuild Sequence From Zero

For a clean rebuild on the existing homelab platform:

1. run the Phase 1 checks for architecture, CPU/RAM, storage, ports, Docker networks, AdGuard/NPM, and qBittorrent `6881`;
2. deploy AIOStreams using the [AIOStreams runbook](../services/aiostreams/README.md);
3. generate a new AIOStreams secret locally and keep it out of Git;
4. validate direct `3001` access, AdGuard resolution, NPM backend connectivity, and final HTTPS;
5. deploy `stremio-libtorrent-server` using the [streaming-server runbook](../services/stremio-libtorrent-server/README.md);
6. bind the persistent cache to `/mnt/data/stremio-libtorrent-server`;
7. validate runtime values rather than trusting only `.env`;
8. validate trusted `*.stremio.rocks:12470` HTTPS without `-k`;
9. continue with Phase 4 source configuration;
10. complete TV/client cutover, router peer-port validation, and large-file resilience testing before marking the full v1 architecture complete.

## Current Acceptance Matrix

| Check | Status |
| --- | --- |
| Host is compatible `x86_64` | PASS |
| Sufficient persistent storage for `300GiB` cache | PASS |
| qBittorrent remains on `6881/TCP+UDP` | PASS |
| AIOStreams self-hosted and healthy | PASS |
| `https://aio.pirocorp.com` through AdGuard + NPM | PASS |
| AIOStreams persistent state | PASS |
| `stremio-libtorrent-server` self-hosted and healthy | PASS |
| `10GiB` runtime read-ahead | PASS |
| `300GiB` runtime cache budget | PASS |
| `6882/TCP+UDP` runtime peer port | PASS |
| Trusted client HTTPS method | PASS |
| Torrentio P2P source configured through AIOStreams | PENDING — Phase 4 |
| Primary TV uses the central streaming server | PENDING — Phase 5 |
| Router forwarding/inbound peers on `6882` | PENDING — Phase 6 |
| Large-file seek/buffer resilience test | PENDING — Phase 7 |
| Proof that client no longer joins the swarm directly | PENDING — Phases 5/7 |

## Next Step

Continue with Phase 4 only after this as-built baseline is preserved: configure Torrentio in AIOStreams in P2P/non-debrid mode, then verify that returned results contain torrent information usable by the central streaming server.
