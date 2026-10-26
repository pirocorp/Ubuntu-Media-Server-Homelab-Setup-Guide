# OnTheSpot — Self-Hosted Infrastructure Runbook

Status: **Infrastructure deployed** (2026-10-26); Spotify account/download acceptance **pending**.  
Purpose: Reproduce the working OnTheSpot Docker Web UI and its Nginx Proxy Manager, DNS and Homepage integration on this homelab.  
Depends on: [Docker and Portainer](../../platform/docker-and-portainer.md), [Networking and reverse proxy](../../platform/networking-and-reverse-proxy.md), [Homepage](../homepage/README.md).

## 1. What was deployed, and why

OnTheSpot is a browser-managed music downloader, hosted alongside the existing Docker services. The **application and persistent settings stay on the system SSD**, while music is stored on the existing large NTFS data disk.

| Item | Accepted implementation | Reason |
| --- | --- | --- |
| Upstream source | Git tag `v2.0.2beta4`, commit `48f6061` | Reproducible source instead of a moving branch |
| Docker image | `onthespot:v2.0.2beta4-local1` | Fixed, locally built image with a required permissions fix |
| Stack | `/srv/docker/onthespot` | Same organization as the other homelab services |
| Compose configuration | `compose.yml` + private `.env` | Keep image tag, host paths and port outside the Compose YAML |
| App config | `/srv/docker/onthespot/otsdata/Config` | Survives recreation and upgrades |
| Music | `/mnt/data/Music/Spotify` | Keeps large downloaded files off the system SSD |
| UI | `https://onthespot.pirocorp.com` | Reuses the existing TLS reverse proxy |
| Access | NPM `LAN Only` list | No public Web UI exposure planned |
| Discovery | Existing DNS + Homepage `Downloads & Discovery` card | One familiar entry point; no new DNS service |

`Artist/Album/Track.mp3` and MP3 quality are **desired application settings**, not yet validated outcomes. No Spotify credentials or audio files are stored in this repository.

### Data flow

```text
Windows / iPhone (LAN)
    | DNS: onthespot.pirocorp.com -> 192.168.0.10
    v
Nginx Proxy Manager (HTTPS, wildcard certificate, LAN Only)
    | HTTP to 192.168.0.10:6767
    v
OnTheSpot FastAPI + Web UI (UID/GID 1000:1000)
    |-- /root/.config/onthespot -> /srv/docker/onthespot/otsdata/Config
    |-- /root/Music/OnTheSpot  -> /mnt/data/Music/Spotify
    '-- /root/Videos/OnTheSpot -> /srv/docker/onthespot/otsdata/Videos

Homepage -> OnTheSpot link in Downloads & Discovery
```

## 2. Preconditions — before changing anything

These instructions assume the Ubuntu server already runs Docker/Compose, NPM, AdGuard Home and Homepage. On the recorded host, `piroman` is UID/GID `1000:1000`, the server address is `192.168.0.10`, and the LAN is `192.168.0.0/24`. **Substitute your own values when reproducing elsewhere.**

Check the prerequisites and the mounted music filesystem:

```bash
id
docker compose version
docker version
git --version
findmnt -T /mnt/data -o TARGET,SOURCE,FSTYPE,OPTIONS
df -h /mnt/data
```

On the recorded host, `/mnt/data` was a writable NTFS (`fuseblk`) mount, with about 4 TB free. Do **not** change the disk-wide mount options, ownership or Samba permissions just for this deployment. Verify the intended host user can create a file:

```bash
mkdir -p /mnt/data/Music/Spotify
touch /mnt/data/Music/Spotify/.ots-write-test
rm /mnt/data/Music/Spotify/.ots-write-test
echo 'WRITE TEST: OK'
```

If the test fails, investigate the existing filesystem mount/user permissions first; do not run a recursive `chmod` or `chown` against the NTFS disk.

## 3. Create the stack on the system SSD

Create the folders with the host user's ownership. Example for the recorded `piroman` account:

```bash
sudo install -d -o piroman -g piroman -m 755 /srv/docker/onthespot
cd /srv/docker/onthespot
mkdir -p otsdata/Config otsdata/Videos
chmod 700 otsdata/Config
```

Use a regular (non-`sudo`) editor for files in this directory so they remain readable by the Docker-running account. `.env` should be mode `600` because it may eventually contain sensitive configuration.

### 3.1 Retrieve the *fixed* source release

```bash
cd /srv/docker/onthespot
git clone --depth 1 --branch v2.0.2beta4 \
  https://github.com/ots-downloader/onthespot.git source
git -C source describe --tags --exact-match
git -C source rev-parse --short HEAD
```

Expected: `v2.0.2beta4` and `48f6061`. The detached HEAD notice is normal for a tag checkout.

**Why build locally?** The checked Docker Hub repository `moddroid94/ots-dl` did not publish the requested `v2.0.2beta4` manifest (`no such manifest`). Its moving `latest` tag was not a safe substitute for the required 2.x Web UI. The upstream tag includes a multi-stage Node/React + Python/FastAPI Dockerfile.

### 3.2 Build, diagnose and apply the local permission fix

```bash
docker build -t onthespot:v2.0.2beta4 ./source
```

The unmodified image on this host failed at runtime with `Could not import module "onthespot.main"`. The root cause was confirmed inside the container: `/app/app/onthespot` was `root:root` mode `700`, and `main.py` was mode `600`, but Uvicorn ran as UID 1000. Running `sudo docker compose` on the **host** does not change the user **inside** the container.

Preserve upstream code and make a small, local overlay. Create `/srv/docker/onthespot/Dockerfile.local`:

```dockerfile
FROM onthespot:v2.0.2beta4

USER root
RUN chown -R 1000:1000 /app/app
USER 1000:1000
```

Then build the corrected image:

```bash
cd /srv/docker/onthespot
docker build -f Dockerfile.local -t onthespot:v2.0.2beta4-local1 .
docker image inspect onthespot:v2.0.2beta4-local1 --format '{{.Id}}'
```

**Import acceptance gate — do not start Compose before this passes:**

```bash
docker run --rm \
  --entrypoint /app/.venv/bin/python \
  onthespot:v2.0.2beta4-local1 \
  -c 'import sys; sys.path.insert(0, "/app/app"); import onthespot.main; print("IMPORT OK")'
```

Expected final line: `IMPORT OK`. A one-time message about moving accounts into the encrypted credential store can also appear.

### 3.3 Configure `.env`

Create `/srv/docker/onthespot/.env` with the following **host-specific, nonsensitive values**:

```dotenv
ONTHESPOT_IMAGE=onthespot:v2.0.2beta4-local1
ONTHESPOT_WEB_PORT=6767
ONTHESPOT_MUSIC_DIR=/mnt/data/Music/Spotify
ONTHESPOT_VIDEO_DIR=./otsdata/Videos
ONTHESPOT_CONFIG_DIR=./otsdata/Config
```

```bash
chmod 600 /srv/docker/onthespot/.env
```

**Why `.env`?** Changing a host port, the music disk or the pinned image does not require modifying Compose YAML. Keep the real file and future account/session material **on the host only**.

### 3.4 Create `compose.yml`

This is a **reproducible equivalent** of the deployed one-service setup, adapted from the official Compose service. It deliberately omits the optional YouTube helper container. Use this as the starting point on a fresh server; the exact host-local production Compose file is not committed.

```yaml
services:
  onthespot:
    image: ${ONTHESPOT_IMAGE}
    pull_policy: never
    container_name: onthespot
    user: "1000:1000"
    ports:
      - "${ONTHESPOT_WEB_PORT}:6767"
    environment:
      ONTHESPOTDIR: /root/.config/onthespot
      ONTHESPOTCACHEDIR: /root/.config/onthespot/cache
    volumes:
      - "${ONTHESPOT_MUSIC_DIR}:/root/Music/OnTheSpot"
      - "${ONTHESPOT_VIDEO_DIR}:/root/Videos/OnTheSpot"
      - "${ONTHESPOT_CONFIG_DIR}:/root/.config/onthespot"
    restart: unless-stopped
```

The `/root/...` paths are **inside the image**, intentionally matching upstream environment variables and Dockerfile ownership setup; the real music and app-data locations are the host-side bind mounts. `pull_policy: never` avoids any registry fetch: a missing local image is an error, not a surprise upgrade. The host port is published on all host interfaces in this as-built setup; do not configure new router port forwarding. If reproducing in a different network, assess firewall/binding policy separately.

### 3.5 Validate and start **only OnTheSpot**

```bash
cd /srv/docker/onthespot
docker compose config --quiet
docker compose config --images
docker compose config | grep -E 'image:|pull_policy:|/mnt/data/Music/Spotify'
docker compose up -d --no-deps --pull never onthespot
docker compose ps
docker compose logs --tail=80 onthespot
curl -sS -o /dev/null -w 'HTTP %{http_code}\n' http://127.0.0.1:6767/
```

Expected image: `onthespot:v2.0.2beta4-local1`; pull policy: `never`; HTTP: `200`; browser at `http://192.168.0.10:6767` shows `v2.0.2 Beta 4` and `FastAPI Core Ready`. Confirm the **Web UI**, not merely the container's `Up` status: the original broken image also appeared to start while logging repeated ASGI errors.

## 4. Publish through the existing reverse proxy

In **Nginx Proxy Manager → Access Lists**, create `LAN Only`:

| Setting | Value |
| --- | --- |
| Name | `LAN Only` |
| Satisfy Any | Off |
| Pass Auth to Upstream | Off |
| Authorizations | No users / remove empty placeholder row |
| Rules, first | Allow `192.168.0.0/24` |
| Rules, final | Deny `all` (NPM may append automatically) |

**Why:** The beta Web UI is not treated as an independently authenticated public endpoint. The access list limits requests **through NPM**; it does **not** secure direct host port `6767`. The host is behind NAT; no new router forwarding was configured.

Then create **Hosts → Proxy Hosts → Add Proxy Host**:

| Field | Value |
| --- | --- |
| Domain Names | `onthespot.pirocorp.com` |
| Scheme | `http` |
| Forward Hostname/IP | `192.168.0.10` |
| Forward Port | `6767` |
| Access List | `LAN Only` |
| Cache Assets | Off |
| Block Common Exploits | On |
| Websockets Support | On |
| SSL Certificate | Existing `pirocorp.com, *.pirocorp.com` |
| Force SSL / HTTP/2 | On / On |

Save and open `https://onthespot.pirocorp.com` on a LAN client. The screenshot on 2026-10-26 showed the UI running through HTTPS. External-origin access-list testing was deliberately skipped for this NAT-protected deployment; do not claim it was tested.

## 5. DNS / AdGuard — verify before adding rules

The DNS answer for `onthespot.pirocorp.com` already resolved to `192.168.0.10`, so **no OnTheSpot-specific rewrite was added**. Before making changes, check from Windows PowerShell:

```powershell
nslookup onthespot.pirocorp.com
nslookup onthespot.pirocorp.com 192.168.0.10
```

Both tests returned the private server address. The default DNS resolver in the observed Windows test was Telenet; the direct query to `192.168.0.10` also worked. Windows intentionally has both AdGuard and Telenet DNS configured because router DHCP is not controlled by the user. Do not change this accepted arrangement; multiple configured resolvers do **not** necessarily implement strict primary/fallback ordering.

**On a different installation, only if the name does not resolve correctly**, add a DNS rewrite in AdGuard Home for `onthespot.pirocorp.com` → the NPM host's private LAN IP. Avoid a duplicate rewrite when existing wildcard or authoritative DNS already works.

## 6. Add to Homepage without disturbing existing cards

Back up the existing configuration on the host first:

```bash
sudo cp -a /srv/docker/homepage/config/services.yaml \
  /srv/docker/homepage/config/services.yaml.bak-$(date +%Y%m%d-%H%M%S)
```

In `/srv/docker/homepage/config/services.yaml`, find the existing `Downloads & Discovery` group and add **one service item**, leaving its siblings and all other groups intact:

```yaml
    - OnTheSpot:
        icon: mdi-music-circle
        href: https://onthespot.pirocorp.com
        description: Spotify Music Downloader
        target: _blank
```

**Why this group?** OnTheSpot downloads music; `Media & Libraries` is for playback and libraries. No widget, monitoring endpoint or extra container is required. A green status indicator was **not** part of this acceptance.

Verify in the browser at `https://home.pirocorp.com`. The card appeared under `Downloads & Discovery` in the recorded acceptance screenshot; no Homepage restart was needed.

## 7. Acceptance record, operations and troubleshooting

### Infrastructure acceptance (observed 2026-10-26)

| Check | Result |
| --- | --- |
| NTFS music directory writable by UID 1000 host user | **PASS** — `WRITE TEST: OK` |
| Exact upstream Git tag / commit | **PASS** — `v2.0.2beta4` / `48f6061` |
| Local pinned image created | **PASS** — `onthespot:v2.0.2beta4-local1` |
| Python import after permissions overlay | **PASS** — `IMPORT OK` |
| Compose validation, image and no-pull policy | **PASS** |
| FastAPI/Web UI on port 6767 | **PASS** — HTTP 200 |
| HTTPS via NPM | **PASS** — UI displayed |
| DNS answer from host DNS service | **PASS** — `192.168.0.10` |
| Homepage card in correct group | **PASS** |
| External-origin Access List test | Not performed by decision |
| Spotify Premium account, track download and Artist/Album filename layout | **PENDING** — separate application phase |

### Routine operations

```bash
cd /srv/docker/onthespot
docker compose config --quiet
docker compose ps
docker compose logs --tail=80 onthespot
docker compose up -d --no-deps --pull never onthespot
```

Update deliberately, **never via `latest`**: review the new upstream version; build a new versioned image and, if needed, a new overlay; verify import and Web UI; back up persistent config; then change only `ONTHESPOT_IMAGE` in the host-local `.env` and recreate this service. A local pinned tag is independent of Docker Hub availability. Do not use `docker compose pull` for this image.

### Common failures

| Symptom | Cause / next diagnostic step |
| --- | --- |
| `docker compose config` → permission denied opening `.env` | Files created with `sudo nano` may be `root:root` mode `600`; inspect `ls -l`, then adjust **only** the stack files' ownership to the host user. |
| Docker Hub → `no such manifest` | Requested beta tag was not published there; build from the pinned Git tag as above. |
| `Could not import module "onthespot.main"` | Verify `/app/app/onthespot` ownership/mode in the **image**, then rebuild the local overlay; host `sudo` alone does not help. |
| Container `Up`, browser fails | Check `docker compose logs` and `curl`; process may be restarting or returning errors. |
| Writing music fails | Inspect bind mounts with `docker compose config`, test the host directory as UID 1000, and check NTFS mount permissions; do not recursively change the whole disk. |
| HTTPS address fails but `:6767` works | Inspect NPM proxy host/access list, certificate and DNS answer. |
| Homepage card absent | Check YAML indentation under `Downloads & Discovery` and restore the timestamped backup if needed. |

### Rollback / boundary

To stop **only** this app without deleting music or settings:

```bash
cd /srv/docker/onthespot
docker compose stop onthespot
```

If retiring it, remove or disable **only** the OnTheSpot NPM proxy host and Homepage card, and restore the pre-edit Homepage `services.yaml` backup if necessary. Preserve `/srv/docker/onthespot/otsdata/Config` and `/mnt/data/Music/Spotify` until separately reviewed for deletion. Do not remove shared NPM/AdGuard/Homepage services or change the NTFS mount.

This repository contains **documentation and sanitized example configuration only**: no running-host Compose files, real `.env`, passwords, tokens, music, images or other private app state were uploaded. This PR itself makes **no production changes**.
