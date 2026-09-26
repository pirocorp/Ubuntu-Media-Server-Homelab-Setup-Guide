# Stremio + AIOStreams Implemented Architecture

Status: Partially implemented — Phases 1-4 complete  
Purpose: Record the as-built Stremio streaming architecture after host preflight, AIOStreams deployment, central torrent-engine deployment, and source/end-to-end P2P validation.  
Depends on: [Current state](./current-state.md), [Docker and Portainer](../platform/docker-and-portainer.md), [Networking and reverse proxy](../platform/networking-and-reverse-proxy.md), [Storage and Samba](../platform/storage-and-samba.md)  
Related docs: [Original architecture roadmap](../roadmaps/stremio-aiostreams/README.md), [Phase 4 completion](../roadmaps/stremio-aiostreams/phase-4-completion.md), [AIOStreams runbook](../services/aiostreams/README.md), [stremio-libtorrent-server runbook](../services/stremio-libtorrent-server/README.md), [qBittorrent](../services/qbittorrent/README.md)

## Implementation Scope

The original roadmap defined a seven-phase free/self-hosted Stremio rebuild. This document records the live server state after completion of:

1. Phase 1 — preflight;
2. Phase 2 — self-hosted AIOStreams deployment;
3. Phase 3 — `stremio-libtorrent-server` deployment;
4. Phase 4 — Torrentio P2P/non-debrid source configuration and end-to-end torrent validation.

Phase 5 has started with successful primary-TV playback through the central path, but full client acceptance is still pending. Phases 6-7 remain pending for router peer-port validation and formal large-file resilience testing.

## Locked Architecture Preserved

The deployed state still follows the original locked decisions, with one explicitly reopened cache-size decision:

- free/self-hosted components only;
- AIOStreams self-hosted on the Ubuntu server;
- ordinary P2P BitTorrent, with no paid debrid backend;
- `stremio-libtorrent-server` as the central torrent/streaming engine;
- `10GiB` read-ahead target;
- persistent torrent cache reduced from the original `300GiB` plan to `100GiB` after live validation;
- qBittorrent unchanged on `6881/TCP+UDP`;
- Stremio torrent engine on `6882/TCP+UDP`;
- no TorBox, Real-Debrid, Premiumize, StremThru, or MediaFlow in v1;
- no GPU/transcoding path in normal v1 operation;
- Docker stacks under `/srv/docker/<service>`;
- explicit AdGuard local DNS rewrites remain the normal LAN DNS model;
- Cloudflare DNS-only wildcard `*.pirocorp.com -> 192.168.0.10` is used as a compatibility fallback for clients that bypass local DNS;
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
     +--> local clients: AdGuard explicit rewrite
     |                    aio.pirocorp.com -> 192.168.0.10
     |
     +--> DNS-bypassing clients: public DNS-only wildcard
                          *.pirocorp.com -> 192.168.0.10
     |
     v
Nginx Proxy Manager :443
     |
     | HTTP -> 192.168.0.10:3001
     v
AIOStreams v2.34.1
     |
     +--> Torrentio P2P / non-debrid
     |
     v
normalized torrent-capable stream results
     |
     | infoHash + fileIdx
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
     +--> 100 GiB persistent cache
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

The current `100GiB` cache budget therefore has substantial free-space headroom.

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

AdGuard Home continues to use explicit per-host DNS rewrites for normal LAN operation.

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
| Local DNS | explicit AdGuard rewrite to `192.168.0.10` |
| Public DNS fallback | DNS-only `*.pirocorp.com -> 192.168.0.10` |
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

The original local-only DNS design was extended after the primary TV failed to resolve `aio.pirocorp.com` reliably even with a manual DNS server. Cloudflare now publishes a DNS-only wildcard record to the private LAN IP. Google DNS (`8.8.8.8`) was verified to return `192.168.0.10` for `aio.pirocorp.com`.

See the [AIOStreams runbook](../services/aiostreams/README.md) for the sanitized `.env`, Compose file, exact installation sequence, NPM settings, backup, and troubleshooting.

## Phase 3 — Central Torrent Engine As Built

| Item | Deployed value |
| --- | --- |
| Version | `1.6.15` |
| Image | `androshack/stremio-libtorrent-server:1.6.15` |
| Stack root | `/srv/docker/stremio-libtorrent-server` |
| Persistent state/cache | `/mnt/data/stremio-libtorrent-server` |
| Read-ahead | `10GiB` |
| Cache budget | `100GiB` |
| Cache eviction grace | upstream default `1800s` / 30 minutes |
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
STREMIOSRV_CACHE_SIZE=100GiB
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

## Phase 4 — Source And End-to-End Validation

Torrentio is configured inside AIOStreams in P2P/non-debrid mode. AIOStreams stream API responses were inspected directly and confirmed normal torrent results carrying usable `infoHash` and `fileIdx` data.

The AIOStreams addon was installed into the Stremio account. Optional third-party source addons were removed during troubleshooting because one or more polluted the normal movie/detail flow with deep-link style entries. The stable baseline retained Cinemeta, Local Files, and AIOStreams.

Desktop playback of `Mayday` created live media and resume-state updates under `/mnt/data/stremio-libtorrent-server`, proving the selected torrent was downloaded by the Ubuntu server rather than the desktop client.

The primary TV also successfully resolved AIOStreams through the DNS fallback and played P2P content through the central path. This initial TV success starts Phase 5, but full client acceptance is not yet declared complete.

See [Phase 4 completion](../roadmaps/stremio-aiostreams/phase-4-completion.md) for the detailed acceptance evidence.

## Cache Policy Reopened After Live Testing

The original architecture planned a `300GiB` cache. Live testing showed that `STREMIOSRV_READAHEAD_BYTES=10GiB` is only the high-priority playhead window; upstream continues filling the wanted file after playback closes.

A custom fork was rejected to avoid maintaining patched images across releases. The upstream-only policy was changed to:

```text
STREMIOSRV_READAHEAD_BYTES=10GiB
STREMIOSRV_CACHE_SIZE=100GiB
STREMIOSRV_CACHE_EVICT_GRACE=1800
```

The grace period is not a TTL. Cleanup remains size-triggered LRU eviction, and active/protected entries can temporarily push usage above the nominal budget.

During real 4K playback the cache reached approximately `101G`. The evictor removed the older `Mayday` cache entry (~24G) while preserving the active `In The Grey` stream, reducing total usage to approximately `78G`. This validates the current `100GiB` upstream-only policy.

If normal usage begins to include individual REMUX files larger than `100GiB`, revisit the budget because a cache larger than the largest expected file is safer.

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

AIOStreams follows the homelab application pattern:

```text
DNS -> Nginx Proxy Manager -> AIOStreams
```

Normal LAN clients resolve through explicit AdGuard rewrites. DNS-bypassing clients can resolve through the public DNS-only wildcard that points to the same private LAN IP.

### Streaming Server

The media server uses:

```text
Stremio client -> trusted *.stremio.rocks:12470 -> stremio-libtorrent-server
```

This was selected because the upstream service provides trusted client HTTPS directly, and it avoids adding NPM as another hop for large video transfers. Range/seek behavior remains part of the formal client/resilience acceptance work.

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

The public `*.pirocorp.com -> 192.168.0.10` DNS record exposes only a private RFC1918 destination and does not make the services Internet-routable.

### NTFS Permission Caveat

`/mnt/data` is mounted through NTFS/fuse. Unix mode bits shown by `ls` may appear broadly permissive because permissions are governed by mount options. Since `certificates.pem` includes a private key, verify that `/mnt/data/stremio-libtorrent-server` is not unintentionally available through Samba or another share before final acceptance.

## Rebuild Sequence From Zero

For a clean rebuild on the existing homelab platform:

1. run the Phase 1 checks for architecture, CPU/RAM, storage, ports, Docker networks, AdGuard/NPM, and qBittorrent `6881`;
2. deploy AIOStreams using the [AIOStreams runbook](../services/aiostreams/README.md);
3. generate a new AIOStreams secret locally and keep it out of Git;
4. validate direct `3001` access, DNS resolution, NPM backend connectivity, and final HTTPS;
5. deploy `stremio-libtorrent-server` using the [streaming-server runbook](../services/stremio-libtorrent-server/README.md);
6. bind the persistent cache to `/mnt/data/stremio-libtorrent-server`;
7. validate runtime values rather than trusting only `.env`;
8. validate trusted `*.stremio.rocks:12470` HTTPS without `-k`;
9. configure Torrentio in AIOStreams in P2P/non-debrid mode and install the AIOStreams addon into Stremio;
10. verify actual server-side cache activity during playback;
11. complete full TV/client acceptance, router peer-port validation, and formal large-file resilience testing before marking the full v1 architecture complete.

## Current Acceptance Matrix

| Check | Status |
| --- | --- |
| Host is compatible `x86_64` | PASS |
| Sufficient persistent storage for `100GiB` cache | PASS |
| qBittorrent remains on `6881/TCP+UDP` | PASS |
| AIOStreams self-hosted and healthy | PASS |
| `https://aio.pirocorp.com` through DNS + NPM | PASS |
| AIOStreams persistent state | PASS |
| `stremio-libtorrent-server` self-hosted and healthy | PASS |
| `10GiB` runtime read-ahead | PASS |
| `100GiB` runtime cache budget | PASS |
| LRU eviction above cache budget | PASS |
| `6882/TCP+UDP` runtime peer port | PASS |
| Trusted client HTTPS method | PASS |
| Torrentio P2P source configured through AIOStreams | PASS — Phase 4 |
| AIOStreams results contain usable infoHash/fileIdx data | PASS — Phase 4 |
| Proof that Ubuntu server downloads/caches selected torrent | PASS — Phase 4 |
| Primary TV reaches AIOStreams and plays through central path | PASS — initial Phase 5 validation |
| Full primary-TV/client acceptance | IN PROGRESS — Phase 5 |
| Router forwarding/inbound peers on `6882` | PENDING — Phase 6 |
| Formal large-file seek/buffer resilience test | PENDING — Phase 7 |

## Next Step

Phase 4 is closed. Continue with Phase 5 client acceptance while preserving the validated source and cache baseline. After that, validate router forwarding for `6882/TCP+UDP` and run the formal large-file resilience tests.
