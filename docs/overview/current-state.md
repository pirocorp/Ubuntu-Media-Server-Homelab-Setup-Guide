# Current State

Status: Implemented
Purpose: Record the live homelab baseline, key endpoints, and current implementation scope.
Depends on: [Platform docs](../platform/README.md)
Related docs: [Architecture](./architecture.md), [Service inventory](./service-inventory.md), [Services](../services/README.md), [Stremio streaming architecture](./stremio-streaming-architecture.md), [Homepage](../services/homepage/README.md)

## Host Baseline

| Property | Value |
| --- | --- |
| Hostname | `piroman-server` |
| LAN IP | `192.168.0.10` |
| OS | `Ubuntu 26.04 LTS` |
| Kernel | `Linux 7.0.0-22-generic` |
| Architecture | `x86-64` |
| Main admin user | `piroman` |
| Primary service domain | `*.pirocorp.com` |
| Remote access | Tailscale subnet router on host |
| Tailscale server IP | `100.94.205.122` |
| Advertised VPN route | `192.168.0.0/24` |
| Container root | `/srv/docker` |
| Main shared data mount | `/mnt/data` |

## Current Access URLs

These service names normally resolve through the current AdGuard local DNS design. A public wildcard DNS fallback also maps `*.pirocorp.com` to the private LAN address for clients such as the TV that bypass local DNS.

| Service | URL |
| --- | --- |
| Homepage dashboard | `https://home.pirocorp.com` |
| Server monitoring (Netdata) | `https://server.pirocorp.com` |
| Portainer | `https://portainer.pirocorp.com` |
| AdGuard Home | `https://adguard.pirocorp.com` |
| Nginx Proxy Manager admin | `http://npm.pirocorp.com:81` |
| Nextcloud | `https://nextcloud.pirocorp.com` |
| Bitmagnet | `https://bitmagnet.pirocorp.com` |
| qBittorrent | `https://qbittorrent.pirocorp.com` |
| Plex | `https://plex.pirocorp.com` |
| Audiobookshelf | `https://audiobookshelf.pirocorp.com` |
| Immich | `https://immich.pirocorp.com` |
| Kavita | `https://kavita.pirocorp.com` |
| ShadowBroker | `https://shadowbroker.pirocorp.com` |
| NetAlertX | `https://netalertx.pirocorp.com` |
| AIOStreams | `https://aio.pirocorp.com` |
| Stremio Web UI | `https://stremio.pirocorp.com` |

Homepage is the central operational landing page. It is published through Nginx Proxy Manager to `192.168.0.10:3002`. AdGuard has explicit rewrites for `home.pirocorp.com`, `pirocorp.com`, and `www.pirocorp.com`. NPM has one online `301` redirection host for both `pirocorp.com` and `www.pirocorp.com` targeting `https://home.pirocorp.com`. `https://home.pirocorp.com` is validated; final authoritative/public DNS validation of the newly-created apex `pirocorp.com` A record is still pending.

NetAlertX uses an explicit AdGuard rewrite for `netalertx.pirocorp.com`, Nginx Proxy Manager forwarding to `192.168.0.10:20211`, and the existing `pirocorp.com` / `*.pirocorp.com` Let's Encrypt certificate.

AIOStreams uses the same local publishing model: explicit AdGuard rewrite for `aio.pirocorp.com`, Nginx Proxy Manager forwarding to `192.168.0.10:3001`, and the existing wildcard certificate.

For TV/client compatibility, Cloudflare public DNS has DNS-only private-address records:

```text
*.pirocorp.com -> 192.168.0.10
pirocorp.com   -> 192.168.0.10
```

The explicit apex A record was added on 2026-09-29 because a wildcard does not cover the zone apex. At the last validation point, Cloudflare's UI showed the record but direct A queries to both authoritative Cloudflare nameservers had not yet returned the address, so apex resolution remains a pending follow-up rather than a completed acceptance check.

These records publish only an RFC1918/private address and do not make the services Internet-routable. The wildcard exists because the primary TV did not reliably honor the local AdGuard resolver even with manual DNS configuration.

The Stremio Web UI also follows the standard local publishing model: explicit AdGuard rewrite for `stremio.pirocorp.com`, Nginx Proxy Manager forwarding to `192.168.0.10:8081`, WebSockets enabled, and the existing wildcard certificate. The final HTTPS endpoint was validated with `HTTP 200`.

`stremio-libtorrent-server` still uses the upstream trusted `*.stremio.rocks:12470` HTTPS method for the actual Stremio media/streaming-server path rather than putting large media transfers behind Nginx Proxy Manager. The generated hostname is instance-specific and should be read from the running server logs or `httpsCert.json`.

## Direct Ports

| Service | Address |
| --- | --- |
| AdGuard Home web UI | `192.168.0.10:3000` |
| AIOStreams | `192.168.0.10:3001` |
| Homepage | `192.168.0.10:3002` |
| Nginx Proxy Manager admin | `192.168.0.10:81` |
| Portainer | `192.168.0.10:9443` |
| Netdata | `192.168.0.10:19999` |
| Plex | `192.168.0.10:32400` |
| qBittorrent web UI | `192.168.0.10:8080` |
| Bitmagnet web/API | `192.168.0.10:3333-3334` |
| Nextcloud app | `192.168.0.10:8090` |
| ShadowBroker frontend | `192.168.0.10:3010` |
| ShadowBroker backend | `192.168.0.10:8010` |
| Audiobookshelf | `192.168.0.10:13378` |
| Immich | `192.168.0.10:2283` |
| Kavita | `192.168.0.10:5000` |
| NetAlertX web UI | `192.168.0.10:20211` |
| NetAlertX API/metrics | `192.168.0.10:20212` |
| Stremio web UI | `192.168.0.10:8081` |
| Stremio streaming API | `192.168.0.10:11470` |
| Stremio trusted HTTPS | `192.168.0.10:12470` |
| Stremio BitTorrent peers | `6882/TCP+UDP` |

qBittorrent remains on `6881/TCP+UDP`; the Stremio torrent engine uses the separate `6882/TCP+UDP` assignment.

The router now explicitly forwards `6882/TCP+UDP` to `192.168.0.10:6882`. Both TCP and UDP are recorded as operationally validated, while qBittorrent's existing `6881/TCP+UDP` mapping remains unchanged.

## Active Docker Stack Roots

- `/srv/docker/adguard-home`
- `/srv/docker/aiostreams`
- `/srv/docker/audiobookshelf`
- `/srv/docker/bitmagnet`
- `/srv/docker/homepage`
- `/srv/docker/immich`
- `/srv/docker/kavita`
- `/srv/docker/netalertx`
- `/srv/docker/nextcloud`
- `/srv/docker/nginx-proxy-manager`
- `/srv/docker/plex`
- `/srv/docker/portainer`
- `/srv/docker/qbittorrent`
- `/srv/docker/shadowbroker`
- `/srv/docker/stremio-libtorrent-server`

## Homepage Dashboard Snapshot

| Component | Current deployed state |
| --- | --- |
| Homepage | `ghcr.io/gethomepage/homepage:v2.4.0`, healthy |
| Homepage container | `homepage` |
| Homepage host mapping | `192.168.0.10:3002 -> 3000/tcp` |
| Canonical URL | `https://home.pirocorp.com` |
| Docker Socket Proxy | `ghcr.io/tecnativa/docker-socket-proxy:v0.4.2` |
| Socket proxy container | `homepage-dockerproxy` |
| Docker socket access | only through socket proxy; Homepage has no direct socket mount |
| Socket proxy host port | none; `2375/tcp` is not published to host/LAN |
| Docker API mutation | `POST=0` |
| Main Docker network | `homepage_default`; observed subnet `172.31.0.0/16` |
| Private Docker API network | `homepage_docker_api`, internal |
| Config root | `/srv/docker/homepage/config` |
| Secret root | `/srv/docker/homepage/secrets`, mounted read-only, files mode `0600` |
| Dashboard groups | Infrastructure; Media & Libraries; Downloads & Discovery; Cloud & Applications |
| Top-level cards | 15 |

Homepage preflight found port `3000` already occupied by AdGuard Home, so the approved roadmap's planned host port was changed to `3002` without changing the internal Homepage port. `3001` remains assigned to AIOStreams.

The Docker integration is validated end-to-end: Homepage reaches the private socket proxy, Docker `_ping` returns `OK`, Docker API version `29.1.3` was read successfully, container discovery works, and green status indicators expand on demand to CPU, memory, RX, and TX statistics.

Native widgets validated in the deployed dashboard:

- Netdata: `warnings`, `criticals`
- Portainer: `running`, `stopped`, `total`; environment ID `3`
- Nginx Proxy Manager: `enabled`, `disabled`, `total`
- AdGuard Home: `queries`, `blocked`, `filtered`, `latency`
- NetAlertX v2: `total`, `connected`, `new_devices`, `down_alerts`
- Plex: `streams`, `movies`, `tv`
- Immich v2: `users`, `photos`, `videos`, `storage`
- Audiobookshelf: `books`, `booksDuration`
- Kavita: `seriesCount`, `totalFiles`
- qBittorrent: `leech`, `download`, `seed`, `upload`
- Nextcloud: `activeusers`, `numfiles`, `numshares`, `freespace`

Stremio and AIOStreams intentionally use Docker status only. Bitmagnet and ShadowBroker use Docker plus HTTP health monitoring and no custom widget.

All service secrets are local file-backed values referenced with `HOMEPAGE_FILE_*`; no real credentials are committed. Least-privileged or dedicated credentials were used where supported, including Immich `server.statistics`, a non-admin Audiobookshelf `homepage` identity, Kavita authorization key, qBittorrent API key, Nextcloud `NC-Token`, and a regenerated NetAlertX API token.

NetAlertX v2 required one narrow UFW exception because the Homepage container could not initially reach the host API port:

```text
172.31.0.0/16 -> 192.168.0.10:20212/tcp
```

After this rule, the Homepage container received HTTP `200` from the NetAlertX metrics endpoint and the widget became operational.

See [Homepage deployment and operations](../services/homepage/README.md) for the deployed Compose shape, secret-file layout, widget matrix, validation commands, and publishing status.

## Stremio Streaming Stack Snapshot

| Component | Current deployed state |
| --- | --- |
| AIOStreams | `ghcr.io/viren070/aiostreams:v2.34.1`, healthy |
| AIOStreams state | `/srv/docker/aiostreams/data` |
| AIOStreams URL | `https://aio.pirocorp.com` |
| AIOStreams source | Torrentio P2P / non-debrid, validated |
| Streaming engine | `androshack/stremio-libtorrent-server:1.6.15`, healthy |
| Streaming cache/state | `/mnt/data/stremio-libtorrent-server` |
| Stremio Web UI | `https://stremio.pirocorp.com` via AdGuard + NPM to `192.168.0.10:8081` |
| Streaming-server HTTPS | generated trusted `*.stremio.rocks:12470` endpoint, direct media path |
| Read-ahead | `10GiB` |
| Cache budget | `100GiB` |
| Cache eviction grace | upstream default `1800s` / 30 minutes |
| Download rate | unlimited |
| Adaptive picking | disabled |
| Torrent peer port | `6882/TCP+UDP`, router-forwarded and validated |
| qBittorrent peer port | `6881/TCP+UDP`, unchanged |
| Transcoding | not part of normal v1 path |
| Current implementation phase | Phases 1-7 complete |

Phase 4 end-to-end proof confirmed that selected Torrentio results contain usable torrent metadata and that `stremio-libtorrent-server` downloads/caches the selected media on the Ubuntu server.

Phase 5 client acceptance is complete. The primary TV successfully uses the central streaming-server path, forward and backward seeking were exercised repeatedly, stop/reopen/resume worked, sustained 4K playback was validated, and desktop playback/seek were also confirmed. A known client-side limitation remains: the primary TV's built-in `100 Mbit/s` Ethernet link can become the bottleneck for very high-bitrate UHD BluRay REMUX files and can cause brief sub-second stalls when the media demand exceeds the practical Fast Ethernet ceiling.

Phase 6 router/peer-connectivity validation is complete. `6882/TCP+UDP` is listening on the host, LAN TCP reachability was validated, Docker DNAT/forwarding rules for both protocols were confirmed, and live bidirectional UDP traffic with Internet peers was observed. The explicit router `6882/TCP+UDP -> 192.168.0.10:6882` mapping is now enabled. Public inbound TCP on `6882` was independently confirmed open from the Internet, and the UDP peer path is recorded as operational. qBittorrent remains unchanged on `6881/TCP+UDP`. The Stremio web UI, API, and trusted-media ports remain outside the public-forward scope.

Phase 7 resilience acceptance is complete. Two full 4K films were watched through the central server path without server-side playback failure. Forward/backward seek, stop/reopen/resume, desktop long-seek behaviour, `10GiB` read-ahead configuration, and `100GiB` cache/LRU behaviour are accepted for the deployed stack. A synthetic throughput-drop test was intentionally not run; final acceptance is based on real-world sustained 4K use. The known primary-TV `100 Mbit/s` Ethernet ceiling remains the only characterized high-bitrate playback limitation.

The cache budget was reduced from the original `300GiB` plan to `100GiB` after live testing showed that upstream continues filling the wanted file after playback closes. A custom fork was rejected. The current upstream-only policy retains `10GiB` read-ahead and uses size-triggered LRU cleanup. During live 4K playback the cache reached approximately `101G`; the evictor removed the older `Mayday` entry (~24G) while preserving the active `In The Grey` stream, reducing usage to approximately `78G`.

See [Stremio + AIOStreams implemented architecture](./stremio-streaming-architecture.md) for the as-built architecture, validated preflight values, port plan, security boundaries, and completed acceptance status. See [Phase 4 completion](../roadmaps/stremio-aiostreams/phase-4-completion.md) for the source configuration and end-to-end evidence. See [Phase 5 completion](../roadmaps/stremio-aiostreams/phase-5-completion.md) for client acceptance, seek/resume evidence, and the documented TV Ethernet limitation. See [Phase 6 completion](../roadmaps/stremio-aiostreams/phase-6-completion.md) for listener/firewall checks, Docker rules, live UDP peer evidence, and completed TCP+UDP router-forward validation. See [Phase 7 completion](../roadmaps/stremio-aiostreams/phase-7-completion.md) for final sustained-4K/read-ahead/resilience acceptance. See the [Stremio Web UI publishing runbook](../services/stremio-libtorrent-server/web-ui-publishing.md) for the DNS and reverse-proxy configuration.

## NetAlertX Network Publishing Notes

NetAlertX runs with `network_mode: host`, while Nginx Proxy Manager runs in Docker network `nginx-proxy-manager_default`.

Current NPM network details:

```text
Subnet:  172.20.0.0/16
NPM IP:  172.20.0.2
Gateway: 172.20.0.1
```

UFW permits the NPM Docker subnet to reach only the NetAlertX web port on the host:

```text
172.20.0.0/16 -> 192.168.0.10:20211/tcp
```

Homepage has a separate narrow UFW exception for the NetAlertX v2 backend:

```text
172.31.0.0/16 -> 192.168.0.10:20212/tcp
```

Direct LAN access remains permitted from `192.168.0.0/24`.

AIOStreams was validated directly from the NPM container on `192.168.0.10:3001`; no additional UFW rule was required for that path in the current deployment. The Stremio Web UI is now also published through NPM to `192.168.0.10:8081` and returns `HTTP 200` through `https://stremio.pirocorp.com`. Homepage is published through NPM to `192.168.0.10:3002` and is validated at `https://home.pirocorp.com`.

## Mounted Storage Snapshot

| Mount | Label | Filesystem | Size | Used | Available | Use |
| --- | --- | --- | --- | --- | --- | --- |
| `/` | system | `ext4` | `1.8T` | `125G` | `1.6T` | `8%` |
| `/mnt/iac` | `IAC` | `ntfs` | `3.7T` | `1.8T` | `1.9T` | `50%` |
| `/mnt/lp` | `LP` | `ntfs` | `7.3T` | `3.7T` | `3.7T` | `51%` |
| `/mnt/data` | `DATA` | `ntfs` | `7.3T` | `3.3T` | `4.1T` | `45%` |
| `/mnt/ia` | `IA` | `ntfs` | `3.7T` | `2.8T` | `894G` | `77%` |
| `/mnt/comp` | `COMP` | `ntfs` | `3.7T` | `1.1T` | `2.7T` | `28%` |

`/mnt/data/stremio-libtorrent-server` is the selected persistent cache/state location for the central Stremio torrent engine. The current `100GiB` cache budget has substantial filesystem headroom. The budget is not a hard quota and active/protected entries can temporarily push observed usage above it before an eviction pass runs.

## Scope

### Implemented

- Base Ubuntu Server host and SSH administration
- Docker and Docker Compose workloads
- Portainer container management
- AdGuard Home local DNS and filtering
- Nginx Proxy Manager internal ingress
- Tailscale remote access with `piroman-server` as subnet router
- Storage mounts and Samba shares
- Homepage central dashboard under `/srv/docker/homepage`, published at `https://home.pirocorp.com`, with private Docker Socket Proxy, native widgets, Docker status, and file-backed secrets
- Plex media server
- Nextcloud
- qBittorrent
- Bitmagnet
- Audiobookshelf
- Immich
- Kavita
- UPS monitoring with NUT and Netdata
- ShadowBroker
- NetAlertX LAN device inventory and presence monitoring with HTTPS access through `netalertx.pirocorp.com`
- AIOStreams self-hosted control/discovery layer at `https://aio.pirocorp.com`
- Torrentio P2P/non-debrid source configured through AIOStreams with usable infoHash/fileIdx validation
- `stremio-libtorrent-server` central torrent engine with `10GiB` read-ahead, `100GiB` persistent cache, `6882/TCP+UDP`, and trusted client HTTPS
- end-to-end proof that Stremio selections are downloaded and cached by the Ubuntu server
- size-triggered LRU cache eviction validated under real 4K load
- Stremio Web UI published internally at `https://stremio.pirocorp.com` through AdGuard Home and Nginx Proxy Manager
- public DNS-only wildcard `*.pirocorp.com -> 192.168.0.10` for clients that bypass local DNS
- Cloudflare DNS-only apex `pirocorp.com -> 192.168.0.10` created; authoritative/public propagation validation pending
- NPM `301` redirect host configured for `pirocorp.com` and `www.pirocorp.com` to `https://home.pirocorp.com`
- primary-TV client acceptance through the central server path, including forward/backward seek and stop/reopen/resume
- desktop playback and seek acceptance through the configured stack
- primary-TV `100 Mbit/s` Ethernet ceiling characterized as a known client-network limitation for very high-bitrate UHD REMUX playback
- Phase 6 peer-connectivity validation: `6882/TCP+UDP` listeners confirmed, Docker DNAT/forwarding validated, live bidirectional Internet UDP peer traffic observed, explicit router forwarding enabled, public inbound TCP confirmed open, and both TCP/UDP peer paths accepted as operational
- Phase 7 resilience acceptance: two full 4K film sessions through the central path, existing seek/resume validation carried forward, `10GiB` read-ahead and `100GiB` cache policy accepted for normal operation
- Stremio/AIOStreams seven-phase implementation roadmap complete

### Pending Validation

- validate Homepage from a trusted Tailscale client using the existing subnet-route model
- validate authoritative/public resolution of the new Cloudflare apex `pirocorp.com` A record and then confirm the apex `301` redirect end-to-end

### Planned

- [AIOStreams multi-source P2P expansion](../roadmaps/stremio-aiostreams/multi-source-p2p-expansion.md)
- [Usenet stack and architecture roadmap](../roadmaps/usenet/README.md)
- [ShadowBroker OpenClaw integration roadmap](../roadmaps/shadowbroker-openclaw-integration.md)
