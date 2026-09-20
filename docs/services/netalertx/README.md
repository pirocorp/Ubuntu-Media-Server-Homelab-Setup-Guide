# NetAlertX Deployment And Operations Runbook

Status: Implemented
Purpose: Installation, configuration, validation, and operation of NetAlertX for LAN device inventory and presence monitoring.
Depends on: [Docker and Portainer](../../platform/docker-and-portainer.md), [Networking and reverse proxy](../../platform/networking-and-reverse-proxy.md)
Related docs: [Services index](../README.md), [Service inventory](../../overview/service-inventory.md), [IPv6 leak validation and mitigation](../../operations/ipv6-leak-validation-and-mitigation-runbook.md)

## Overview

NetAlertX is deployed as the homelab network inventory and presence-monitoring service. It discovers devices on the physical LAN, tracks whether they are currently present, records IP/MAC/vendor information, and can later be extended with Nmap-based port and service discovery.

Current deployment state:

- NetAlertX version: `26.9.0`
- Host: `192.168.0.10`
- Physical LAN interface: `enp2s0`
- LAN: `192.168.0.0/24`
- Docker networking mode: `host`
- Web UI port: `20211/tcp`
- GraphQL port: `20212/tcp`
- Current access: direct LAN access only
- ARP discovery: enabled every 5 minutes
- Port/service discovery: not enabled yet

The first validated ARP scan successfully populated the inventory with the active devices visible on the LAN.

## Architecture

```text
LAN devices (192.168.0.0/24)
          |
          | ARP discovery
          v
Ubuntu Server (192.168.0.10)
          |
          | enp2s0
          v
NetAlertX container
network_mode: host
          |
          +--> Web UI :20211
          +--> GraphQL :20212
          +--> /srv/docker/netalertx/data
```

Because Layer-2 ARP discovery must operate directly on the physical LAN, NetAlertX runs with `network_mode: host` rather than a normal Docker bridge network.

## Storage Layout

```text
/srv/docker/netalertx
├── compose.yml
└── data
    ├── config
    └── db
```

Persistent application state is mounted as:

```text
/srv/docker/netalertx/data -> /data
```

The NetAlertX container user uses UID/GID `20211`, so the persistent directory is owned accordingly.

## Installation

### 1. Create Directories

```bash
sudo mkdir -p /srv/docker/netalertx/data/config
sudo mkdir -p /srv/docker/netalertx/data/db
sudo chown -R 20211:20211 /srv/docker/netalertx/data
cd /srv/docker/netalertx
```

Validate ownership:

```bash
ls -ldn /srv/docker/netalertx/data
```

Expected owner/group:

```text
20211 20211
```

### 2. Docker Compose

Current `compose.yml`:

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

The image is pinned to version `26.9.0` rather than using `latest`.

## Host ARP Flux Settings

Do **not** put these sysctls inside the Compose service when using `network_mode: host` on this host. Docker rejects them with an error similar to:

```text
sysctl "net.ipv4.conf.all.arp_announce" not allowed in host network namespace
```

Apply them on the Ubuntu host instead:

```bash
sudo sysctl -w net.ipv4.conf.all.arp_ignore=1
sudo sysctl -w net.ipv4.conf.all.arp_announce=2
```

Make them persistent:

```bash
sudo nano /etc/sysctl.d/99-netalertx-arp-flux.conf
```

Contents:

```text
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
```

Apply all sysctl configuration:

```bash
sudo sysctl --system
```

Validate:

```bash
sysctl net.ipv4.conf.all.arp_ignore
sysctl net.ipv4.conf.all.arp_announce
```

Expected:

```text
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
```

## Start The Stack

Validate Compose first:

```bash
cd /srv/docker/netalertx
docker compose config
```

Pull the pinned image:

```bash
docker compose pull
```

Start:

```bash
docker compose up -d
```

Check status:

```bash
docker ps --filter name=netalertx
```

View logs:

```bash
docker logs --tail 100 netalertx
```

A healthy startup should show successful read/write checks for:

```text
/data/config/app.conf
/data/db/app.db
```

## Firewall

The host uses UFW. NetAlertX direct access is restricted to the local LAN.

Allow only `192.168.0.0/24` to reach the web UI:

```bash
sudo ufw allow from 192.168.0.0/24 to any port 20211 proto tcp comment 'NetAlertX LAN'
```

Validate:

```bash
sudo ufw status numbered
```

Do not use a broad rule such as:

```bash
sudo ufw allow 20211/tcp
```

unless wider access is explicitly intended.

## Web UI

Current direct LAN URL:

```text
http://192.168.0.10:20211
```

The reverse-proxy/DNS publication step has not yet been implemented for NetAlertX.

Local application validation from the host:

```bash
curl -sS -o /dev/null -w 'HTTP %{http_code}\n' http://127.0.0.1:20211/
```

A redirect such as `HTTP 302` confirms the web service is responding.

Verify the listening socket:

```bash
ss -lntp | grep 20211
```

Expected binding:

```text
0.0.0.0:20211
```

## LAN Discovery Configuration

Open:

```text
Settings -> Core -> Networks to scan (SCAN_SUBNETS)
```

Remove automatically detected Docker bridge or Tailscale networks from the scan list. The production scan target should be only:

```text
192.168.0.0/24 --interface=enp2s0
```

Do not scan the Docker `172.x.x.x` bridge networks unless container-level inventory is intentionally required.

Do not scan the Tailscale interface unless tailnet inventory is intentionally required.

## ARP Scan Schedule

The current ARPSCAN configuration is:

```text
ARPSCAN_RUN='schedule'
ARPSCAN_RUN_SCHD='*/5 * * * *'
ARPSCAN_RUN_TIMEOUT=300
```

This means the scan runs at clock times divisible by five minutes:

```text
18:45
18:50
18:55
19:00
...
```

It does **not** mean "five minutes after the service starts".

The UI may therefore show a countdown such as:

```text
Next scan in about 2m 37s
```

when the next cron boundary is approaching.

## Validate Discovery Manually

Confirm that the container sees the physical interface:

```bash
docker exec netalertx ip -br addr show enp2s0
```

Expected:

```text
enp2s0 UP 192.168.0.10/24
```

Run a direct ARP scan from inside the container:

```bash
docker exec netalertx arp-scan --interface=enp2s0 192.168.0.0/24
```

A successful scan should return active LAN devices with IP, MAC, and vendor information.

Inspect the saved configuration:

```bash
docker exec netalertx sh -c \
"grep -E '^(SCAN_SUBNETS|ARPSCAN_RUN|ARPSCAN_RUN_SCHD|ARPSCAN_RUN_TIMEOUT)=' /data/config/app.conf"
```

Expected values:

```text
SCAN_SUBNETS=['192.168.0.0/24 --interface=enp2s0']
ARPSCAN_RUN='schedule'
ARPSCAN_RUN_TIMEOUT=300
ARPSCAN_RUN_SCHD='*/5 * * * *'
```

## Current Enabled Scanners

Current baseline:

| Plugin | Mode | Schedule / Trigger | Purpose |
| --- | --- | --- | --- |
| ARPSCAN | `schedule` | `*/5 * * * *` | LAN device discovery |
| Internet-Check | `schedule` | `*/5 * * * *` | Internet presence/gateway check |
| AVAHISCAN | `before_name_updates` | `*/30 * * * *` | Device naming |
| NBTSCAN | `before_name_updates` | `*/30 * * * *` | NetBIOS naming |
| NSLOOKUP | `before_name_updates` | `*/30 * * * *` | DNS naming |
| DIGSCAN | `before_name_updates` | `*/30 * * * *` | DNS name resolution |

During initial deployment, the naming plugins may log timeouts. These do not prevent ARP discovery from functioning.

## Port And Service Discovery

Nmap-based port/service discovery is **not yet enabled** in the current deployment.

The current implemented scope is:

- device presence;
- IP address;
- MAC address;
- vendor identification;
- basic naming where available.

Nmap/NMAPDEV should be enabled and tuned separately after the baseline inventory is stable, because service scanning is more intrusive and more resource-intensive than ARP presence discovery.

## Docker Operations

Navigate to stack:

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

Resource usage:

```bash
docker stats netalertx
```

## Updating

The image is pinned. To update, first choose and verify the desired NetAlertX release/tag, then edit:

```text
/srv/docker/netalertx/compose.yml
```

After changing the tag:

```bash
docker compose config
docker compose pull
docker compose up -d
```

Verify the UI and ARP discovery after every upgrade.

## Backup

The persistent state is under:

```text
/srv/docker/netalertx/data
```

A basic stopped-container backup can be taken with:

```bash
cd /srv/docker/netalertx
docker compose down
sudo tar -czf netalertx-data-backup.tar.gz data/
docker compose up -d
```

For routine backups, include `/srv/docker/netalertx/data` in the homelab backup policy.

## Troubleshooting

### Container Fails With Host-Network Sysctl Error

Symptom:

```text
sysctl "net.ipv4.conf.all.arp_announce" not allowed in host network namespace
```

Fix:

1. Remove the `sysctls:` block from `compose.yml`.
2. Apply `arp_ignore=1` and `arp_announce=2` on the Ubuntu host.
3. Persist them in `/etc/sysctl.d/99-netalertx-arp-flux.conf`.
4. Start the stack again.

### UI Works Locally But Times Out From Another LAN Client

Confirm local response:

```bash
curl -sS -o /dev/null -w 'HTTP %{http_code}\n' http://127.0.0.1:20211/
```

If local access works but remote LAN access times out, inspect UFW:

```bash
sudo ufw status numbered
```

Ensure the LAN-only `20211/tcp` rule exists.

### Dashboard Shows No Devices

Check the configured scan subnet:

```bash
docker exec netalertx sh -c \
"grep '^SCAN_SUBNETS=' /data/config/app.conf"
```

Then test ARP directly:

```bash
docker exec netalertx arp-scan --interface=enp2s0 192.168.0.0/24
```

If the direct command finds devices, Layer-2 networking and container capabilities are working. Then inspect scheduler/plugin logs:

```bash
docker logs -f netalertx
```

### Scheduler Shows `ARPSCAN: NO`

This can be normal between cron boundaries. With:

```text
*/5 * * * *
```

ARPSCAN should switch to `YES` at the next five-minute boundary.

### Naming Plugin Timeouts

AVAHISCAN, NBTSCAN, NSLOOKUP, or DIGSCAN may time out during initial setup. Treat these separately from ARP discovery. A successful manual ARP scan confirms the core LAN discovery path independently.

## Security Notes

- NetAlertX uses host networking because ARP discovery requires Layer-2 access.
- The container drops all capabilities and adds back only the required set.
- The filesystem is read-only except for `/data` and the temporary tmpfs.
- Direct UI access is restricted by UFW to the LAN.
- Native public IPv6 on `enp2s0` is disabled separately; see the [IPv6 leak validation and mitigation runbook](../../operations/ipv6-leak-validation-and-mitigation-runbook.md).
- Do not expose port `20211` directly to the public Internet.

## Operational Health Checklist

```bash
cd /srv/docker/netalertx
docker compose ps
docker logs --tail 100 netalertx
sysctl net.ipv4.conf.all.arp_ignore
sysctl net.ipv4.conf.all.arp_announce
sudo ufw status numbered
docker exec netalertx ip -br addr show enp2s0
```

In the UI verify:

- `SCAN_SUBNETS` contains only `192.168.0.0/24 --interface=enp2s0`;
- ARPSCAN is scheduled every 5 minutes;
- devices appear under **Devices -> All devices**;
- the next scan countdown advances normally.
