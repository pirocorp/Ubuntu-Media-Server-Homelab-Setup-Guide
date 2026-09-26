# Current State

Status: Implemented
Purpose: Record the live homelab baseline, key endpoints, and current implementation scope.
Depends on: [Platform docs](../platform/README.md)
Related docs: [Architecture](./architecture.md), [Service inventory](./service-inventory.md), [Services](../services/README.md), [Stremio streaming architecture](./stremio-streaming-architecture.md)

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

These service names resolve through the current AdGuard local DNS design. They are the intended LAN names and VPN names for trusted Tailscale clients.

| Service | URL |
| --- | --- |
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

NetAlertX uses an explicit AdGuard rewrite for `netalertx.pirocorp.com`, Nginx Proxy Manager forwarding to `192.168.0.10:20211`, and the existing `pirocorp.com` / `*.pirocorp.com` Let's Encrypt certificate.

AIOStreams uses the same local publishing model: explicit AdGuard rewrite for `aio.pirocorp.com`, Nginx Proxy Manager forwarding to `192.168.0.10:3001`, and the existing wildcard certificate.

`stremio-libtorrent-server` uses the upstream trusted `*.stremio.rocks:12470` HTTPS method for Stremio client access rather than putting the media path behind Nginx Proxy Manager. The generated hostname is instance-specific and should be read from the running server logs or `httpsCert.json`.

## Direct Ports

| Service | Address |
| --- | --- |
| AdGuard Home setup | `192.168.0.10:3000` |
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
| AIOStreams | `192.168.0.10:3001` |
| Stremio web UI | `192.168.0.10:8081` |
| Stremio streaming API | `192.168.0.10:11470` |
| Stremio trusted HTTPS | `192.168.0.10:12470` |
| Stremio BitTorrent peers | `6882/TCP+UDP` |

qBittorrent remains on `6881/TCP+UDP`; the Stremio torrent engine uses the separate `6882/TCP+UDP` assignment.

## Active Docker Stack Roots

- `/srv/docker/adguard-home`
- `/srv/docker/aiostreams`
- `/srv/docker/audiobookshelf`
- `/srv/docker/bitmagnet`
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

## Stremio Streaming Stack Snapshot

| Component | Current deployed state |
| --- | --- |
| AIOStreams | `ghcr.io/viren070/aiostreams:v2.34.1`, healthy |
| AIOStreams state | `/srv/docker/aiostreams/data` |
| AIOStreams URL | `https://aio.pirocorp.com` |
| Streaming engine | `androshack/stremio-libtorrent-server:1.6.15`, healthy |
| Streaming cache/state | `/mnt/data/stremio-libtorrent-server` |
| Read-ahead | `10GiB` |
| Cache budget | `300GiB` |
| Download rate | unlimited |
| Adaptive picking | disabled |
| Torrent peer port | `6882/TCP+UDP` |
| qBittorrent peer port | `6881/TCP+UDP`, unchanged |
| Transcoding | not part of normal v1 path |
| Current implementation phase | Phases 1-3 complete; Phases 4-7 pending |

See [Stremio + AIOStreams implemented architecture](./stremio-streaming-architecture.md) for the as-built architecture, validated preflight values, port plan, security boundaries, and remaining acceptance checks.

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

Direct LAN access remains permitted from `192.168.0.0/24`.

AIOStreams was validated directly from the NPM container on `192.168.0.10:3001`; no additional UFW rule was required for that path in the current deployment.

## Mounted Storage Snapshot

| Mount | Label | Filesystem | Size | Used | Available | Use |
| --- | --- | --- | --- | --- | --- | --- |
| `/` | system | `ext4` | `1.8T` | `125G` | `1.6T` | `8%` |
| `/mnt/iac` | `IAC` | `ntfs` | `3.7T` | `1.8T` | `1.9T` | `50%` |
| `/mnt/lp` | `LP` | `ntfs` | `7.3T` | `3.7T` | `3.7T` | `51%` |
| `/mnt/data` | `DATA` | `ntfs` | `7.3T` | `3.3T` | `4.1T` | `45%` |
| `/mnt/ia` | `IA` | `ntfs` | `3.7T` | `2.8T` | `894G` | `77%` |
| `/mnt/comp` | `COMP` | `ntfs` | `3.7T` | `1.1T` | `2.7T` | `28%` |

`/mnt/data/stremio-libtorrent-server` is the selected persistent cache/state location for the central Stremio torrent engine. The `300GiB` cache budget is well within the preflight free-space margin.

## Scope

### Implemented

- Base Ubuntu Server host and SSH administration
- Docker and Docker Compose workloads
- Portainer container management
- AdGuard Home local DNS and filtering
- Nginx Proxy Manager internal ingress
- Tailscale remote access with `piroman-server` as subnet router
- Storage mounts and Samba shares
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
- `stremio-libtorrent-server` central torrent engine with `10GiB` read-ahead, `300GiB` persistent cache, `6882/TCP+UDP`, and trusted client HTTPS

### Planned

- [Stremio + AIOStreams remaining implementation phases](../roadmaps/stremio-aiostreams/README.md): Torrentio/source configuration, client cutover, router peer-port validation, and large-file resilience testing
- [Usenet stack and architecture roadmap](../roadmaps/usenet/README.md)
- [ShadowBroker OpenClaw integration roadmap](../roadmaps/shadowbroker-openclaw-integration.md)
