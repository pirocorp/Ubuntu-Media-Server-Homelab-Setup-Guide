# IPv6 Leak Validation And Mitigation Runbook

Status: Implemented  
Purpose: Detect unintended IPv6 Internet egress and disable public IPv6 on the physical LAN interface while preserving IPv4, Docker networking, and overlay/VPN interfaces such as Tailscale.  
Applies to: Ubuntu Server homelab host  
Validated: 2026-09-20  
Related docs: [Operations index](./README.md), [Tailscale remote access runbook](./tailscale-remote-access-runbook.md), [Networking and reverse proxy](../platform/networking-and-reverse-proxy.md)

## Scope

In this runbook, an **IPv6 leak** means that a host can reach the public Internet over IPv6 when the intended network/security path is IPv4-only or when a VPN/tunnel is expected to carry all Internet traffic but does not capture IPv6.

This matters because IPv4 NAT does **not** protect or translate native IPv6 traffic. If the host has a globally routable IPv6 address and an IPv6 default route, applications may use IPv6 directly.

The mitigation used on this homelab is intentionally **per-interface**:

- disable IPv6 on the physical Ethernet interface;
- keep IPv4 on that interface;
- do not globally disable the IPv6 stack;
- leave Docker, loopback, and Tailscale interfaces alone unless there is a separate reason to change them.

## Address Types To Recognize

Before changing anything, distinguish these address types:

| Address/prefix | Meaning | Internet-routable? | Expected action |
| --- | --- | --- | --- |
| `::1/128` | IPv6 loopback | No | Leave alone |
| `fe80::/10` | IPv6 link-local | No | Normally leave alone |
| `fd00::/8` | IPv6 Unique Local Address (ULA) / private overlay | No, not globally routed | Usually leave alone |
| `2000::/3` | IPv6 Global Unicast range | Yes | Investigate if IPv6 Internet egress is not intended |

Example: Tailscale may use an address under `fd7a:115c:a1e0::/48`. That is an overlay-network address and is not the ISP-provided public IPv6 address.

## 1. Identify The Physical LAN Interface

Do not assume the interface name on another host.

```bash
ip -br addr
ip route
```

On the current homelab host the physical LAN interface is:

```text
enp2s0
```

Confirm this before applying the commands below.

## 2. Baseline IPv6 State

Inspect addresses and routes:

```bash
ip -6 addr show dev enp2s0
ip -6 route
```

A public IPv6 configuration typically shows:

- a Global Unicast address on `enp2s0`, often beginning with `2...` or `3...`;
- a default IPv6 route, often learned through Router Advertisement (`proto ra`);
- a link-local gateway beginning with `fe80::`.

A link-local gateway is normal. `fe80::/10` is **not** IPv6 localhost. IPv6 localhost is `::1/128`.

## 3. Test Actual Internet Egress

Test IPv4 first:

```bash
curl -4 --connect-timeout 5 https://ifconfig.co
```

Expected result:

```text
<public IPv4 address>
```

Now force IPv6:

```bash
curl -6 --connect-timeout 5 https://ifconfig.co
```

### Interpretation

If the IPv6 command returns a public IPv6 address, the host has working IPv6 Internet egress.

If the security design expects IPv4-only egress, this is an IPv6 leak path.

Do not rely only on `ip -6 route`. The most useful validation is whether an application can actually establish an IPv6 Internet connection.

## 4. Temporarily Disable IPv6 On The Physical Interface

Apply the change to the running kernel:

```bash
sudo sysctl -w net.ipv6.conf.enp2s0.disable_ipv6=1
```

This affects only `enp2s0`.

Verify the kernel setting:

```bash
sysctl net.ipv6.conf.enp2s0.disable_ipv6
```

Expected:

```text
net.ipv6.conf.enp2s0.disable_ipv6 = 1
```

Check that IPv6 addresses have disappeared from the interface:

```bash
ip -6 addr show dev enp2s0
```

Expected: no IPv6 addresses on `enp2s0`.

## 5. Validate The Mitigation

Verify that IPv4 still works:

```bash
curl -4 --connect-timeout 5 https://ifconfig.co
```

Expected: public IPv4 address.

Verify that public IPv6 egress no longer works:

```bash
curl -6 --connect-timeout 5 https://ifconfig.co
```

Expected examples:

```text
curl: (...) Network is unreachable
```

or:

```text
curl: (28) Connection timed out ...
```

The exact error can vary. The important result is that the forced IPv6 connection does **not** succeed.

Also check:

```bash
ip -6 addr show dev enp2s0
ip -6 route
```

`fe80::` routes on Docker, veth, Tailscale, or other internal interfaces are not by themselves evidence of an IPv6 Internet leak.

If a previously learned `proto ra` route is still displayed immediately after the change, allow its lifetime to expire or reboot and check again. The decisive test is that the physical interface has no IPv6 address and forced IPv6 Internet egress fails.

## 6. Make The Change Persistent

Create a dedicated sysctl file:

```bash
sudo nano /etc/sysctl.d/99-disable-ipv6-enp2s0.conf
```

Add exactly:

```text
net.ipv6.conf.enp2s0.disable_ipv6 = 1
```

Save the file and apply sysctl configuration:

```bash
sudo sysctl --system
```

Confirm:

```bash
sysctl net.ipv6.conf.enp2s0.disable_ipv6
```

Expected:

```text
net.ipv6.conf.enp2s0.disable_ipv6 = 1
```

## 7. Post-Reboot Validation

After the next reboot, verify all three conditions:

```bash
ip -6 addr show dev enp2s0
sysctl net.ipv6.conf.enp2s0.disable_ipv6
curl -6 --connect-timeout 5 https://ifconfig.co
```

Expected:

1. `enp2s0` has no IPv6 addresses.
2. `disable_ipv6 = 1`.
3. The forced IPv6 Internet request fails.

Also confirm IPv4 remains healthy:

```bash
curl -4 --connect-timeout 5 https://ifconfig.co
```

## 8. Quick Audit

Use this when checking the server later:

```bash
IFACE=enp2s0

echo "=== IPv6 setting ==="
sysctl "net.ipv6.conf.${IFACE}.disable_ipv6"

echo
echo "=== IPv6 addresses on ${IFACE} ==="
ip -6 addr show dev "$IFACE"

echo
echo "=== IPv6 routes ==="
ip -6 route

echo
echo "=== IPv4 Internet ==="
curl -4 --connect-timeout 5 https://ifconfig.co || true

echo
echo
echo "=== IPv6 Internet test (expected to FAIL) ==="
if curl -6 -fsS --connect-timeout 5 https://ifconfig.co; then
    echo
    echo "FAIL: public IPv6 egress is available."
else
    echo "PASS: no public IPv6 egress detected."
fi
```

## 9. Why IPv6 Is Not Disabled Globally

Avoid this unless there is a specific reason:

```text
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
```

The homelab deliberately uses a narrower control:

```text
net.ipv6.conf.enp2s0.disable_ipv6 = 1
```

This removes native IPv6 from the physical LAN/uplink interface without unnecessarily changing IPv6 behavior on:

- `tailscale0`;
- Docker bridges;
- Docker `veth` interfaces;
- loopback;
- future internal/overlay interfaces.

Internal `fe80::` link-local addresses or `fd00::/8` ULA addresses are not equivalent to public IPv6 Internet access.

## 10. Rollback

To re-enable IPv6 on `enp2s0`, remove or comment out the persistent setting:

```bash
sudo rm /etc/sysctl.d/99-disable-ipv6-enp2s0.conf
```

Temporarily re-enable it in the running kernel:

```bash
sudo sysctl -w net.ipv6.conf.enp2s0.disable_ipv6=0
```

Then reboot to ensure the interface receives fresh Router Advertisements and autoconfiguration cleanly:

```bash
sudo reboot
```

After reboot:

```bash
ip -6 addr show dev enp2s0
ip -6 route
curl -6 --connect-timeout 5 https://ifconfig.co
```

## Troubleshooting

### `curl -6` times out instead of saying "Network is unreachable"

That is acceptable for this validation. Different resolver, routing, and socket states can produce different errors. The pass condition is that no public IPv6 connection succeeds.

### `ip -6 route` still shows many `fe80::/64` entries

This is normal when Docker or other virtual interfaces exist. `fe80::/10` is link-local and cannot be routed across the public Internet.

### Tailscale still has an IPv6-looking address

Expected. Tailscale can maintain its own private overlay address space independently of native ISP IPv6. The mitigation in this runbook intentionally targets only `enp2s0`.

### A service stops working after the change

First determine whether that service explicitly depends on native IPv6:

```bash
docker ps
ss -lntup
journalctl -b --priority=warning
```

If necessary, temporarily roll back using the procedure above and investigate before making the change persistent again.

## Security Note

Disabling native IPv6 removes one possible unintended egress path, but it is **not** a substitute for:

- a host firewall;
- correct router firewall policy;
- VPN kill-switch rules when a VPN is required;
- DNS privacy controls;
- application-level TLS.

For VPN use, the correct security property is **fail closed**: when the tunnel is down, neither IPv4 nor IPv6 should be able to bypass the intended tunnel policy.

## References

- Linux kernel IP sysctl documentation — `disable_ipv6`:  
  https://kernel.org/doc/html/latest/networking/ip-sysctl.html
- Linux kernel IPv6 documentation:  
  https://kernel.org/doc/html/latest/networking/ipv6.html
- RFC 4291 — IPv6 Addressing Architecture:  
  https://www.rfc-editor.org/rfc/rfc4291
- curl command-line documentation (`-4` / `-6`):  
  https://curl.se/docs/manpage.html
