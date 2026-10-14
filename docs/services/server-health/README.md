# Server Health Dashboard — deployed runbook (Phase 1)

Status: **Implemented and accepted — 2026-10-14**  
Purpose: Reproduce the deployed **Homepage + Glances + PeaNUT** solution for Ubuntu host resources, six filesystem-capacity indicators and PowerWalker UPS visibility, and explain its trade-offs and tested failure fixes.  
Depends on: [Homepage](../homepage/README.md), [host NUT UPS](../ups-monitoring/README.md), [networking and reverse proxy](../../platform/networking-and-reverse-proxy.md).  
Decision history: [Phase-1 architecture roadmap](../../roadmaps/server-health-dashboard/README.md). Future work: [separate Phase-2 SMART plan](../../roadmaps/server-health-dashboard/smart-extension.md).

> This is **as-built documentation**, not a script that was executed by GitHub. The configuration examples below reconstruct the successful 2026-10-14 deployment from the validated Compose settings and interactive tests. Always compare the local files, paths, IP ranges and images with your host before reproducing. Never commit actual credentials.

## 1. What we built and why

The requirement was a **single simple, low-maintenance Homepage dashboard** for real host CPU/RAM/CPU temperature/uptime, filesystem usage on six mounts and the basic UPS state. Detailed observability history and the Netdata alerting platform were not required for this deployment.

| Role | Implementation | Tested version | Endpoint |
| --- | --- | --- | --- |
| User interface and existing Docker status | Existing Homepage and private Docker Socket Proxy | Homepage `v2.4.0` | `https://home.pirocorp.com`, host `192.168.0.10:3002` |
| Host CPU, RAM, temperature, uptime and space | Glances in its own Docker Compose stack | `nicolargo/glances:4.5.6` | `https://glances.pirocorp.com`, API `:61208` |
| UPS visualisation | PeaNUT in its own Compose stack | `brandawg93/peanut:6.0.0` | `https://peanut.pirocorp.com`, API `:61210` |
| **UPS shutdown authority** | **Existing NUT on Ubuntu, not in Docker** | Existing installation unchanged | `powerwalker@127.0.0.1:3493` |
| Reverse proxy, certificates, DNS | Existing NPM, wildcard Let's Encrypt and AdGuard Home | Existing installation unchanged | Explicit per-host DNS rewrites |
| Superseded monitoring | Netdata native Ubuntu APT installation | Removed 2026-10-14 | Old `server.pirocorp.com` / `:19999` removed |

```text
Trusted browser (LAN / trusted VPN)
  |--> HTTPS home.pirocorp.com --> NPM --> Homepage :3002
  |                                    |--> http://glances:61208 (Docker bridge)
  |                                    '--> http://192.168.0.10:61210 (UFW)
  |                                                |
  |                                                v
  |                                              PeaNUT (host networking)
  |                                                |
  |                                                v
  |                                         NUT 127.0.0.1:3493 --> PowerWalker
  |
  |--> HTTPS glances.pirocorp.com --> NPM --> 192.168.0.10:61208
  '--> HTTPS peanut.pirocorp.com  --> NPM --> 192.168.0.10:61210

NUT + upsmon handle safe shutdown independently of Homepage, Glances and PeaNUT.
```

**Architecture choices:**

- Glances uses **Docker bridge** and joins the already-existing `homepage_default` network, NOT `homepage_docker_api`. That private network is reserved for the Docker Socket Proxy. Glances never mounts the Docker socket or writes to monitored disks.
- PeaNUT deliberately uses **`network_mode: host`**. NUT only listens on the host's localhost; a normal bridge-container localhost would refer to the container, not the Ubuntu NUT daemon. We considered exposing NUT on a host/bridge/LAN address and rejected it: changing a safety-critical UPS shutdown configuration for a dashboard was unnecessary.
- PeaNUT listens only on Ubuntu's **`192.168.0.10:61210`** and requires authentication; UFW selectively allows LAN, NPM and Homepage to reach it. `3493` stays loopback-only; there are no router forwards for either monitoring API.
- Each application's HTTPS URL is terminated by the existing NPM wildcard certificate. The **browser-facing `href` is not the server-to-server API `url`** in Homepage.
- Homepage uses its **existing file-backed `HOMEPAGE_FILE_*` secret convention**. No `.env` with credentials is needed for Homepage; the two new stacks use `.env` only to pin image versions.
- PeaNUT is a management-capable application with command API endpoints. Its web/API authentication **does not imply that the API is read-only**; do not configure NUT shutdown/admin credentials in PeaNUT merely for monitoring.
- Netdata was stopped and purged **only after the user accepted the replacement**. Detailed histories and Netdata alerts are intentionally not replicated.

## 2. Preconditions and observed host topology

```bash
docker network inspect homepage_default --format '{{range .IPAM.Config}}{{.Subnet}} {{.Gateway}}{{end}}'
docker network inspect nginx-proxy-manager_default --format '{{range .IPAM.Config}}{{.Subnet}}{{end}}'
sudo ss -lntp '( sport = :3493 or sport = :61208 or sport = :61210 )'
sudo grep -E '^[[:space:]]*LISTEN' /etc/nut/upsd.conf || true
upsc -l
upsc powerwalker@127.0.0.1 ups.status
```

Observed networks: `homepage_default = 172.31.0.0/16` (gateway `172.31.0.1`); `nginx-proxy-manager_default = 172.20.0.0/16`. Actual Homepage container `172.31.0.2`, NPM `172.20.0.2` on the test day; **do not depend on those container IPs**. Host `192.168.0.10` is the Ubuntu server's LAN address.

The absence of an explicit `LISTEN` line in `/etc/nut/upsd.conf` was observed. NUT served loopback; verify with `ss` on the target machine. `upsc` may print `Init SSL without certificate database` before valid readings, which did not prevent monitoring.

**Mounts** (system capacity is shown via a read-only path on the root filesystem):

| Display tile | Ubuntu path | Glances API alias / path |
| --- | --- | --- |
| SYSTEM SSD | `/` (backing FS of `/srv/docker/glances`) | `System SSD` / `/host-system` |
| LP | `/mnt/lp` | `lp` / `/mnt/lp` |
| DATA | `/mnt/data` | `data` / `/mnt/data` |
| IA | `/mnt/ia` | `ia` / `/mnt/ia` |
| COMP | `/mnt/comp` | `comp` / `/mnt/comp` |
| IAC | `/mnt/iac` | `iac` / `/mnt/iac` |

The five data volumes are NTFS mounted via Samba for Windows administration; **do not** change their filesystem, mount options, ownership or sharing to run these monitoring services. The system SSD usage refers to the underlying root filesystem; the bind mount intentionally **does not expose the whole host root tree**.

## 3. Reproduce Glances

Prepare the independent stack (ports and directories are specific to this deployment):

```bash
sudo mkdir -p /srv/docker/glances
cd /srv/docker/glances
sudo ss -lntp '( sport = :61208 )'   # must be free before first deployment
```

`/srv/docker/glances/.env`:

```dotenv
GLANCES_VERSION=4.5.6
```

`/srv/docker/glances/compose.yaml`:

```yaml
services:
  glances:
    image: nicolargo/glances:${GLANCES_VERSION}
    container_name: glances
    restart: unless-stopped

    ports:
      - "192.168.0.10:61208:61208"

    environment:
      GLANCES_OPT: "-w -C /etc/glances/glances.conf"

    read_only: true
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true

    volumes:
      - /srv/docker/glances:/host-system:ro
      - /mnt/iac:/mnt/iac:ro
      - /mnt/lp:/mnt/lp:ro
      - /mnt/data:/mnt/data:ro
      - /mnt/ia:/mnt/ia:ro
      - /mnt/comp:/mnt/comp:ro
      - /etc/os-release:/etc/os-release:ro
      - ./glances.conf:/etc/glances/glances.conf:ro

    tmpfs:
      - /tmp

    networks:
      - homepage

networks:
  homepage:
    external: true
    name: homepage_default
```

`/srv/docker/glances/glances.conf`:

```ini
[fs]
show=/host-system,/mnt/iac,/mnt/lp,/mnt/data,/mnt/ia,/mnt/comp
alias=/host-system:System SSD,/mnt/iac:iac,/mnt/lp:lp,/mnt/data:data,/mnt/ia:ia,/mnt/comp:comp
```

Important: mount `/srv/docker/glances` on the **same root filesystem** as `/`; otherwise `/host-system` will report the capacity of a different filesystem. Glances's FS reporting concerns mounted **filesystem capacity**, not directory size.

Start and test:

```bash
cd /srv/docker/glances
sudo docker compose config --quiet
sudo docker compose pull
sudo docker compose up -d
sudo docker compose ps
curl -fsS http://192.168.0.10:61208/api/4/fs
sudo docker exec homepage node -e 'fetch("http://glances:61208/api/4/fs").then(r=>r.json()).then(x=>console.log(x.map(v=>[v.mnt_point,v.percent])))'
df -h / /mnt/iac /mnt/lp /mnt/data /mnt/ia /mnt/comp
```

**Validation performed:** six FS records; host CPU/RAM/`coretemp Package id 0` temperature about 37°C and uptime appeared in Homepage, with plausible CPU percentages after warm-up. One early `0.0%` CPU result was transient; a later sample series showed activity. The container process list did not represent the full host task tree; do not claim that process-level visibility is complete.

## 4. Add Glances to Homepage, retaining the existing layout

In `/srv/docker/homepage/config/widgets.yaml` (existing file previously `[]`), deploy the info strip:

```yaml
- glances:
    url: http://glances:61208
    href: https://glances.pirocorp.com
    version: 4
    cpu: true
    mem: true
    cputemp: true
    uptime: true
    expanded: true
    label: Server
```

The `url` is internal Docker DNS; `href` is the URL users open in their browser.

In the **existing** `Infrastructure` group of `/srv/docker/homepage/config/services.yaml`, insert **one** `Storage` card with **two native customapi widgets** (no CSS, no custom JavaScript):

```yaml
    - Storage:
        href: https://glances.pirocorp.com
        description: Disk space · 6 filesystems
        widgets:
          - type: customapi
            url: http://glances:61208/api/4/fs
            display: block
            refreshInterval: 30000
            mappings:
              - field: "0.percent"
                label: SYSTEM SSD
                format: percent
              - field: "1.percent"
                label: LP
                format: percent
              - field: "2.percent"
                label: DATA
                format: percent
          - type: customapi
            url: http://glances:61208/api/4/fs
            display: block
            refreshInterval: 30000
            mappings:
              - field: "3.percent"
                label: IA
                format: percent
              - field: "4.percent"
                label: COMP
                format: percent
              - field: "5.percent"
                label: IAC
                format: percent
```

**Critical caveat: positional mapping.** These six labels reflect the **observed order on the deployed `/api/4/fs` response**, not a guaranteed API sorting contract and **not** the order written in `show=`. Re-check `mnt_point`/`alias` vs index **after every Glances restart, update or filesystem change**, otherwise tiles can silently display the right percentages under the **wrong disk labels**. This native six-tile layout was preferred and accepted, but it is not a stable-identity mapping. A future identity-based fix is separate work; do not misrepresent the indexed layout as permanently stable.

## 5. Reproduce PeaNUT without disturbing NUT

**Do not edit** `/etc/nut/upsd.conf`, `/etc/nut/upsmon.conf`, `/etc/nut/upsd.users`, `/etc/nut/ups.conf` or the NUT driver as part of this deployment. NUT owns safe shutdown.

```bash
sudo mkdir -p /srv/docker/peanut
sudo install -d -m 700 -o 1000 -g 1000 /srv/docker/peanut/config
cd /srv/docker/peanut
sudo ss -lntp '( sport = :61210 )'   # must be free
upsc powerwalker@127.0.0.1 battery.charge
upsc powerwalker@127.0.0.1 ups.load
upsc powerwalker@127.0.0.1 ups.status
```

`/srv/docker/peanut/.env` contains **only** the image pin:

```dotenv
PEANUT_VERSION=6.0.0
```

`/srv/docker/peanut/compose.yaml` (final configuration, including both verified v6 fixes):

```yaml
services:
  peanut:
    image: brandawg93/peanut:${PEANUT_VERSION}
    container_name: peanut
    restart: unless-stopped

    network_mode: host
    user: "1000:1000"

    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true

    environment:
      WEB_HOST: "192.168.0.10"
      WEB_PORT: "61210"
      AUTH_DISABLED: "false"
      AUTH_FILE_PATH: "/config/auth.yaml"
      NUT_HOST: "127.0.0.1"
      NUT_PORT: "3493"

    volumes:
      - ./config:/config

    healthcheck:
      test:
        - CMD
        - node
        - -e
        - "fetch('http://' + process.env.WEB_HOST + ':' + process.env.WEB_PORT + '/api/ping').then(r => process.exit(r.ok ? 0 : 1)).catch(() => process.exit(1))"
      interval: 30s
      timeout: 5s
      start_period: 10s
      retries: 3
```

**Why the overrides?**

1. `network_mode: host` is the minimal solution for an otherwise inaccessible host-local NUT `127.0.0.1:3493`. No `ports:` entry is valid/needed in host mode.
2. PeaNUT 6.0.0's built-in healthcheck targets `127.0.0.1:WEB_PORT`, but the intentionally LAN-bound UI listens at `192.168.0.10:61210`. Docker initially showed `unhealthy` **even while `GET /api/ping` returned 200**. Override the healthcheck target with `process.env.WEB_HOST`.
3. PeaNUT v6's default credential storage can resolve to `/app/config/auth.yaml` (inside its ephemeral container), not the mounted `/config`. Set **`AUTH_FILE_PATH=/config/auth.yaml` before first account creation** so credentials persist. This was observed in the live installation and matches [upstream issue #423](https://github.com/Brandawg93/PeaNUT/issues/423).

```bash
cd /srv/docker/peanut
sudo docker compose config --quiet
sudo docker compose pull
sudo docker compose up -d
sudo docker compose ps
curl -fsS http://192.168.0.10:61210/api/ping  # expects "pong"
```

Create the PeaNUT account **through HTTPS after section 6 is configured**, not on plain HTTP, then inspect its device page for `powerwalker` and live charge/load/`OL CHRG` (Online + Charging).

### Existing PeaNUT 6.0.0 install: one-time credential-file migration

Only if an existing v6 container **already wrote** `/app/config/auth.yaml` and `/config/auth.yaml` does not exist, copy the credential hash **before recreating the container**:

```bash
sudo docker exec peanut node -e 'const fs=require("node:fs"); for(const p of ["/app/config/auth.yaml","/config/auth.yaml"]) console.log(fs.existsSync(p)?"EXISTS":"MISSING",p)'
sudo docker cp peanut:/app/config/auth.yaml /srv/docker/peanut/config/auth.yaml
sudo chown 1000:1000 /srv/docker/peanut/config/auth.yaml
sudo chmod 600 /srv/docker/peanut/config/auth.yaml
# Add AUTH_FILE_PATH: "/config/auth.yaml" to Compose; recreate PeaNUT.
cd /srv/docker/peanut && sudo docker compose up -d
sudo docker exec peanut node -e 'console.log(process.env.AUTH_FILE_PATH,require("node:fs").existsSync(process.env.AUTH_FILE_PATH))'
```

Never print `auth.yaml`: it contains the bcrypt password hash. The migration was **actually performed and verified** in the live setup; API Basic authentication changed from `401` to `200` afterwards. This observed outcome does **not** prove every future `401` is caused by the storage path alone.

## 6. NPM, AdGuard and UFW — actual network ingress

**Explicit local AdGuard DNS rewrites:**

```text
glances.pirocorp.com -> 192.168.0.10
peanut.pirocorp.com  -> 192.168.0.10
```

No wildcard rewrite, router forward or new public-facing tunnel was introduced.

**Nginx Proxy Manager Proxy Hosts**, using the **existing** wildcard certificate for `pirocorp.com, *.pirocorp.com`:

| Domain | Scheme | Forward hostname/IP | Port | Settings |
| --- | --- | --- | --- | --- |
| `glances.pirocorp.com` | HTTP | `192.168.0.10` | `61208` | Force SSL, existing wildcard cert |
| `peanut.pirocorp.com` | HTTP | `192.168.0.10` | `61210` | Force SSL, HTTP/2, Block Common Exploits, WebSockets ON, cache OFF, HSTS off |

Homepage's existing proxy route to `192.168.0.10:3002` remains unchanged. Use `https://peanut.pirocorp.com`, **not** `http://peanut.pirocorp.com:61210`, to create or enter the PeaNUT password.

**UFW access, confirmed after moving away from container-specific IPs** (adjust local subnet/IPs if recreating elsewhere):

```bash
sudo ufw allow in from 192.168.0.0/24 to 192.168.0.10 port 61210 proto tcp comment 'PeaNUT LAN'
sudo ufw allow in from 172.20.0.0/16  to 192.168.0.10 port 61210 proto tcp comment 'PeaNUT NPM Docker network'
sudo ufw allow in from 172.31.0.0/16  to 192.168.0.10 port 61210 proto tcp comment 'PeaNUT Homepage Docker network'
sudo ufw status numbered | grep 61210
```

The original container-specific rules (`172.20.0.2` and `172.31.0.2`) were removed after the subnet rules were installed. **Trade-off:** any container attached to those two networks can reach PeaNUT's HTTP port, not exclusively NPM/Homepage; authentication still applies. These rules work only as long as the Docker subnet allocation remains the same; confirm on host rebuild. For Glances `:61208`, Docker's published port follows its existing bridge/NAT behavior. Do not generalize this to host-mode services: Ubuntu UFW default-deny explicitly blocked PeaNUT until rules were added.

NUT `3493` must continue to listen only on loopback; no firewall rule or LAN listener was added for NUT.

## 7. Homepage UPS card and file-backed secrets

Retain the existing `/srv/docker/homepage` stack, Docker Socket Proxy, and current cards. The two PeaNUT credentials are **files**, not `.env` entries.

In the existing Homepage service's `compose.yaml` `environment:` block, add:

```yaml
      HOMEPAGE_FILE_PEANUT_USERNAME: /app/secrets/peanut_username
      HOMEPAGE_FILE_PEANUT_PASSWORD: /app/secrets/peanut_password
```

The existing **read-only mount** remains:

```yaml
    volumes:
      - ./config:/app/config
      - ./secrets:/app/secrets:ro
```

Create the two files with no terminating newline. The password prompt below avoids putting the secret on the command line or in shell history:

```bash
sudo install -o 1000 -g 1000 -m 600 /dev/null /srv/docker/homepage/secrets/peanut_username
sudo install -o 1000 -g 1000 -m 600 /dev/null /srv/docker/homepage/secrets/peanut_password
printf %s 'admin' | sudo tee /srv/docker/homepage/secrets/peanut_username >/dev/null
read -rsp 'PeaNUT password: ' PEANUT_PASS; echo
printf %s "$PEANUT_PASS" | sudo tee /srv/docker/homepage/secrets/peanut_password >/dev/null
unset PEANUT_PASS
sudo chmod 600 /srv/docker/homepage/secrets/peanut_*
```

`/srv/docker/homepage/config/services.yaml`, **inside** the current `Infrastructure` group, alongside its other cards (do not create a duplicate `Infrastructure` group):

```yaml
    - UPS:
        icon: peanut.png
        href: https://peanut.pirocorp.com
        description: PowerWalker UPS
        widget:
          type: peanut
          url: http://192.168.0.10:61210
          key: powerwalker
          username: "{{HOMEPAGE_FILE_PEANUT_USERNAME}}"
          password: "{{HOMEPAGE_FILE_PEANUT_PASSWORD}}"
```

Apply:

```bash
cd /srv/docker/homepage
sudo docker compose config --quiet
sudo docker compose up -d
```

**Verified v2.4.0 secret gotcha:** Homepage substitutes `HOMEPAGE_FILE_*` values by reading **the exact file bytes**, without trimming `\n`. The live initial attempt used files edited in Nano with trailing newlines; Homepage returned `401 Unauthorized` even though interactive `curl -u admin` returned `200`. We reproduced:

```text
USERNAME trailing newline: true
PASSWORD trailing newline: true
Raw credentials HTTP: 401
Without trailing newlines HTTP: 200
```

After stripping **only trailing CR/LF** from both files, raw credentials returned `200`. A second `401` persisted until `services.yaml` was corrected from **old** `HOMEPAGE_VAR_PEANUT_*` placeholders to the **new** `HOMEPAGE_FILE_PEANUT_*` placeholders. This was the final issue; the UPS tile rendered `100%` charge, `10%` load and `OL CHRG`.

To verify bytes **without printing secrets**:

```bash
sudo docker exec homepage node -e '
const fs=require("node:fs");
for(const k of ["USERNAME","PASSWORD"]){
  const p=process.env["HOMEPAGE_FILE_PEANUT_"+k];
  if(!p){console.log(k,"ENV MISSING");continue}
  const v=fs.readFileSync(p,"utf8");
  console.log(k,"configured:",!!v,"trailing newline:",/[\r\n]$/.test(v));
}'
```

To repair terminating newlines if necessary:

```bash
sudo python3 - <<'PY'
from pathlib import Path
for name in ("peanut_username", "peanut_password"):
    p = Path("/srv/docker/homepage/secrets") / name
    p.write_bytes(p.read_bytes().rstrip(b"\r\n"))
PY
cd /srv/docker/homepage && sudo docker compose restart homepage
```

Do not print the values via `docker compose config` without `--quiet` or commit `secrets/`.

## 8. Verification and operational checks

```bash
# A. NUT / shutdown dependency still works independently:
systemctl is-active nut-server
upsc powerwalker@127.0.0.1 ups.status
upsc powerwalker@127.0.0.1 battery.charge
upsc powerwalker@127.0.0.1 ups.load

# B. Containers are working:
cd /srv/docker/glances && sudo docker compose ps
cd /srv/docker/peanut  && sudo docker compose ps
sudo docker inspect peanut --format '{{.State.Health.Status}}'

# C. Each origin is reachable:
curl -fsS http://192.168.0.10:61210/api/ping
curl -fsS http://192.168.0.10:61208/api/4/fs

# D. Homepage container's route across UFW:
sudo docker exec homepage node -e 'fetch("http://192.168.0.10:61210/api/ping",{signal:AbortSignal.timeout(5000)}).then(async r=>console.log(r.status,await r.text()))'

# E. Check filesystem labels against actual Glances API order:
curl -fsS http://192.168.0.10:61208/api/4/fs

# F. Ensure NUT is not newly exposed to LAN:
sudo ss -lntp '( sport = :3493 )'

# G. Check UFW for the expected three PeaNUT rules:
sudo ufw status numbered | grep 61210
```

PeaNUT authenticated read check (`curl` prompts for the account password; do not post it):

```bash
curl -sS -o /dev/null -w 'HTTP %{http_code}\n' -u admin http://192.168.0.10:61210/api/v1/devices/powerwalker
# expected: HTTP 200
```

Browser acceptance (desktop observed): `https://home.pirocorp.com` displays the server strip, six compact usage tiles and the three-field UPS tile; `https://glances.pirocorp.com` loads the full Glances UI; `https://peanut.pirocorp.com` loads the protected PeaNUT UI. **An iPhone-specific screenshot/acceptance was not established in these logs**; do not imply separately validated mobile layout.

PowerWalker observed `battery.charge=100`, `ups.load=8–10`, `ups.status=OL CHRG` across verification. Treat these as historical snapshots, not constants.

### Interpreting swap correctly

During retirement preflight: host RAM about `30 GiB`, available about `22 GiB`; swap about `5.7/8 GiB` used. `vmstat 1 5` showed `si=so=0` in the four fresh samples and negligible I/O wait. The first `vmstat` row is a since-boot average. High swap *occupancy* alone did not demonstrate active swap thrashing; do not run `swapoff -a` reflexively. Glances could display `MEMSWAP` warnings while system resources were otherwise responsive.

## 9. Netdata retirement — actually executed after acceptance

The old Netdata was a **native Ubuntu systemd/official APT nightly** install, not a Docker container. The recorded order was:

1. Verify Glances, PeaNUT and Homepage **while Netdata is still running**, check the functional data needed by this homelab, and explicitly accept loss of Netdata historical charts and alerting.
2. `sudo systemctl stop netdata`; observe `inactive` and port `19999` closed.
3. `sudo systemctl disable netdata`; verify `inactive / disabled`.
4. Remove the `Server / Netdata` Homepage service card. Remove the old NPM host `server.pirocorp.com` and its **explicit** AdGuard rewrite. Remove UFW `19999/tcp` rules (IPv4/IPv6).
5. List **only installed Netdata packages**, simulate, then purge. Do not blindly purge unrelated packages:

```bash
mapfile -t packages < <(
  dpkg-query -W -f='${db:Status-Abbrev} ${binary:Package}\n' 'netdata*' 2>/dev/null |
    awk '$1 == "ii" {print $2}'
)
printf '%s\n' "Netdata packages:" "${packages[@]}"
sudo apt-get -s purge "${packages[@]}"
# Confirm the simulated removal list contains only Netdata packages.
sudo apt-get purge "${packages[@]}"
```

The actual simulation removed **19 Netdata-only packages**, including `netdata-repo-edge`, and the purge freed an APT-reported **513 MB**. APT then reported only `libnetfilter-acct1` as no longer required; separately simulated and purged it (33.8 kB). There was **no broad `apt autoremove`**.

6. After verifying no installed Netdata packages and no unit on disk, identify generated leftovers. The observed `/etc/netdata/go.d/nut.conf` belonged to Netdata's NUT collector, **not** to NUT itself; the remaining `/etc/netdata` (12 KB) and `/var/log/netdata` (32 MB) were removed. Do not remove `/etc/nut`.

```bash
dpkg -l 'netdata*' 2>/dev/null | grep '^ii' || echo 'No installed Netdata packages'
systemctl list-unit-files 'netdata*' --no-pager
sudo ss -lntp '( sport = :19999 )'
sudo ls -ld /etc/netdata /var/lib/netdata /var/cache/netdata /var/log/netdata 2>/dev/null || true
systemctl is-active nut-server
upsc powerwalker@127.0.0.1 ups.status
```

Final actual result: zero installed Netdata packages, zero Netdata unit files, no listener `:19999`, no remaining `/etc/netdata`/`/var/lib/netdata`/`/var/cache/netdata`/`/var/log/netdata` directories; NUT remained `active` and `OL CHRG`. The former `server.pirocorp.com` route is **retired and should not be reintroduced**.

**Alert/history trade-off:** an old Netdata warning was visible before retirement, but its exact alert rule/root cause was **not conclusively captured** in the available acceptance output. The user explicitly accepted the simpler monitoring scope; this change must **not** be described as full Netdata alert parity or a completed forensic review of that warning.

## 10. Troubleshooting and rollback

| Symptom | Check / proven resolution |
| --- | --- |
| PeaNUT local `curl /api/ping` = `200`, browser timeout | Host mode traffic goes through Ubuntu UFW. Add **LAN-only** TCP `61210` rule. |
| HTTPS DNS resolves/443 connects, NPM route hangs | NPM container (observed `172.20.0.2`) was blocked by UFW. Test from NPM and permit **its network** `172.20.0.0/16` to `192.168.0.10:61210`. |
| Homepage PeaNUT ping timeout | Homepage network `172.31.0.0/16` was blocked by UFW; permit it to `:61210`. |
| PeaNUT `unhealthy` despite working web UI | Built-in healthcheck targets `127.0.0.1`; override with the confirmed `WEB_HOST`/`WEB_PORT` healthcheck above. |
| PeaNUT API `401` with correct account; auth vanishes on recreate | Check `AUTH_FILE_PATH` and persisted `/config/auth.yaml`; v6 may save under `/app/config` without the fix. |
| Homepage widget `401` while `curl -u admin` gives `200` | Inspect secret files for trailing CR/LF and ensure `services.yaml` uses `HOMEPAGE_FILE_*` **not** stale `HOMEPAGE_VAR_*`. |
| UPS widget says `OL CHRG` | Expected raw compound status: online and charging. Homepage v2.4.0 does not normalize this compound string to the word “Online”. |
| Storage tile label does not match mount | Inspect `/api/4/fs` array index/order; indexed customapi labels are **not inherently stable**. |
| Glances `0.0%` CPU briefly after startup | Sample again once initialized and compare with Ubuntu. Do not conclude that system-wide metrics are broken from one startup sample. |

Revert Homepage only: remove the new `UPS` and `Storage` cards and the Glances info strip, then `sudo docker compose restart homepage`. Stop optional dashboards independently with `sudo docker compose stop` within their own stack directory. **Never stop NUT to debug PeaNUT**. Netdata cannot be brought back with `systemctl start` after its APT purge; reinstall would be a separate deliberate action.

## 11. Limitations and next phase

- Six filesystem **capacity** readings do **not** measure SMART health, bad sectors, disk wear or per-disk temperature. Implement those later under [Phase 2](../../roadmaps/server-health-dashboard/smart-extension.md); do not confuse the two.
- The six storage labels depend on observed Glances FS array order. Re-check after any topology/restart changes.
- PeaNUT HTTP is accessible from trusted LAN and both specified Docker networks, not solely one container; maintain authentication and no router forwarding. PeaNUT `6.0.0` is pinned, with two documented workarounds.
- Glances and PeaNUT are not substitutes for Netdata's alert history and detailed collectors. Their **user-accepted** scope is current-state metrics only.
- Keep all monitored NTFS/Samba data read-only from Glances and keep NUT host-native/loopback-only.
- When reconstructing on another host, verify image/tag availability, USB/NUT configuration, actual UID, host network addresses, fs ordering and reverse-proxy reachability rather than blindly copying `192.168.0.10` or `172.20/31`.

## References

- [Homepage 2.4.0 PeaNUT widget](https://github.com/gethomepage/homepage/blob/v2.4.0/docs/widgets/services/peanut.md)
- [Homepage 2.4.0 Custom API widget](https://github.com/gethomepage/homepage/blob/v2.4.0/docs/widgets/services/customapi.md)
- [Homepage environment-file secrets](https://github.com/gethomepage/homepage/blob/v2.4.0/docs/installation/docker.md)
- [PeaNUT v6 credential-path bug #423](https://github.com/Brandawg93/PeaNUT/issues/423)
- [PeaNUT source](https://github.com/Brandawg93/PeaNUT)
- [NUT upsd.conf / LISTEN semantics](https://networkupstools.org/docs/man/upsd.conf.html)
- [Docker and UFW networking](https://docs.docker.com/engine/network/packet-filtering-firewalls/)
