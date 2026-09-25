# Stremio + AIOStreams Self-Hosted Streaming Architecture

Status: Planned
Purpose: Record the approved zero-subscription Stremio architecture before implementation begins.
Depends on: Existing Ubuntu Docker host, Nginx Proxy Manager, AdGuard Home, Stremio clients, and Internet connectivity for BitTorrent peer traffic
Related docs: [Roadmaps index](../README.md), [Current state](../../overview/current-state.md), [Docker and Portainer](../../platform/docker-and-portainer.md), [Networking and reverse proxy](../../platform/networking-and-reverse-proxy.md), [qBittorrent](../../services/qbittorrent/README.md)

## Locked Decision Record and Implementation Brief

**Status:** Approved architecture; implementation not yet started  
**Version:** 1.0  
**Date:** 2026-09-25  
**Planned implementation:** 2026-09-26

## 1. Purpose

The goal is to replace a collection of direct Stremio torrent-source addons with one centrally managed self-hosted stream aggregation layer while also moving BitTorrent playback away from the individual Stremio client and onto the Ubuntu homelab server.

The design has two primary outcomes:

1. AIOStreams provides one consistent discovery, filtering, sorting, and deduplication layer for free P2P-capable Stremio sources.
2. `stremio-libtorrent-server` becomes the central BitTorrent streaming engine and large read-ahead/cache layer so short-term Internet or swarm throughput drops are absorbed by the server instead of immediately causing playback stalls on a TV or PC.

This document preserves the architecture decisions already agreed. Items marked **LOCKED** are requirements for the first implementation and should not be redesigned unless explicitly reopened.

## 2. Hard constraints

The following constraints are **LOCKED**:

1. The architecture must use **free components only**.
2. The server-side components must be **self-hosted** on the existing Ubuntu homelab server.
3. No paid debrid service is part of v1.
4. No paid hosted AIOStreams service is part of v1.
5. TorBox, Real-Debrid, Premiumize, AllDebrid, EasyDebrid, and similar subscription backends are explicitly out of scope.
6. The design must work with ordinary P2P torrent streams supplied by Stremio-compatible addons.
7. The buffering solution must be server-side rather than relying only on the small cache/buffer of each individual client.
8. The initial read-ahead target is **10 GiB**.
9. The initial torrent disk cache budget is **300 GiB**.
10. Existing homelab conventions remain in force: Docker Compose stacks under `/srv/docker/<service>`, explicit AdGuard rewrites rather than wildcard DNS, and Nginx Proxy Manager for normal internal HTTPS publishing.

## 3. Existing environment

The implementation must fit the existing homelab rather than create a parallel platform model.

Current assumptions:

- Ubuntu host: `piroman-server`
- LAN IP: `192.168.0.10`
- Docker / Docker Compose are already deployed
- service stacks normally live under `/srv/docker/<service>`
- Nginx Proxy Manager is the existing reverse proxy
- AdGuard Home provides explicit local DNS rewrites
- wildcard DNS rewrites are intentionally not used
- the existing certificate covers `pirocorp.com` and `*.pirocorp.com`
- Tailscale provides trusted remote access to the LAN
- qBittorrent is already deployed and already uses BitTorrent peer port `6881/TCP` and `6881/UDP`

The existing qBittorrent `6881` assignment creates a hard port-conflict requirement for the new torrent engine. `stremio-libtorrent-server` must therefore use a different BitTorrent listen port.

## 4. Locked high-level architecture

```text
                          DISCOVERY / CONTROL PATH
                          ========================

Stremio client
     |
     | addon request
     v
AIOStreams (self-hosted)
     |
     +--> Torrentio (P2P / non-debrid)
     +--> other verified free P2P-capable Stremio addons
     +--> optional custom addon endpoints
     |
     | normalized + filtered + sorted + deduplicated stream results
     v
Stremio client
     |
     | selected torrent infohash / file index
     v
stremio-libtorrent-server


                          MEDIA / DATA PATH
                          =================

BitTorrent peers
     |
     | Internet / P2P
     v
stremio-libtorrent-server
     |
     | libtorrent download
     | 10 GiB read-ahead
     | 300 GiB disk cache
     v
local HTTP/HTTPS stream
     |
     | LAN / trusted Tailscale path
     v
Stremio player
```

The most important architectural distinction is that **AIOStreams is not the media proxy**. It is the Stremio addon aggregation/control plane. After the user selects a torrent result, the Stremio client hands the torrent information to its configured streaming server. `stremio-libtorrent-server` is the component that joins the swarm, downloads pieces, maintains the read-ahead window, stores cache data, and serves the media bytes to the player.

## 5. Component responsibilities

### 5.1 Stremio client

Stremio remains the user-facing application on TVs, PCs, phones, and supported browsers.

Responsibilities:

- display metadata and catalogs;
- query installed addons;
- display AIOStreams results;
- send the selected torrent to the configured streaming server;
- play the HTTP/HTTPS stream returned by the streaming server.

The Stremio client must not be the primary torrent engine for streams that are intended to use the new server-side buffer.

### 5.2 AIOStreams

AIOStreams will run as a self-hosted Docker service.

Planned stack root:

```text
/srv/docker/aiostreams
```

Primary responsibilities:

- query configured Stremio stream addons;
- combine results into one addon;
- deduplicate duplicate releases;
- filter unwanted results;
- sort preferred releases consistently;
- normalize stream labels and presentation;
- provide one configuration point for the torrent discovery layer.

**LOCKED:** AIOStreams v1 will use only upstreams that can operate without paid debrid or other subscription services.

Initial source policy:

- Torrentio is the initial P2P source to restore through AIOStreams.
- Additional free P2P-capable sources may be added only after verifying that they return usable torrent/infohash streams without a paid backend.
- Existing special-purpose catalog, subtitle, TV, and metadata addons do not need to be migrated into AIOStreams in v1.
- Bulgarian-specific addons such as Zamunda may remain directly installed until custom-addon compatibility is verified.

### 5.3 stremio-libtorrent-server

`stremio-libtorrent-server` will run as the central torrent playback engine.

Planned stack root:

```text
/srv/docker/stremio-libtorrent-server
```

Primary responsibilities:

- receive the torrent selected in Stremio;
- join DHT/trackers/PEX/LSD and connect to peers;
- accept inbound peers where network configuration permits;
- prioritize pieces around the current playhead;
- build a large read-ahead window ahead of playback;
- maintain a persistent download cache;
- expose the resulting stream to Stremio through HTTP/HTTPS Range serving;
- optionally transcode only when a client cannot direct-play the source.

This component is the main solution to the original playback problem: large 4K files can stall when short-lived swarm or Internet throughput drops reach the player before the client has accumulated enough buffer.

## 6. Locked buffering and cache policy

### Read-ahead

**LOCKED:**

```text
STREMIOSRV_READAHEAD_BYTES=10GiB
```

This value is a playhead read-ahead target, not a startup requirement. Playback does not wait for all 10 GiB to be downloaded before starting. The server starts serving when the required initial pieces are available and continues filling the larger window ahead of the playhead in the background.

The purpose of the large window is to absorb temporary drops in available torrent throughput.

Illustrative protection at common high-bitrate rates:

| Approximate bitrate | 10 GiB represents roughly |
| ---: | ---: |
| 60 Mbit/s | 24 minutes |
| 80 Mbit/s | 18 minutes |
| 100 Mbit/s | 14 minutes |
| 150 Mbit/s | 9 minutes |

These values are approximate and describe the amount of media represented by 10 GiB, not a promise that the server will always have the full target window available. The actual buffer depends on swarm performance and how quickly the server can fetch pieces.

### Disk cache

**LOCKED:**

```text
STREMIOSRV_CACHE_SIZE=300GiB
```

The cache must live on persistent host storage rather than ephemeral container storage.

Implementation must confirm that the selected filesystem has enough real free space for:

- the 300 GiB cache budget;
- container/runtime overhead;
- temporary/transcode data where applicable;
- a safety margin so the filesystem cannot be exhausted.

If the server does not have sufficient free capacity, implementation must stop and choose an appropriate existing storage filesystem before starting the stack. The architecture value remains 300 GiB unless explicitly reopened.

### Initial torrent-engine tuning

The following v1 policy is **LOCKED** unless a deployment test shows a concrete reason to change it:

```text
STREMIOSRV_READAHEAD_BYTES=10GiB
STREMIOSRV_CACHE_SIZE=300GiB
STREMIOSRV_DOWNLOAD_RATE_LIMIT=0
STREMIOSRV_ADAPTIVE_PICKING=false
```

- Download throughput remains uncapped so the server can fill the read-ahead window as quickly as the swarm allows.
- Experimental adaptive piece picking remains disabled for the first deployment. The normal playhead-first behavior will be validated before enabling experimental tuning.
- Stream and first-piece timeout values remain at upstream defaults initially and are changed only if testing on thin swarms demonstrates a need.

## 7. BitTorrent peer port and qBittorrent coexistence

The upstream default BitTorrent listen port for `stremio-libtorrent-server` is `6881`, but this host already uses `6881/TCP` and `6881/UDP` for qBittorrent.

Therefore the new service must not bind host port 6881.

**LOCKED v1 port:**

```text
STREMIOSRV_BT_LISTEN_PORT=6882
```

Publish both protocols consistently:

```text
6882/TCP
6882/UDP
```

Target relationship:

```text
qBittorrent
  -> 6881/TCP+UDP

stremio-libtorrent-server
  -> 6882/TCP+UDP
```

Router forwarding for `6882/TCP` and `6882/UDP` is desirable for maximum inbound-peer connectivity, but must be implemented as a separate explicit router rule and must not replace or disturb the existing qBittorrent `6881` rule.

Only the BitTorrent peer port is a candidate for public inbound forwarding. Web, API, library, admin, and streaming control endpoints must not be exposed as general public-Internet services.

## 8. Network and publishing model

### 8.1 AIOStreams

The planned canonical internal URL is:

```text
https://aio.pirocorp.com
```

Target path:

```text
Stremio / browser
    -> AdGuard Home explicit rewrite
    -> 192.168.0.10 / Nginx Proxy Manager
    -> AIOStreams published host port
```

**LOCKED:**

- add an explicit AdGuard rewrite for `aio.pirocorp.com`;
- do not introduce wildcard DNS;
- terminate normal browser/addon HTTPS through the existing Nginx Proxy Manager pattern;
- do not expose the AIOStreams configuration service directly to the public Internet;
- keep AIOStreams configuration secrets out of Git.

The exact AIOStreams host port and pinned image version are implementation details to verify against the current stable release.

### 8.2 stremio-libtorrent-server

The streaming server must be reachable from the Stremio clients that are intended to use the centralized torrent engine.

Client-facing access must support trusted HTTPS where the Stremio TV application requires it.

The exact v1 TLS/client URL method is intentionally left for implementation validation because the upstream project supports more than one workable pattern:

- its generated trusted `*.stremio.rocks` endpoint;
- a custom trusted certificate and `SERVER_URL`;
- potentially the existing local reverse-proxy pattern if all required Range/stream behavior is verified.

The selected method must satisfy all of these requirements:

1. TVs can save and use the streaming server URL.
2. Range requests and seeks work correctly.
3. Large MKV/HEVC direct-play streams work without an unnecessary transcoding hop.
4. The endpoint is reachable on the LAN and, where desired, through trusted Tailscale access.
5. The web/API endpoint is not published as a general public Internet service.
6. The selected route does not introduce a new bottleneck that defeats the purpose of moving playback buffering to the server.

## 9. Client behavior and known platform differences

### TVs

The target behavior is:

```text
Stremio TV app
   -> AIOStreams for source discovery
   -> stremio-libtorrent-server as Streaming Server
   -> local media stream over LAN
```

The streaming server URL must be configured and validated on at least one primary TV before the deployment is considered complete.

### Desktop Stremio

The upstream `stremio-libtorrent-server` documentation notes that the Stremio desktop v6 shell may force its bundled localhost streaming server when launched normally. The implementation must therefore test the supported custom `--webui-url`/development launch method or another documented current method before declaring desktop clients fully migrated.

This desktop behavior is a client integration detail, not a reason to change the server architecture.

### Browser

Browser playback capability depends on browser codec/container support. Large MKV/HEVC content should be validated primarily with native Stremio TV/desktop clients rather than treating browser playback as the acceptance baseline.

## 10. AIOStreams configuration policy

The purpose of AIOStreams is to improve source presentation, not to hide paid backends inside the configuration.

**LOCKED policy:**

- no debrid API keys;
- no paid Usenet credentials;
- no paid addon/search provider credentials required for normal operation;
- prefer P2P torrent results that expose an infohash/magnet usable by the Stremio torrent engine;
- use deduplication to collapse duplicate releases from multiple sources;
- prefer high-quality results while retaining enough alternatives for weak/sparse swarms;
- do not filter so aggressively that only one release remains for a title.

Detailed sorting rules such as resolution, HDR/Dolby Vision, codec, language, seeders, and maximum file size will be tuned after the base path works end to end.

## 11. Addons that remain outside AIOStreams initially

AIOStreams v1 is a stream-source consolidation project, not a full rebuild of every Stremio addon.

The following categories remain directly installed unless a later cleanup has a demonstrated benefit:

- Cinemeta / metadata providers;
- subtitle addons such as OpenSubtitles and bgSubs;
- live-TV / M3U / EPG addons;
- catalog-only addons;
- local files;
- special-purpose Bulgarian catalog/content addons that are not yet proven as AIOStreams custom upstreams.

The migration is considered successful when duplicate torrent-source addons are no longer needed directly in Stremio and AIOStreams is the primary torrent stream-results addon.

## 12. Components explicitly excluded from v1

The following are intentionally excluded:

- TorBox;
- Real-Debrid;
- Premiumize;
- AllDebrid;
- EasyDebrid;
- other paid debrid providers;
- paid AIOStreams hosting;
- StremThru as a debrid abstraction layer;
- MediaFlow as an additional proxy hop;
- a separate generic HTTP cache proxy;
- experimental adaptive torrent picking;
- unnecessary transcoding when clients can direct-play.

MediaFlow and StremThru are not rejected as software in general. They are simply unnecessary for the locked free P2P v1 path and would add extra hops without solving a demonstrated requirement.

## 13. Security boundaries

### Secrets

Do not commit:

- AIOStreams secret/configuration keys;
- generated user configuration URLs containing private tokens;
- TLS private keys;
- Stremio account tokens;
- any future private addon credentials.

Use local `.env` or file-backed secrets where supported and keep real secret files out of Git.

### Network exposure

Public inbound exposure must be limited to what is required for BitTorrent peer connectivity.

Allowed candidate public forwarding:

```text
6882/TCP
6882/UDP
```

Not allowed as general public endpoints:

- AIOStreams configuration UI;
- stremio-libtorrent-server web UI;
- streaming-server API ports;
- library/download manager UI;
- Docker management ports.

LAN/Tailscale remains the management and client trust boundary.

## 14. Data-flow detail

A normal playback should follow this sequence:

1. A Stremio client opens a film or episode.
2. The client requests streams from AIOStreams.
3. AIOStreams queries Torrentio and any other enabled free P2P upstreams.
4. AIOStreams normalizes, deduplicates, filters, sorts, and returns the torrent stream list.
5. The user selects a stream.
6. Stremio sends the selected torrent infohash/file information to the configured `stremio-libtorrent-server` instance.
7. The server resolves metadata and discovers peers through trackers/DHT/PEX/LSD.
8. libtorrent prioritizes the playhead and starts downloading the required pieces.
9. Playback begins when the initial required data is available; it does not wait for 10 GiB.
10. The server continues filling up to the 10 GiB read-ahead target while playback proceeds.
11. Downloaded data is retained inside the 300 GiB LRU-managed cache.
12. The Stremio player consumes the stream over the LAN/Tailscale path rather than pulling torrent pieces directly from Internet peers itself.

## 15. Expected resilience behavior

The 10 GiB read-ahead is intended to make short-term throughput variation invisible to the player once a healthy buffer has accumulated.

Expected behavior:

```text
Healthy swarm burst
      |
      v
server downloads faster than playback
      |
      v
read-ahead grows
      |
      v
short swarm / ISP throughput dip
      |
      v
player continues reading already cached data
      |
      v
server catches up when throughput recovers
```

A large buffer cannot solve a torrent whose sustainable average download rate remains below the video's required bitrate indefinitely. It can, however, absorb temporary stalls and throughput jitter that currently reach the player too quickly.

## 16. Resource model

### CPU

Direct play should be lightweight. Transcoding is not part of the normal path and should occur only for incompatible client/media combinations.

### RAM

The 10 GiB read-ahead value is not a requirement to reserve 10 GiB of RAM. It defines the torrent download window ahead of the playhead. Actual runtime memory use must be observed during testing.

### Disk

The 300 GiB cache is a real storage budget and requires persistent disk capacity.

Implementation must identify the exact cache filesystem before deployment and record:

- mount/path;
- filesystem;
- available capacity before deployment;
- free-space safety margin;
- expected I/O characteristics.

An SSD is preferred for cache responsiveness if sufficient capacity exists, but the final path must follow the server's real storage layout rather than assuming a device that may not be present.

### Network

A wired server connection is preferred. Torrent download rate should initially remain unlimited so the server can build the read-ahead window aggressively.

## 17. Implementation sequence

The first implementation should follow this order.

### Phase 1 — preflight

- verify the Ubuntu host is `amd64`/x86-64 for the published `stremio-libtorrent-server` image;
- inspect current CPU, RAM, GPU, network interfaces, and free disk space;
- select a persistent filesystem that can safely support the 300 GiB cache;
- verify existing ports, especially qBittorrent `6881/TCP+UDP`;
- reserve `6882/TCP+UDP` for the new torrent engine;
- verify no existing stack uses the planned AIOStreams host port;
- verify existing NPM and AdGuard configuration conventions.

### Phase 2 — deploy AIOStreams

- create `/srv/docker/aiostreams`;
- use the current stable self-hosted Docker image after rechecking upstream documentation;
- generate a strong application secret locally;
- publish a suitable host port;
- create explicit AdGuard rewrite for `aio.pirocorp.com`;
- add NPM proxy host and existing wildcard TLS certificate;
- confirm configuration UI access only from intended networks.

### Phase 3 — deploy stremio-libtorrent-server

- create `/srv/docker/stremio-libtorrent-server`;
- use the current stable published image after verification;
- mount persistent server/cache state on the selected storage path;
- configure `10GiB` read-ahead;
- configure `300GiB` cache;
- configure BitTorrent listen port `6882`;
- publish `6882/TCP` and `6882/UDP` on the host;
- keep download rate unlimited;
- leave experimental adaptive picking disabled;
- choose and validate the trusted client HTTPS method.

### Phase 4 — configure sources

- add Torrentio to AIOStreams in P2P/non-debrid mode;
- verify returned streams contain usable torrent information;
- add additional free P2P upstreams one at a time only after confirming compatibility;
- configure initial deduplication, quality filters, and sorting;
- retain special-purpose addons separately where appropriate.

### Phase 5 — configure Stremio clients

- install the configured AIOStreams addon;
- point the primary TV Stremio client at `stremio-libtorrent-server` as its Streaming Server;
- verify addon results still appear normally;
- verify selecting a torrent causes the server, not the TV, to join the swarm;
- repeat for other supported clients after the first client succeeds.

### Phase 6 — router and peer-connectivity validation

- add router forwarding for `6882/TCP` and `6882/UDP` only if the current topology permits it;
- keep existing qBittorrent `6881` forwarding unchanged;
- verify the new server is actually listening on the configured port;
- verify inbound connectivity without exposing management endpoints.

### Phase 7 — large-file resilience test

Test with a legitimately accessible high-bitrate source large enough to exercise the cache strategy.

Verify:

- startup time is reasonable;
- playback starts before the full 10 GiB read-ahead is accumulated;
- read-ahead grows during playback when torrent throughput exceeds media bitrate;
- server cache usage increases as expected;
- seek forward/back works;
- a temporary bandwidth reduction is absorbed when enough buffer exists;
- client playback does not revert to the local bundled torrent engine;
- no unexpected transcoding occurs for direct-play-capable media.

## 18. Acceptance criteria

The v1 deployment is accepted when all applicable checks pass:

- [ ] AIOStreams runs self-hosted under `/srv/docker/aiostreams`.
- [ ] `https://aio.pirocorp.com` is reachable through explicit AdGuard DNS and Nginx Proxy Manager.
- [ ] No paid debrid or hosted service is required.
- [ ] AIOStreams returns P2P Torrentio results without debrid credentials.
- [ ] Duplicate torrent-source addons no longer need to be installed directly in Stremio.
- [ ] `stremio-libtorrent-server` runs under `/srv/docker/stremio-libtorrent-server`.
- [ ] The torrent engine uses `6882/TCP` and `6882/UDP`, not qBittorrent's `6881`.
- [ ] Read-ahead is configured to `10GiB`.
- [ ] Cache size is configured to `300GiB`.
- [ ] Cache data is stored persistently on a filesystem with a safe free-space margin.
- [ ] Download rate is uncapped initially.
- [ ] Experimental adaptive picking remains disabled initially.
- [ ] A Stremio TV client successfully uses the self-hosted streaming server.
- [ ] A selected AIOStreams torrent is downloaded by the Ubuntu server rather than directly by the client.
- [ ] Large MKV/HEVC content direct-plays correctly on a capable native client.
- [ ] Seeks work correctly.
- [ ] Short-term throughput drops can be absorbed after read-ahead has accumulated.
- [ ] Existing qBittorrent operation and port forwarding remain intact.
- [ ] Management/configuration endpoints are not exposed as public Internet services.
- [ ] No committed file contains real secrets, tokens, private keys, or account credentials.

## 19. Rejected alternatives

### Paid debrid as the primary backend

Rejected because the architecture requirement is free/self-hosted operation with no subscription dependency.

### TorBox free tier

Rejected as an architectural dependency. It is not required for the desired P2P path and the v1 goal is to avoid debrid entirely.

### MediaFlow Proxy in the main path

Rejected for v1. MediaFlow is useful for HTTP/HLS/DASH proxying, but the target source path is direct P2P handled by the dedicated libtorrent engine. Adding MediaFlow would introduce an extra hop without solving a demonstrated requirement.

### StremThru in the main path

Rejected for v1 because debrid-store abstraction is not required when the architecture deliberately excludes debrid backends.

### Relying only on the stock Stremio torrent engine

Rejected because the original problem is insufficient resilience to temporary throughput variation on large high-bitrate files and because the stock client-side engine does not provide the desired centrally controlled 10 GiB read-ahead / 300 GiB cache model.

### Running a generic caching reverse proxy in front of torrent streams

Rejected for v1. The torrent-aware streaming engine already controls piece priority, read-ahead, persistent cache, and Range serving. A generic proxy would add complexity before demonstrating a missing capability.

### Using port 6881 for the new torrent engine

Rejected because qBittorrent already owns `6881/TCP+UDP` on the same host.

## 20. Decisions intentionally left for implementation

The following must be verified during deployment and are not architecture changes:

- exact current stable AIOStreams image/version to pin;
- exact current stable `stremio-libtorrent-server` image/version to pin;
- AIOStreams host port behind NPM;
- exact persistent host path backing the 300 GiB streaming cache;
- final trusted HTTPS/Streaming Server URL method for TVs;
- whether the server uses the generated `*.stremio.rocks` URL or a custom trusted certificate/domain;
- whether NPM is suitable for the streaming-server data path or should remain outside that path;
- exact router rule for `6882/TCP+UDP`;
- final seeding-duration/upload-limit policy;
- whether hardware transcoding should be enabled after inspecting the host GPU and client codec support;
- final Torrentio configuration and additional verified free P2P source addons;
- detailed AIOStreams sorting/formatting rules;
- whether the library UI and optional library addon are useful after the core streaming path is stable;
- desktop Stremio launch configuration required to persist the remote streaming server on the currently installed client version.

## 21. Implementation guardrails

1. Start from this document and preserve every **LOCKED** decision.
2. Re-check current upstream documentation immediately before deployment because both projects evolve quickly.
3. Pin explicit image versions in the final deployed state rather than leaving a floating tag indefinitely.
4. Do not introduce a paid dependency to solve a configuration issue.
5. Do not add debrid credentials.
6. Do not reuse qBittorrent's host peer port `6881`.
7. Keep the new torrent engine on `6882/TCP+UDP` unless a real conflict discovered during preflight forces an explicitly documented change.
8. Do not reduce the 10 GiB read-ahead or 300 GiB cache merely to match upstream defaults.
9. Do not place the 300 GiB cache in ephemeral container storage.
10. Do not publicly expose configuration/admin/API endpoints.
11. Keep direct-play as the preferred media path; do not transcode without a client compatibility reason.
12. Validate one client and one source end to end before adding more upstream addons.
13. Add sources incrementally so a failing addon cannot hide whether the core architecture works.
14. Preserve the existing qBittorrent deployment and router forwarding.
15. After successful implementation, update the repository to document the actual deployed ports, paths, pinned versions, TLS method, source configuration, and operational commands.

## 22. Post-implementation documentation transition

After deployment is validated, create a follow-up documentation PR that:

1. adds an implemented service/runbook for AIOStreams;
2. adds an implemented service/runbook for `stremio-libtorrent-server` or a combined Stremio streaming service document if operationally clearer;
3. adds the deployed URLs and ports to current-state documentation;
4. records the real persistent cache path and storage capacity;
5. records the actual pinned Docker images/versions;
6. records the final trusted HTTPS/client Streaming Server URL strategy;
7. records router forwarding for `6882/TCP+UDP` if enabled;
8. records the validated AIOStreams upstream source list and filtering policy;
9. removes this work from Planned status; and
10. retains this roadmap as an architecture decision record if it remains useful.

## 23. Official implementation references

Verify these again during implementation because supported options and client behavior can change:

- AIOStreams repository: https://github.com/Viren070/AIOStreams
- AIOStreams project documentation: https://docs.aiostreams.viren070.me/
- stremio-libtorrent-server repository: https://github.com/andrewhack/stremio-libtorrent-server
- stremio-libtorrent-server quick start: https://github.com/andrewhack/stremio-libtorrent-server/blob/main/QUICKSTART.md
- stremio-libtorrent-server operations/tuning: https://github.com/andrewhack/stremio-libtorrent-server/blob/main/README.md

## 24. One-paragraph implementation prompt

Implement the approved Stremio streaming architecture on `piroman-server` (`192.168.0.10`) using only free/self-hosted components. Deploy AIOStreams under `/srv/docker/aiostreams` and publish it internally as `https://aio.pirocorp.com` using the existing explicit AdGuard rewrite plus Nginx Proxy Manager pattern. Deploy `stremio-libtorrent-server` under `/srv/docker/stremio-libtorrent-server` as the central P2P streaming engine, configure `STREMIOSRV_READAHEAD_BYTES=10GiB`, `STREMIOSRV_CACHE_SIZE=300GiB`, unlimited initial download rate, experimental adaptive picking off, and BitTorrent peer port `6882/TCP+UDP` because qBittorrent already owns `6881`. Use persistent storage with enough real free capacity, configure Torrentio through AIOStreams in non-debrid P2P mode, point a primary Stremio TV client at the self-hosted streaming server, and validate with a large high-bitrate direct-play source that playback starts before the full read-ahead is accumulated, seeks work, the server builds buffer ahead of the playhead, and temporary throughput dips are absorbed. Do not add debrid, MediaFlow, StremThru, paid hosting, public management endpoints, or any other unapproved dependency.