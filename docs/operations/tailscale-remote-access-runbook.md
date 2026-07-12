# Tailscale Remote Access Runbook

Status: Ready to execute
Purpose: Implement phase-1 private remote access with Tailscale installed directly on `piroman-server` as the homelab subnet router.
Depends on: [Remote access roadmap](../roadmaps/remote-access/README.md), [Networking and reverse proxy](../platform/networking-and-reverse-proxy.md), [Current state](../overview/current-state.md)
Related docs: [Operations index](./README.md), [Common commands](./common-commands.md)

This runbook keeps the current LAN-first design:

- Tailscale runs on the Ubuntu host, not in Docker.
- `piroman-server` advertises the home LAN route.
- AdGuard Home remains the DNS source for `pirocorp.com`.
- Nginx Proxy Manager remains the HTTPS entry point.
- Services stay private to LAN and VPN clients.

## Preflight

Run these on `piroman-server` before installing anything:

```bash
hostnamectl
ip -4 addr show scope global
ip -4 route show
ip route get 1.1.1.1
docker ps
sudo ufw status verbose
```

Confirm and write down:

| Item | Confirmed value |
| --- | --- |
| Hostname | `piroman-server` |
| OS | `Ubuntu 26.04 LTS` |
| Kernel | `Linux 7.0.0-22-generic` |
| Server LAN IP | `192.168.0.10/24` |
| LAN CIDR to advertise | `192.168.0.0/24` |
| LAN interface | `enp2s0` |
| LAN gateway | `192.168.0.1` |
| Default route | `default via 192.168.0.1 dev enp2s0` |
| AdGuard DNS IP | `192.168.0.10` |
| Service DNS zone | `pirocorp.com` |
| Docker bridge networks | Present under `172.17.0.0/16` and other `172.x.0.0/16` ranges |
| UFW baseline | Active; `deny incoming`, `allow outgoing`, `deny routed` |

Preflight confirmed the LAN route as `192.168.0.0/24` on `enp2s0`. Advertise only this LAN CIDR through Tailscale; do not advertise Docker bridge networks.

## Install Tailscale On The Host

Run on `piroman-server`:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo systemctl enable --now tailscaled
sudo tailscale up --accept-dns=false
```

If the install script stops during `apt update` because an unrelated third-party repository is broken, disable that repository first, then install Tailscale from the repo the script already added.

Example observed issue:

```text
https://packagecloud.io/ookla/speedtest-cli/ubuntu resolve Release
404 Not Found
```

Recovery:

```bash
grep -Ril 'packagecloud.io/ookla/speedtest-cli' /etc/apt/sources.list /etc/apt/sources.list.d
sudo mv <OOKLA_SOURCE_FILE> <OOKLA_SOURCE_FILE>.disabled
sudo apt update
sudo apt install -y tailscale
sudo systemctl enable --now tailscaled
```

Open the authentication URL printed by `tailscale up`, sign in, and confirm `piroman-server` appears in the Tailscale admin console.

Verify on the server:

```bash
tailscale ip -4
tailscale status
tailscale netcheck
```

`--accept-dns=false` is intentional. This host already has a local DNS role, so Tailscale should not rewrite the server's own resolver behavior.

Live checkpoint:

- Tailscale package installed successfully: `tailscale 1.98.8`
- `tailscaled.service` was created and enabled
- Browser authentication completed for `piroman-server` in the `pirocorp.github` tailnet
- Tailscale IPv4 assigned: `100.94.205.122`
- `tailscale netcheck` reports UDP connectivity available and nearest DERP as Amsterdam
- Server reported a pending kernel upgrade from `7.0.0-22-generic` to `7.0.0-27-generic`; reboot later after VPN validation or during a planned maintenance window

## Enable Subnet Routing

Enable Linux packet forwarding:

```bash
printf 'net.ipv4.ip_forward = 1\nnet.ipv6.conf.all.forwarding = 1\n' | sudo tee /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

Advertise the confirmed LAN route:

```bash
sudo tailscale set --advertise-routes=<LAN_CIDR>
```

Example only after confirmation:

```bash
sudo tailscale set --advertise-routes=192.168.0.0/24
```

Keep default subnet-route SNAT enabled for phase 1. Do not advertise `0.0.0.0/0`; that would make the host an exit node, which is not part of this design.

Live checkpoint:

- IP forwarding enabled in `/etc/sysctl.d/99-tailscale.conf`
- `net.ipv4.ip_forward = 1`
- `net.ipv6.conf.all.forwarding = 1`
- Advertised route configured: `192.168.0.0/24`
- Advertised subnet route approved in the Tailscale admin console: `192.168.0.0/24`
- Tailscale admin console warns that key expiry is enabled; consider disabling key expiry for `piroman-server` after remote access validation is complete

## Approve The Route

In the Tailscale admin console:

1. Open `Machines`.
2. Select `piroman-server`.
3. Open the `Subnets` section.
4. Approve the advertised LAN route.
5. Leave key expiry enabled for initial testing; consider disabling it later only after the setup is stable.

Live checkpoint:

- Approved route: `192.168.0.0/24`
- Exit node: not allowed, as intended
- Key expiry: enabled for now

If tailnet access controls are locked down, add a rule or grant that allows trusted users/devices to reach `<LAN_CIDR>`. Route approval and access rules are separate controls.

## Optional Later Key Expiry Change

Key expiry is currently enabled for `piroman-server`. That is okay during initial setup and validation.

After remote access is confirmed working, consider disabling key expiry for `piroman-server` in the Tailscale admin console so the subnet router does not unexpectedly require reauthentication later.

Only do this for trusted, stable infrastructure devices. If the server is ever replaced or compromised, revoke the device from the Tailscale admin console.

## Optional Later Router Forward

Do not configure router port forwarding for the initial deployment. Tailscale normally works through outbound connections, NAT traversal, and automatic relay fallback.

Optional later optimization:

```text
UDP 41641 -> 192.168.0.10
```

Forwarding UDP `41641` from the ISP router to `piroman-server` can improve direct connection performance in some strict-NAT cases, but it is not required to start.

Do not forward SSH, SMB, AdGuard, Portainer, Nginx Proxy Manager, or other homelab service ports publicly.

## Configure Split DNS

In the Tailscale admin console DNS page:

1. Add a custom nameserver.
2. Use `192.168.0.10`.
3. Restrict it to the domain `pirocorp.com`.
4. Keep it as split DNS for `pirocorp.com`, not a global DNS override.

This makes remote Tailscale clients ask AdGuard Home for names like:

```text
nextcloud.pirocorp.com
immich.pirocorp.com
server.pirocorp.com
```

The expected DNS answer for those names remains the private LAN address, usually `192.168.0.10`.

Live checkpoint:

- Split DNS nameserver added: `192.168.0.10`
- Restricted domain: `pirocorp.com`
- Windows client `piroman` joined the tailnet with Tailscale IP `100.87.10.92`
- Windows `Resolve-DnsName nextcloud.pirocorp.com` returns `192.168.0.10`
- Windows `Test-NetConnection 192.168.0.10 -Port 443` succeeds through interface `Tailscale`

## Enroll First Clients

Install Tailscale on one trusted laptop and one trusted phone. Sign in with the same tailnet.

Windows, macOS, iOS, and Android clients should accept subnet routes automatically. For a Linux client, run:

```bash
sudo tailscale set --accept-routes
```

## Validate From Outside The LAN

Use mobile data or another external network, then connect Tailscale.

On Windows:

```powershell
tailscale status
Resolve-DnsName nextcloud.pirocorp.com
Test-NetConnection 192.168.0.10 -Port 443
Test-NetConnection 192.168.0.10 -Port 22
```

On Linux or macOS:

```bash
tailscale status
nslookup nextcloud.pirocorp.com
curl -I https://nextcloud.pirocorp.com
ssh piroman@192.168.0.10
```

Browser checks:

```text
https://server.pirocorp.com
https://nextcloud.pirocorp.com
https://immich.pirocorp.com
https://plex.pirocorp.com
https://portainer.pirocorp.com
http://npm.pirocorp.com:81
```

SMB check from Windows:

```text
\\192.168.0.10\data
```

Acceptance criteria:

- A remote client can connect to Tailscale.
- `*.pirocorp.com` resolves through AdGuard Home while remote.
- Existing HTTPS service names open through Nginx Proxy Manager.
- SSH to `piroman-server` works through the VPN.
- SMB shares work through the VPN.
- Local LAN access still works when Tailscale is disconnected.

Live checkpoint:

- Same-LAN Windows validation succeeded over the Tailscale interface.
- Remaining validation: repeat DNS and service checks from outside the LAN, such as phone mobile data or Windows through a phone hotspot.

## Troubleshooting

Check route advertisement on the server:

```bash
tailscale status
tailscale ip -4
sudo sysctl net.ipv4.ip_forward
sudo sysctl net.ipv6.conf.all.forwarding
```

If DNS fails but raw IP access works:

- confirm the split DNS nameserver is `192.168.0.10`
- confirm it is restricted to `pirocorp.com`
- confirm the route to the LAN CIDR is approved
- confirm AdGuard still has the `*.pirocorp.com -> 192.168.0.10` rewrite

If LAN device access fails but host access works:

- confirm the advertised CIDR matches the real LAN
- confirm the route is approved in Tailscale
- check `sudo ufw status verbose`
- check the actual LAN interface from `ip route get 1.1.1.1`

If a Linux client cannot reach the LAN route:

```bash
sudo tailscale set --accept-routes
```

## Rollback

Disable the subnet route first in the Tailscale admin console, then run on `piroman-server`:

```bash
sudo tailscale set --advertise-routes=
sudo systemctl disable --now tailscaled
```

To fully remove the package later:

```bash
sudo apt remove tailscale
sudo rm /etc/sysctl.d/99-tailscale.conf
sudo sysctl -w net.ipv4.ip_forward=0
sudo sysctl -w net.ipv6.conf.all.forwarding=0
```

Only turn forwarding back off if no other local service depends on it.

## Post-Implementation Documentation

After live validation succeeds, update:

- [Current state](../overview/current-state.md): add Tailscale remote access as implemented.
- [Service inventory](../overview/service-inventory.md): move remote access from planned to implemented platform component.
- [Remote access roadmap](../roadmaps/remote-access/README.md): mark phase 1 deployed and record the actual LAN CIDR.

## Official References

- Tailscale Linux install: https://tailscale.com/docs/install/linux
- Tailscale subnet routers: https://tailscale.com/docs/features/subnet-routers
- Tailscale DNS and split DNS: https://tailscale.com/docs/reference/dns-in-tailscale
