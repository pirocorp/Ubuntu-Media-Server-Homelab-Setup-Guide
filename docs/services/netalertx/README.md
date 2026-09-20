# NetAlertX Deployment And Operations Runbook

Status: Implemented  
Purpose: Install, configure, publish, validate, and operate NetAlertX for LAN device inventory and presence monitoring.  
Depends on: [Docker and Portainer](../../platform/docker-and-portainer.md), [Networking and reverse proxy](../../platform/networking-and-reverse-proxy.md)  
Related docs: [Services index](../README.md), [Service inventory](../../overview/service-inventory.md), [IPv6 leak validation and mitigation](../../operations/ipv6-leak-validation-and-mitigation-runbook.md)

## Current Deployment

NetAlertX is the homelab network inventory and presence-monitoring service. It discovers devices on the physical LAN, records IP/MAC/vendor information, and tracks whether devices are currently present.

| Item | Current value |
| --- | --- |
| NetAlertX version | `26.9.0` |
| Host | `piroman-server` / `192.168.0.10` |
| Physical LAN interface | `enp2s0` |
| LAN | `192.168.0.0/24` |
| Docker network mode | `host` |
| Web UI port | `20211/tcp` |
| GraphQL port | `20212/tcp` |
| Preferred URL | `https://netalertx.pirocorp.com` |
| Direct LAN URL | `http://192.168.0.10:20211` |
| ARP discovery | every 5 minutes |
| Port/service discovery | not enabled yet |

The first validated ARP scans successfully populated the device inventory.

## Architecture

```text
LAN devices (192.168.0.0/24)
          |
          | ARP discovery via enp2s0
          v
NetAlertX on piroman-server
network_mode: host
          |
          +--> :20211 Web UI
          +--> :20212 GraphQL
          +--> /srv/docker/netalertx/data

Client
  |
  | netalertx.pirocorp.com
  v
AdGuard Home DNS rewrite
  |
  | 192.168.0.10
  v
Nginx Proxy Manager :443
  |
  | HTTP to 192.168.0.10:20211
  v
NetAlertX
```

NetAlertX uses host networking because Layer-2 ARP discovery must operate directly on the physical LAN.

## Storage Layout

```text
/srv/docker/netalertx
├── compose.yml
└── data
    ├── config
    └── db
```

Persistent application state:

```text
/srv/docker/netalertx/data -> /data
```

The container uses UID/GID `20211`, so the persistent directory is owned by that numeric user/group.

## Installation

### 1. Create Directories

```bash
sudo mkdir -p /srv/docker/netalertx/data/config
sudo mkdir -p /srv/docker/netalertx/data/db
sudo chown -R 20211:20211 /srv/docker/netalertx/data
cd /srv/docker/netalertx
```

Validate:

```bash
ls -ldn /srv/docker/netalertx/data
```

Expected owner/group:

```text
20211 20211
```

### 2. Create `compose.yml`

```yaml
services:
  netalertx:
    image: ghcr.io/netalertx/netalertx:26.9.0
    container_name: netalertx

    network_mode: host

    read_only: true

    cap_drop:
      - ALL

    cap_add:
      - NET_ADMIN
      - NET_RAW
      - NET_BIND_SERVICE
      - CHOWN
      - SETUID
      - SETGID

    volumes:
      - ./data:/data:rw
      - /etc/localtime:/etc/localtime:ro

    tmpfs:
      - "/tmp:mode=1700,uid=0,gid=0,rw,noexec,nosuid,nodev,async,noatime,nodiratime"

    environment:
      PUID: 20211
      PGID: 20211
      LISTEN_ADDR: 0.0.0.0
      PORT: 20211
      GRAPHQL_PORT: 20212

    mem_limit: 2048m
    mem_reservation: 1024m
    cpu_shares: 512
    pids_limit: 512

    logging:
      options:
        max-size: "10m"
        max-file: "3"

    restart: unless-stopped
```

The image is pinned to `26.9.0` rather than `latest`.

### 3. Host ARP Flux Settings

Do not put these sysctls in the Compose service while using `network_mode: host`; Docker rejects them in the host network namespace.

Apply them on Ubuntu:

```bash
sudo sysctl -w net.ipv4.conf.all.arp_ignore=1
sudo sysctl -w net.ipv4.conf.all.arp_announce=2
```

Persist them:

```bash
sudo nano /etc/sysctl.d/99-netalertx-arp-flux.conf
```

```text
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
```

Apply and verify:

```bash
sudo sysctl --system
sysctl net.ipv4.conf.all.arp_ignore
sysctl net.ipv4.conf.all.arp_announce
```

Expected:

```text
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
```

### 4. Validate And Start

```bash
cd /srv/docker/netalertx
docker compose config
docker compose pull
docker compose up -d
```

Check status and logs:

```bash
docker ps --filter name=netalertx
docker logs --tail 100 netalertx
```

Healthy startup includes successful read/write checks for:

```text
/data/config/app.conf
/data/db/app.db
```

## Firewall

NetAlertX listens on the host because it uses host networking. UFW therefore controls access to `20211/tcp`.

### LAN Access

Allow the local LAN only:

```bash
sudo ufw allow from 192.168.0.0/24 to any port 20211 proto tcp comment 'NetAlertX LAN'
```

### Nginx Proxy Manager Access

Nginx Proxy Manager runs in Docker network:

```text
Network: nginx-proxy-manager_default
Subnet:  172.20.0.0/16
NPM IP:  172.20.0.2
Gateway: 172.20.0.1
```

Because NPM reaches the host from this Docker subnet, explicitly allow that subnet to the NetAlertX web port:

```bash
sudo ufw allow from 172.20.0.0/16 to 192.168.0.10 port 20211 proto tcp comment 'NetAlertX from NPM'
```

Validate:

```bash
sudo ufw status numbered
```

Do not replace these restricted rules with a broad rule such as:

```bash
sudo ufw allow 20211/tcp
```

If the NPM Docker network is ever recreated with a different subnet, re-check it before updating UFW:

```bash
docker inspect -f '{{range $k,$v := .NetworkSettings.Networks}}{{$k}} IP={{$v.IPAddress}} Gateway={{$v.Gateway}}{{"\n"}}{{end}}' nginx-proxy-manager

docker network inspect nginx-proxy-manager_default \
  --format '{{range .IPAM.Config}}{{.Subnet}}{{end}}'
```

## Initial Web Validation

Confirm that NetAlertX responds locally:

```bash
curl -sS -o /dev/null -w 'HTTP %{http_code}\n' http://127.0.0.1:20211/
```

Expected response is a redirect such as:

```text
HTTP 302
```

Verify the socket:

```bash
ss -lntp | grep 20211
```

Expected binding:

```text
0.0.0.0:20211
```

## LAN Discovery Configuration

In NetAlertX open:

```text
Settings -> Core -> Networks to scan (SCAN_SUBNETS)
```

Remove automatically detected Docker bridge and Tailscale networks. Keep only:

```text
192.168.0.0/24 --interface=enp2s0
```

Do not scan Docker `172.x.x.x` networks unless container-level inventory is intentionally required. Do not scan `tailscale0` unless tailnet inventory is intentionally required.

## ARP Scan Schedule

Current configuration:

```text
ARPSCAN_RUN='schedule'
ARPSCAN_RUN_SCHD='*/5 * * * *'
ARPSCAN_RUN_TIMEOUT=300
```

`*/5 * * * *` means scans run on clock boundaries divisible by five minutes, for example:

```text
18:45
18:50
18:55
19:00
```

It does not mean five minutes after startup.

### Manual Discovery Validation

```bash
docker exec netalertx ip -br addr show enp2s0
```

Expected:

```text
enp2s0 UP 192.168.0.10/24
```

Run ARP scan directly:

```bash
docker exec netalertx arp-scan --interface=enp2s0 192.168.0.0/24
```

Inspect saved settings:

```bash
docker exec netalertx sh -c \
"grep -E '^(SCAN_SUBNETS|ARPSCAN_RUN|ARPSCAN_RUN_SCHD|ARPSCAN_RUN_TIMEOUT)=' /data/config/app.conf"
```

Expected:

```text
SCAN_SUBNETS=['192.168.0.0/24 --interface=enp2s0']
ARPSCAN_RUN='schedule'
ARPSCAN_RUN_TIMEOUT=300
ARPSCAN_RUN_SCHD='*/5 * * * *'
```

## Current Enabled Scanners

| Plugin | Mode | Schedule / Trigger | Purpose |
| --- | --- | --- | --- |
| ARPSCAN | `schedule` | `*/5 * * * *` | LAN device discovery |
| Internet-Check | `schedule` | `*/5 * * * *` | Internet presence/gateway check |
| AVAHISCAN | `before_name_updates` | `*/30 * * * *` | Device naming |
| NBTSCAN | `before_name_updates` | `*/30 * * * *` | NetBIOS naming |
| NSLOOKUP | `before_name_updates` | `*/30 * * * *` | DNS naming |
| DIGSCAN | `before_name_updates` | `*/30 * * * *` | DNS name resolution |

Naming plugins may log timeouts without preventing ARP discovery from working.

## DNS Publishing With AdGuard Home

The homelab currently uses explicit DNS rewrites rather than a wildcard rewrite.

In AdGuard Home add:

```text
netalertx.pirocorp.com -> 192.168.0.10
```

Validate from a LAN client:

```bash
nslookup netalertx.pirocorp.com
```

Expected address:

```text
192.168.0.10
```

## Nginx Proxy Manager

Create a dedicated Proxy Host.

### Details

```text
Domain Names:        netalertx.pirocorp.com
Scheme:              http
Forward Hostname/IP: 192.168.0.10
Forward Port:        20211
Access List:         Publicly Accessible
Cache Assets:        OFF
Block Common Exploits: ON
Websockets Support:    ON
```

### SSL

Select the existing wildcard certificate covering:

```text
pirocorp.com
*.pirocorp.com
```

Enable:

```text
Force SSL:      ON
HTTP/2 Support: ON
HSTS:           OFF
```

The final preferred URL is:

```text
https://netalertx.pirocorp.com
```

### Validate NPM-To-NetAlertX Connectivity

Test from inside the NPM container:

```bash
docker exec nginx-proxy-manager \
  curl -sS -o /dev/null -w 'HTTP %{http_code}\n' \
  --connect-timeout 5 http://192.168.0.10:20211/
```

Expected:

```text
HTTP 302
```

A `504 Gateway Time-out` from NPM while direct LAN access works usually means UFW is blocking the NPM Docker subnet. Verify the `172.20.0.0/16 -> 192.168.0.10:20211/tcp` rule.

## Port And Service Discovery

Nmap-based port/service discovery is not yet enabled in the current deployment.

Current implemented scope:

- device presence;
- IP address;
- MAC address;
- vendor identification;
- basic naming where available.

Nmap/NMAPDEV should be enabled and tuned separately after the baseline inventory is stable.

## Docker Operations

```bash
cd /srv/docker/netalertx
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

Logs:

```bash
docker logs -f netalertx
```

Resources:

```bash
docker stats netalertx
```

## Updating

The image is pinned. Choose and verify the desired version before changing the image tag in:

```text
/srv/docker/netalertx/compose.yml
```

Then:

```bash
docker compose config
docker compose pull
docker compose up -d
```

Verify both ARP discovery and `https://netalertx.pirocorp.com` after every upgrade.

## Backup

Persistent state is under:

```text
/srv/docker/netalertx/data
```

Stopped-container backup:

```bash
cd /srv/docker/netalertx
docker compose down
sudo tar -czf netalertx-data-backup.tar.gz data/
docker compose up -d
```

## Troubleshooting

### Host-Network Sysctl Error

Symptom:

```text
sysctl "net.ipv4.conf.all.arp_announce" not allowed in host network namespace
```

Fix: remove the `sysctls:` block from Compose and apply `arp_ignore=1` / `arp_announce=2` on the Ubuntu host.

### Direct LAN UI Times Out

Test locally:

```bash
curl -sS -o /dev/null -w 'HTTP %{http_code}\n' http://127.0.0.1:20211/
```

If local access works, inspect UFW and confirm the `192.168.0.0/24` rule.

### NPM Returns 504

Confirm the NPM Docker network and subnet, then test connectivity from the NPM container. The current deployment requires:

```text
172.20.0.0/16 -> 192.168.0.10:20211/tcp
```

### Dashboard Shows No Devices

Check:

```bash
docker exec netalertx sh -c "grep '^SCAN_SUBNETS=' /data/config/app.conf"
docker exec netalertx arp-scan --interface=enp2s0 192.168.0.0/24
```

If the direct ARP command finds devices, Layer-2 networking and container capabilities are working; inspect the NetAlertX scheduler logs next.

### Scheduler Shows `ARPSCAN: NO`

This is normal between cron boundaries. With `*/5 * * * *`, the scanner runs at the next five-minute boundary.

### Naming Plugin Timeouts

AVAHISCAN, NBTSCAN, NSLOOKUP, or DIGSCAN may time out during initial setup. Treat them separately from core ARP discovery.

## Security Notes

- NetAlertX uses host networking because ARP discovery requires Layer-2 access.
- The container drops all capabilities and adds back only the required set.
- The filesystem is read-only except for `/data` and tmpfs.
- Direct UI access is restricted to the LAN by UFW.
- NPM access is restricted to the NPM Docker subnet by UFW.
- HTTPS access uses the existing Let's Encrypt wildcard certificate through Nginx Proxy Manager.
- Native public IPv6 on `enp2s0` is disabled separately; see the [IPv6 leak validation and mitigation runbook](../../operations/ipv6-leak-validation-and-mitigation-runbook.md).
- Do not expose `20211/tcp` directly to the public Internet.

## Operational Health Checklist

```bash
cd /srv/docker/netalertx
docker compose ps
docker logs --tail 100 netalertx
sysctl net.ipv4.conf.all.arp_ignore
sysctl net.ipv4.conf.all.arp_announce
sudo ufw status numbered
docker exec netalertx ip -br addr show enp2s0
nslookup netalertx.pirocorp.com
```

In the UI verify:

- `SCAN_SUBNETS` contains only `192.168.0.0/24 --interface=enp2s0`;
- ARPSCAN is scheduled every 5 minutes;
- devices appear under **Devices -> All devices**;
- the next scan countdown advances normally.

Finally verify:

```text
https://netalertx.pirocorp.com
```
