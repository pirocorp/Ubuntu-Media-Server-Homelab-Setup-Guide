# OnTheSpot — Self-Hosted Infrastructure

Status: **Deployed (infrastructure only)** — 2026-10-26  
Purpose: Record the validated OnTheSpot Web UI deployment and its integration with existing Docker, Nginx Proxy Manager, local DNS and Homepage.  
Depends on: [Docker and Portainer](../../platform/docker-and-portainer.md), [Networking and reverse proxy](../../platform/networking-and-reverse-proxy.md), [Homepage](../homepage/README.md).

## As-built

| Setting | Deployed value |
| --- | --- |
| Application | OnTheSpot upstream tag `v2.0.2beta4` (Git commit `48f6061`) |
| Effective local image | `onthespot:v2.0.2beta4-local1` |
| Docker Compose stack | `/srv/docker/onthespot/compose.yml` |
| Environment file | `/srv/docker/onthespot/.env` (host-local; do not commit) |
| Persistent app data | `/srv/docker/onthespot/otsdata/Config` |
| Video path | `/srv/docker/onthespot/otsdata/Videos` |
| Music path | `/mnt/data/Music/Spotify` (existing NTFS mount; write test passed) |
| Web UI | `https://onthespot.pirocorp.com` via Nginx Proxy Manager |
| Backend | `http://192.168.0.10:6767` |

The music path is intentionally outside the system SSD; the Compose stack and application configuration stay under `/srv/docker`. The desired `Artist/Album/Track.mp3` layout is **not yet acceptance-tested**.

### Image build and permission workaround

No usable published `v2.0.2beta4` Docker Hub manifest was found for `moddroid94/ots-dl`, so the fixed upstream Git tag was built locally:

```bash
cd /srv/docker/onthespot
git clone --depth 1 --branch v2.0.2beta4 https://github.com/ots-downloader/onthespot.git source
docker build -t onthespot:v2.0.2beta4 ./source
```

The initial container repeatedly failed with `Could not import module "onthespot.main"`. Diagnosis found `/app/app/onthespot` owned by root with mode `700` and `main.py` mode `600`, while the app ran as UID 1000. A **separate local overlay Dockerfile**, rather than an edit to the tagged upstream source, fixes ownership:

```dockerfile
# /srv/docker/onthespot/Dockerfile.local
FROM onthespot:v2.0.2beta4
USER root
RUN chown -R 1000:1000 /app/app
USER 1000:1000
```

```bash
docker build -f Dockerfile.local -t onthespot:v2.0.2beta4-local1 .
```

The corrected image passed a direct `import onthespot.main` test (`IMPORT OK`).

### Compose/environment convention

Version and paths are configured in **`.env`**, not hardcoded in the Compose file:

```dotenv
ONTHESPOT_IMAGE=onthespot:v2.0.2beta4-local1
ONTHESPOT_WEB_PORT=6767
ONTHESPOT_MUSIC_DIR=/mnt/data/Music/Spotify
ONTHESPOT_VIDEO_DIR=./otsdata/Videos
ONTHESPOT_CONFIG_DIR=./otsdata/Config
```

The Compose service uses `image: ${ONTHESPOT_IMAGE}` and `pull_policy: never` to **require the locally built, pinned image**. Host music, video and configuration directories are persistent bind mounts. Credentials and live `.env` contents remain on the server.

## Publishing and discovery

- **Nginx Proxy Manager:** `onthespot.pirocorp.com` -> `http://192.168.0.10:6767`; existing `pirocorp.com, *.pirocorp.com` certificate; Force SSL, HTTP/2, WebSockets and Block Common Exploits enabled.
- **NPM access list:** `LAN Only` with `allow 192.168.0.0/24`, then `deny all`; no Basic Auth. The browser loaded the site successfully through HTTPS. Access restriction from outside the LAN was **configured but not independently tested**.
- **DNS:** resolving `onthespot.pirocorp.com` returned `192.168.0.10`, including a direct DNS query to `192.168.0.10`. No new AdGuard rewrite was needed. Windows deliberately keeps both local AdGuard and Telenet DNS; resolver ordering is not a guaranteed strict failover.
- **Homepage:** added the `OnTheSpot` card to **Downloads & Discovery**, alongside qBittorrent/Bitmagnet, linking to `https://onthespot.pirocorp.com`. Existing cards were retained.

NPM's `LAN Only` policy governs **proxy access**, not direct traffic to the host's published port `6767`. No new router port forwarding was configured for this deployment; the host is behind NAT.

## Checks and remaining work

Observed on 2026-10-26:

- NTFS music directory test under `piroman`: **WRITE TEST: OK**; roughly 4 TB available at test time.
- `docker compose config --quiet` passed; `docker compose config --images` resolved `onthespot:v2.0.2beta4-local1`; `pull_policy: never` verified.
- Python module import returned `IMPORT OK`.
- `curl http://127.0.0.1:6767/` returned **HTTP 200**; browser showed **FastAPI Core Ready** through HTTPS.
- Homepage displayed the new card in the intended group.

Not yet accepted: Spotify account setup, a successful song download, and metadata/folder-layout checks. The WAN-origin access-list test was intentionally skipped for this NAT-protected deployment.

### Operations

```bash
cd /srv/docker/onthespot
docker compose config --quiet
docker compose ps
docker compose logs --tail=80 onthespot
# Restart only this service, using the pinned local image:
docker compose up -d --no-deps --pull never onthespot
```

This change documents an existing deployment only. **No credentials, session material, live Compose files, binaries, downloaded media, or new production changes** are included in the repository.
