# Infrastructure HowTo

Status: Implemented
Purpose: Canonical step-by-step guide for building the current homelab platform from a fresh Ubuntu Server installation up to the point where the first workload can be deployed.
Depends on: Fresh Ubuntu Server install media, a stable LAN IP, and access to the required storage devices and DNS provider account
Related docs: [Platform index](./README.md), [Base server setup](./base-server-setup.md), [Docker and Portainer](./docker-and-portainer.md), [Storage and Samba](./storage-and-samba.md), [Networking and reverse proxy](./networking-and-reverse-proxy.md), [Let's Encrypt public-domain guide](./lets-encrypt-public-domain.md), [Legacy build history](../archive/legacy-root-readme.md)

## What This Guide Builds

This is the canonical build path for the current infrastructure baseline.

Follow this guide when you want to recreate the homelab platform itself:

- Ubuntu Server host
- SSH administration
- UFW baseline firewall
- Docker Engine and Docker Compose
- `/srv/docker` stack layout
- mounted storage under `/mnt/*`
- Samba shares
- AdGuard Home local DNS
- Nginx Proxy Manager
- Let's Encrypt and `*.pirocorp.com`
- Portainer

Stop here after the platform is ready. Deploy Plex, Immich, Nextcloud, qBittorrent, and the other workloads only after this guide is complete.

This guide intentionally does not cover:

- workload deployment steps
- workload-specific reverse proxy details beyond the base pattern
- VPN / remote access
- the older Cockpit-based management path
- the older `.home` or self-signed-certificate naming scheme as the active target state

## Target End State

| Item | Target |
| --- | --- |
| Hostname | `piroman-server` |
| Main admin user | `piroman` |
| Stable LAN IP | `192.168.0.10` |
| OS | `Ubuntu 26.04 LTS` |
| Container root | `/srv/docker` |
| Shared storage root | `/mnt/*` |
| DNS model | AdGuard Home local DNS rewrites |
| Published domain pattern | `*.pirocorp.com` |
| Reverse proxy | Nginx Proxy Manager |
| Container management | Portainer |

If you rebuild on different hardware or a different LAN, keep the same structure and substitute the actual stable IP where required.

## Phase 1 - Install Ubuntu And Set The Host Baseline

### Goal

Install a clean headless Ubuntu Server host with the current naming and storage assumptions.

### Commands / Config

Prepare the installer with these target choices:

| Setting | Recommended value |
| --- | --- |
| Edition | Ubuntu Server 26.04 LTS |
| Install type | Minimized |
| Hostname | `piroman-server` |
| Main user | `piroman` |
| System disk | NVMe SSD |
| Root filesystem | `ext4` |
| LVM | Off |
| Disk encryption | Off unless you explicitly need it |
| OpenSSH during install | Enabled |

Reserve or otherwise keep the server on a stable LAN IP:

```text
192.168.0.10
```

The simplest current-aligned approach is to keep DHCP on the ISP router and use a DHCP reservation for the server rather than reintroducing a different static-IP scheme.

### Important Decisions / Caveats

- This guide assumes the system disk hosts the OS and container state, while large media and shared data live on mounted HDDs.
- Do not bring back Cockpit as part of the main bootstrap path.
- Do not use `.home`, `.lan`, or self-signed certificates as the intended final naming model.

### Validation Checkpoint

Run on the new server:

```bash
hostnamectl
ip addr show
df -h /
```

Confirm:

- hostname is `piroman-server`
- the server has the expected LAN IP
- the system disk is mounted and healthy

### State Before Next Phase

You have a bootable Ubuntu Server host with SSH available and a stable LAN identity.

## Phase 2 - First Login, Updates, And Admin Hardening

### Goal

Get the host fully updated, install the core admin utilities, and switch to key-based SSH access.

### Commands / Config

Connect from Windows PowerShell:

```powershell
ssh piroman@192.168.0.10
```

On the server:

```bash
sudo apt update
sudo apt full-upgrade -y
sudo apt autoremove -y

sudo apt install -y \
  tmux \
  curl \
  wget \
  git \
  tree \
  ncdu \
  ca-certificates \
  gnupg \
  ufw \
  ntfs-3g
```

Create an SSH key on Windows if you do not already have one dedicated to this server:

```powershell
ssh-keygen -t ed25519 -C "piroman-windows" -f $env:USERPROFILE\.ssh\piroman-server
Get-Content $env:USERPROFILE\.ssh\piroman-server.pub
```

Then on the server:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Paste the Windows public key into `~/.ssh/authorized_keys`, save the file, then test a new login from Windows:

```powershell
ssh -i $env:USERPROFILE\.ssh\piroman-server piroman@192.168.0.10
```

### Important Decisions / Caveats

- Keep password login available until you have confirmed key-based SSH works from a second session.
- This guide standardizes on a dedicated key name of `piroman-server` on the Windows side for clarity.

### Validation Checkpoint

Run:

```bash
hostnamectl
uname -a
free -h
```

From Windows, confirm the SSH key login succeeds without asking for the Linux account password.

### State Before Next Phase

The host is fully updated, the core admin tools are installed, and you can manage the server reliably over SSH.

## Phase 3 - Establish The Firewall Baseline

### Goal

Apply the base UFW policy before Docker services and shares are added.

### Commands / Config

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status verbose
```

### Important Decisions / Caveats

- At this stage only SSH needs to be explicitly opened.
- Samba, DNS, and reverse-proxy ports are opened later when those components are actually deployed.
- Docker publishes ports through its own networking rules, so always validate real exposure after each platform service is added instead of assuming UFW alone tells the whole story.

### Validation Checkpoint

Confirm:

- `OpenSSH` is allowed
- incoming traffic is denied by default
- the current SSH session stays connected after UFW is enabled

### State Before Next Phase

The host has a clean firewall baseline and is still reachable over SSH.

## Phase 4 - Install Docker Engine And Docker Compose

### Goal

Install the container runtime used by the entire homelab.

### Commands / Config

```bash
sudo apt install -y docker.io docker-compose-v2
sudo systemctl enable docker
sudo systemctl start docker
sudo docker run hello-world
sudo usermod -aG docker piroman
```

Log out and reconnect so the new group membership takes effect, then run:

```bash
docker ps
docker compose version
systemctl status docker
```

### Important Decisions / Caveats

- This guide uses the Ubuntu-packaged Docker Engine and Compose plugin because that matches the current homelab build history.
- Add `piroman` to the `docker` group only after Docker itself is installed and working.

### Validation Checkpoint

Confirm:

- `hello-world` runs successfully
- `docker ps` works without `sudo`
- `docker compose version` returns a version string
- the Docker service is enabled and active

### State Before Next Phase

The host can run and manage Docker containers as the main admin user.

## Phase 5 - Establish The Docker Stack Layout

### Goal

Create the canonical directory structure that the platform and later workloads will use.

### Commands / Config

```bash
sudo mkdir -p /srv/docker
sudo chown -R piroman:piroman /srv/docker

mkdir -p /srv/docker/adguard-home
mkdir -p /srv/docker/nginx-proxy-manager
mkdir -p /srv/docker/portainer
```

Optional quick check:

```bash
tree /srv/docker
```

### Important Decisions / Caveats

- The current convention is one stack directory per service under `/srv/docker/<app>`.
- Keep compose files, local persistent data, and environment files together in each stack directory whenever possible.

### Validation Checkpoint

Confirm that `/srv/docker` exists and is writable by `piroman`.

### State Before Next Phase

The server now has the standard stack root expected by the rest of the repo.

## Phase 6 - Mount Storage And Configure Samba

### Goal

Make the shared HDD storage stable for both Docker bind mounts and Windows file access.

### Commands / Config

Create the mount points:

```bash
sudo mkdir -p /mnt/{iac,lp,data,ia,comp}
```

Inspect drives and collect the real UUIDs from this machine:

```bash
lsblk -f
```

Edit `/etc/fstab` and add one entry per NTFS storage domain using the current mount layout:

```fstab
UUID=<IAC_UUID>   /mnt/iac   ntfs-3g  defaults,uid=1000,gid=1000,umask=0000,nofail  0  0
UUID=<LP_UUID>    /mnt/lp    ntfs-3g  defaults,uid=1000,gid=1000,umask=0000,nofail  0  0
UUID=<DATA_UUID>  /mnt/data  ntfs-3g  defaults,uid=1000,gid=1000,umask=0000,nofail  0  0
UUID=<IA_UUID>    /mnt/ia    ntfs-3g  defaults,uid=1000,gid=1000,umask=0000,nofail  0  0
UUID=<COMP_UUID>  /mnt/comp  ntfs-3g  defaults,uid=1000,gid=1000,umask=0000,nofail  0  0
```

Apply and verify:

```bash
sudo mount -a
df -h
```

If an NTFS volume fails to mount because of a dirty flag left by Windows, repair that specific partition and retry:

```bash
sudo ntfsfix /dev/<affected-partition>
sudo mount -a
```

Install and configure Samba:

```bash
sudo apt install -y samba
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.backup
sudo nano /etc/samba/smb.conf
```

Append the current share model:

```ini
[iac]
   path = /mnt/iac
   browseable = yes
   writable = yes
   guest ok = no
   valid users = piroman

[lp]
   path = /mnt/lp
   browseable = yes
   writable = yes
   guest ok = no
   valid users = piroman

[data]
   path = /mnt/data
   browseable = yes
   writable = yes
   guest ok = no
   valid users = piroman

[ia]
   path = /mnt/ia
   browseable = yes
   writable = yes
   guest ok = no
   valid users = piroman

[comp]
   path = /mnt/comp
   browseable = yes
   writable = yes
   guest ok = no
   valid users = piroman
```

Validate and start Samba:

```bash
testparm
sudo smbpasswd -a piroman
sudo systemctl enable smbd
sudo systemctl restart smbd
sudo systemctl status smbd
sudo ufw allow Samba
```

### Important Decisions / Caveats

- The `umask=0000` setting is intentional in this environment so Samba shares stay read/write from Windows.
- `nofail` prevents the whole system from blocking boot if one data drive is temporarily unavailable.
- Keep the mount points stable because Docker bind mounts and Samba shares both depend on them.

### Validation Checkpoint

Confirm:

- `df -h` shows the expected mounts under `/mnt/*`
- `testparm` succeeds
- `smbd` is active
- from Windows Explorer, `\\192.168.0.10` or `\\piroman-server` opens the shares

### State Before Next Phase

Shared storage is mounted persistently and the Windows-accessible Samba layer is ready.

## Phase 7 - Deploy AdGuard Home For Local DNS

### Goal

Replace ad-hoc client DNS with a dedicated local DNS service that will later resolve `*.pirocorp.com` to the server.

### Commands / Config

Open the required LAN ports:

```bash
sudo ufw allow 53/tcp
sudo ufw allow 53/udp
sudo ufw allow 3000/tcp
```

Free port 53 from the Ubuntu DNS stub listener:

```bash
sudo nano /etc/systemd/resolved.conf
```

Under `[Resolve]`, set:

```ini
DNSStubListener=no
```

Then run:

```bash
sudo systemctl restart systemd-resolved
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
resolvectl query google.com
sudo ss -tulpn | grep :53
```

Create the stack layout:

```bash
mkdir -p /srv/docker/adguard-home/{work,conf}
cd /srv/docker/adguard-home
nano compose.yml
```

Use the current-aligned compose file:

```yaml
services:
  adguard-home:
    image: adguard/adguardhome:v0.107.77
    container_name: adguard-home
    restart: unless-stopped
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "3000:80/tcp"
    volumes:
      - ./work:/opt/adguardhome/work
      - ./conf:/opt/adguardhome/conf
```

Validate and deploy:

```bash
docker compose config
docker compose up -d
docker ps
```

Finish the initial AdGuard setup in a browser:

```text
http://192.168.0.10:3000
```

During the wizard:

- bind the admin interface to all interfaces
- keep AdGuard DNS on port `53`
- create the admin account
- do not enable AdGuard DHCP; the ISP router remains the DHCP server

After the wizard, configure trusted upstream DNS providers. A good starting point is:

- Quad9 over DNS-over-HTTPS
- Cloudflare over DNS-over-HTTPS

Then point at least one test client to AdGuard Home as its DNS server. The preferred production path is to make the ISP router hand out `192.168.0.10` as LAN DNS if the router supports custom DNS settings.

### Important Decisions / Caveats

- The current setup does not expose DHCP ports `67/68` because DHCP still lives on the ISP router.
- This guide goes straight to the final host port mapping of `3000 -> 80` for AdGuard's web UI.
- If `53` is still busy after disabling the stub listener, resolve that before moving on.

### Validation Checkpoint

Confirm:

- `resolvectl query google.com` still works on the server
- `docker ps` shows published ports for `53` and `3000`
- `http://192.168.0.10:3000` opens
- a LAN client using AdGuard can resolve a normal internet name with `nslookup`

### State Before Next Phase

AdGuard Home is running as the LAN DNS service and is ready to support homelab name resolution.

## Phase 8 - Deploy Nginx Proxy Manager

### Goal

Add the central reverse-proxy layer that will terminate HTTPS for homelab services.

### Commands / Config

Open the intended reverse-proxy and admin ports:

```bash
sudo ufw allow 80/tcp
sudo ufw allow 81/tcp
sudo ufw allow 443/tcp
```

Create the stack layout:

```bash
mkdir -p /srv/docker/nginx-proxy-manager/{data,letsencrypt}
cd /srv/docker/nginx-proxy-manager
nano compose.yml
```

Use the current-aligned compose file:

```yaml
services:
  nginx-proxy-manager:
    image: jc21/nginx-proxy-manager:2.15.1
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      - "80:80"
      - "81:81"
      - "443:443"
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
```

Validate and deploy:

```bash
docker compose config
docker compose up -d
docker ps
sudo ss -tulpn | grep -E ':80|:81|:443'
```

Open the admin UI:

```text
http://192.168.0.10:81
```

Log in with the default bootstrap credentials:

```text
Email: admin@example.com
Password: changeme
```

Immediately replace them with real admin credentials.

### Important Decisions / Caveats

- NPM is the central HTTPS entry point for this homelab.
- The admin UI intentionally stays on direct port `81`. The current environment does not proxy NPM back through itself.

### Validation Checkpoint

Confirm:

- the NPM container is running
- ports `80`, `81`, and `443` are bound
- the admin UI opens and accepts the new credentials

### State Before Next Phase

The reverse proxy exists and is ready for certificates, DNS-backed names, and proxy hosts.

## Phase 9 - Configure Let's Encrypt And The `pirocorp.com` Naming Model

### Goal

Move from raw IPs to the current `*.pirocorp.com` access pattern used across the homelab.

### Commands / Config

Use the full [Let's Encrypt public-domain guide](./lets-encrypt-public-domain.md) as the detailed reference for this phase. The short version of the current target flow is:

1. Create or reuse the DNS-provider API token used by Nginx Proxy Manager.
2. In NPM, create a wildcard certificate for:

```text
pirocorp.com
*.pirocorp.com
```

3. In AdGuard Home, add the local DNS rewrite:

```text
*.pirocorp.com -> 192.168.0.10
```

4. Keep `npm.pirocorp.com:81` as a direct host-and-port access path for NPM admin.
5. Create the AdGuard proxy host in NPM:

```text
Domain: adguard.pirocorp.com
Scheme: http
Forward Host: 192.168.0.10
Forward Port: 3000
SSL Certificate: pirocorp.com, *.pirocorp.com
Enable: Force SSL, HTTP/2
```

From a LAN client that uses AdGuard for DNS, test:

```bash
nslookup adguard.pirocorp.com
nslookup npm.pirocorp.com
```

### Important Decisions / Caveats

- The current homelab uses `*.pirocorp.com` as a local split-DNS naming scheme backed by AdGuard rewrites to the private server IP.
- This is not the same thing as making every service a public internet endpoint.
- Keep the wildcard rewrite and the wildcard certificate aligned. They are the foundation for later workload publishing too.

### Validation Checkpoint

Confirm:

- `nslookup adguard.pirocorp.com` returns `192.168.0.10`
- `https://adguard.pirocorp.com` opens without certificate warnings
- `http://npm.pirocorp.com:81` opens on the LAN

### State Before Next Phase

The platform has the current naming and TLS model in place.

## Phase 10 - Deploy Portainer And Publish It Through The Platform

### Goal

Add the current container-management UI and publish it under the same naming model as the rest of the platform.

### Commands / Config

Open the direct fallback UI port:

```bash
sudo ufw allow 9443/tcp
```

Create the stack layout:

```bash
mkdir -p /srv/docker/portainer/data
cd /srv/docker/portainer
nano compose.yml
```

Use the current-aligned compose file:

```yaml
services:
  portainer:
    image: portainer/portainer-ce:2.39.3
    container_name: portainer
    restart: unless-stopped
    ports:
      - "8000:8000"
      - "9443:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /srv/docker/portainer/data:/data
```

Validate and deploy:

```bash
docker compose config
docker compose up -d
docker ps
```

Finish the initial setup at the direct fallback address:

```text
https://192.168.0.10:9443
```

If the initial admin-creation page times out before you complete it, restart the stack and retry:

```bash
docker compose restart
```

After the local Docker environment is connected inside Portainer, create the NPM proxy host:

```text
Domain: portainer.pirocorp.com
Scheme: https
Forward Host: 192.168.0.10
Forward Port: 9443
SSL Certificate: pirocorp.com, *.pirocorp.com
Enable: Force SSL, HTTP/2
```

### Important Decisions / Caveats

- Portainer's direct `9443` address is the bootstrap and fallback path.
- The intended day-to-day access path after this phase is `https://portainer.pirocorp.com`.

### Validation Checkpoint

Confirm:

- `https://192.168.0.10:9443` opens
- Portainer shows the local Docker environment
- `https://portainer.pirocorp.com` opens successfully through NPM

### State Before Next Phase

The base platform stack is complete and accessible through the current homelab naming model.

## Phase 11 - Final Platform Readiness Checklist

### Goal

Verify that the platform is ready before the first workload is deployed.

### Commands / Config

Run these checks on the server:

```bash
hostnamectl
docker ps
df -h
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT
sudo ufw status verbose
resolvectl status
```

From a LAN client, validate:

- `https://adguard.pirocorp.com`
- `http://npm.pirocorp.com:81`
- `https://portainer.pirocorp.com`
- Samba access to `\\192.168.0.10` or `\\piroman-server`

### Important Decisions / Caveats

- Do not start workload deployment until DNS, reverse proxying, storage mounts, and Portainer access all work as expected.
- If a future workload runbook assumes `*.pirocorp.com`, the LAN clients must already be using AdGuard for DNS.

### Validation Checkpoint

The platform is ready when all of the following are true:

- SSH key login works
- UFW is enabled
- Docker runs without `sudo`
- `/srv/docker` exists and is writable
- the `/mnt/*` storage mounts are present
- Samba shares are reachable
- AdGuard Home resolves names for at least one LAN client
- NPM serves HTTPS with the wildcard certificate
- Portainer is reachable both directly and via `portainer.pirocorp.com`

### State Before First Workload

You now have the infrastructure required for workload-specific runbooks.

Good next docs after this point:

- [Plex](../services/plex/README.md)
- [Immich](../services/immich/README.md)
- [Nextcloud](../services/nextcloud/README.md)
- [qBittorrent](../services/qbittorrent/README.md)
