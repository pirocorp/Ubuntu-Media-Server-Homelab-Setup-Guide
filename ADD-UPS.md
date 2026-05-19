# Home Server UPS + Netdata + Nginx Proxy Manager Setup Guide

# Overview

This guide documents the complete setup for:

- UPS monitoring on Ubuntu
- Netdata monitoring
- NUT (Network UPS Tools)
- Nginx Proxy Manager
- AdGuard local DNS
- UFW firewall
- Local-only secure access

---

# Final Architecture

```text
Browser
   ↓
server.home
   ↓
AdGuard DNS Rewrite
   ↓
192.168.0.10
   ↓
Nginx Proxy Manager
   ↓
Netdata
   ↓
UPS Metrics via NUT
```

---

# Part 1 — Install UPS Support (NUT)

Install Network UPS Tools.

```bash
sudo apt update
# Updates Ubuntu package indexes

sudo apt install nut
# Installs Network UPS Tools (NUT)
```

---

# Part 2 — Detect UPS

Verify Ubuntu detects the UPS over USB.

```bash
lsusb
# Lists connected USB devices
```

Look for:
- APC
- Eaton
- CyberPower
- etc.

---

# Part 3 — Configure NUT

---

## Configure UPS Device

Edit:

```bash
sudo nano /etc/nut/ups.conf
# Opens UPS device configuration
```

Example configuration:

```ini
[myups]
driver = usbhid-ups
port = auto
desc = "Home Server UPS"
```

Explanation:
- `driver` → UPS driver
- `port = auto` → automatic USB detection
- `desc` → human-readable description

---

## Configure NUT Mode

Edit:

```bash
sudo nano /etc/nut/nut.conf
# Opens NUT mode configuration
```

Set:

```ini
MODE=standalone
```

Explanation:
- `standalone` means:
  - local UPS
  - local monitoring
  - no network UPS server

---

## Configure Monitoring

Edit:

```bash
sudo nano /etc/nut/upsmon.conf
# Opens UPS monitoring configuration
```

Add:

```ini
MONITOR myups@localhost 1 monuser password master
```

Explanation:
- `myups` → UPS name from ups.conf
- `localhost` → monitor locally
- `monuser` → monitoring user
- `password` → monitoring password
- `master` → this machine controls shutdown logic

---

## Create Monitoring User

Edit:

```bash
sudo nano /etc/nut/upsd.users
# Opens UPS daemon users configuration
```

Add:

```ini
[monuser]
password = password
upsmon master
```

Explanation:
- creates UPS monitoring user
- allows upsmon access

---

# Part 4 — Start NUT Services

Restart services:

```bash
sudo systemctl restart nut-server
# Restarts NUT server daemon

sudo systemctl restart nut-monitor
# Restarts UPS monitoring daemon
```

Enable startup on boot:

```bash
sudo systemctl enable nut-server
# Enables nut-server on boot

sudo systemctl enable nut-monitor
# Enables nut-monitor on boot
```

---

# Part 5 — Verify UPS Monitoring

Run:

```bash
upsc myups
# Displays UPS statistics and status
```

Example output:

```text
battery.charge: 100
battery.runtime: 3600
ups.status: OL
```

Explanation:
- `battery.charge` → battery %
- `battery.runtime` → estimated runtime
- `OL` → Online power

---

# Part 6 — Install Netdata

Install Netdata monitoring.

```bash
bash <(curl -Ss https://my-netdata.io/kickstart.sh)
# Downloads and installs Netdata automatically
```

---

# Part 7 — Verify Netdata

Open:

```text
http://192.168.0.10:19999
```

Expected:
- Netdata dashboard loads
- system metrics visible

---

# Part 8 — Enable UPS Metrics in Netdata

Netdata automatically detects NUT.

Restart Netdata:

```bash
sudo systemctl restart netdata
# Restarts Netdata service
```

Check logs:

```bash
sudo journalctl -u netdata -f
# Shows live Netdata logs
```

Expected:
- UPS charts appear automatically

---

# Part 9 — Configure Netdata Network Binding

You wanted:
- LAN-only access
- no global exposure
- clean networking

---

## Edit Netdata Configuration

```bash
sudo nano /etc/netdata/netdata.conf
# Opens Netdata configuration
```

Set:

```ini
[web]
    bind to = 192.168.0.10
```

Explanation:
- binds Netdata only to LAN IP
- safer than:
  ```ini
  bind to = 0.0.0.0
  ```

---

## Restart Netdata

```bash
sudo systemctl restart netdata
# Applies Netdata configuration changes
```

---

## Verify Listening Address

```bash
sudo ss -tulpn | grep 19999
# Shows processes listening on port 19999
```

Expected:

```text
192.168.0.10:19999
```

NOT:

```text
0.0.0.0:19999
```

---

# Part 10 — Configure UFW Firewall

You simplified everything to:
- allow LAN access
- avoid Docker bridge complexity

---

## Allow Netdata Port

```bash
sudo ufw allow 19999/tcp
# Allows TCP access to Netdata
```

Reload firewall:

```bash
sudo ufw reload
# Reloads UFW firewall rules
```

Verify:

```bash
sudo ufw status numbered
# Shows numbered firewall rules
```

Final expected rules:

```text
OpenSSH      ALLOW Anywhere
Samba        ALLOW Anywhere
19999/tcp    ALLOW Anywhere
```

---

# Part 11 — Install Nginx Proxy Manager

NPM runs inside Docker.

Purpose:
- reverse proxy
- local domains
- optional SSL
- centralized access

---

# Part 12 — Configure Local DNS (AdGuard)

Add DNS rewrites:

```text
server.home     → 192.168.0.10
npm.home        → 192.168.0.10
portainer.home  → 192.168.0.10
```

Explanation:
- local DNS shortcuts
- easier service access

---

# Part 13 — Configure NPM Proxy Host

---

## Proxy Host Settings

### Domain

```text
server.home
```

### Scheme

```text
http
```

### Forward Hostname/IP

```text
192.168.0.10
```

### Forward Port

```text
19999
```

---

## Enable Options

Enabled:
- Block Common Exploits
- Websocket Support

Disabled:
- Force SSL
- HTTP/2
- HSTS

Explanation:
- Netdata works best internally over HTTP
- avoids redirect loops

---

# Part 14 — Problems Encountered

---

## 504 Gateway Timeout

Cause:
- NPM could not reach Netdata

Root causes included:
- wrong bind address
- HTTPS redirect loops
- Docker bridge confusion

---

## HSTS Browser Problems

Chrome cached HTTPS for:

```text
server.home
```

This forced HTTPS even when:
- Netdata was HTTP-only

Symptoms:
- endless redirects
- timeout loops
- broken pages

---

## Fix HSTS

Open:

```text
chrome://net-internals/#hsts
```

Delete:

```text
server.home
```

Explanation:
- clears cached HTTPS enforcement

---

# Part 15 — Final Working Configuration

---

## Netdata

Bound only to:

```ini
[web]
    bind to = 192.168.0.10
```

---

## UFW

Simple rule:

```bash
sudo ufw allow 19999/tcp
# Allows Netdata access from LAN
```

---

## NPM

Proxy target:

```text
192.168.0.10:19999
```

---

# Final Working URLs

---

## Direct Access

```text
http://192.168.0.10:19999
```

---

## DNS Access

```text
http://server.home
```

---

# Final Security Design

---

## Netdata

- LAN-only binding
- no WAN exposure

---

## Firewall

- only required ports open

---

## Docker

- no special bridge firewall hacks

---

# Lessons Learned

---

## 1. Simpler Networking Is Better

Direct LAN binding was:
- cleaner
- safer
- easier to debug

---

## 2. HSTS Is Dangerous in Homelabs

Browsers aggressively cache HTTPS rules.

---

## 3. Docker Networking Can Confuse Reverse Proxies

Avoid unnecessary:
- bridge routing
- localhost tricks
- subnet exceptions

---

# Final Result

You now have:

- UPS-backed Ubuntu server
- automatic UPS monitoring
- Netdata monitoring
- UPS metrics inside Netdata
- local DNS
- reverse proxy access
- clean firewall setup
- stable LAN-only monitoring
