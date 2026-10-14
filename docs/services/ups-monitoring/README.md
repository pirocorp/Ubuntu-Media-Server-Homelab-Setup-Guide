# PowerWalker UPS / NUT — host-native shutdown and monitoring

Status: **Implemented** — existing NUT retained; PeaNUT added and Netdata removed on 2026-10-14.  
Purpose: Record the live NUT safety boundary and basic operating checks, then link to the reproducible dashboard deployment runbook.  
Related docs: [Server Health Dashboard](../server-health/README.md), [Homepage](../homepage/README.md), [Services index](../README.md), [Legacy Netdata guide (archive)](../../archive/netdata-legacy-setup.md).

## Current architecture

```text
PowerWalker USB UPS
  -> Ubuntu NUT driver and upsd (host)
     -> NUT upsmon / automatic safe shutdown (host; authoritative)
     -> read-only status queries on 127.0.0.1:3493
           -> PeaNUT 6.0.0 (Docker host network; authenticated web/API)
                 -> Homepage UPS widget + HTTPS peanut.pirocorp.com
```

The host NUT instance and its safe-shutdown responsibilities were **not** moved into Docker and **not** reconfigured to listen on LAN or `0.0.0.0`. PeaNUT's visual interface is optional to the safety mechanism. No Docker dashboard has been given the responsibility to power down the host.

### Observed live setup

| Property | Value |
| --- | --- |
| Ubuntu host | `piroman-server` / `192.168.0.10` |
| UPS identifier | `powerwalker` |
| NUT server | `127.0.0.1:3493` and IPv6 loopback |
| NUT daemon | `nut-server` (native systemd service) |
| NUT monitoring/shutdown | Existing host NUT / upsmon logic unchanged |
| PeaNUT interface | `https://peanut.pirocorp.com` → `192.168.0.10:61210` |
| PeaNUT image | `brandawg93/peanut:6.0.0` |
| Homepage | `https://home.pirocorp.com`; UPS charge, load and raw status |
| Netdata | **Removed** from Ubuntu, NPM, AdGuard and UFW on 2026-10-14 |

Do not replace the real `powerwalker` name with legacy generic examples such as `myups`; do not copy sample UPS driver/user/password settings from the retired guide into production.

## Read-only verification without modifying shutdown

```bash
systemctl is-active nut-server
upsc -l
upsc powerwalker@127.0.0.1 battery.charge
upsc powerwalker@127.0.0.1 ups.load
upsc powerwalker@127.0.0.1 ups.status
sudo ss -lntp '( sport = :3493 )'
```

Validated historical readings: 100% battery, 8–10% load and `ups.status: OL CHRG`, meaning online and charging. Values fluctuate and must not be hardcoded. Some `upsc` runs show `Init SSL without certificate database` while still successfully returning UPS fields.

**Safety requirements:**

- Do not edit `/etc/nut/ups.conf`, `upsd.conf`, `upsmon.conf` or `upsd.users` just to make a dashboard reachable.
- NUT `3493/tcp` must stay host-loopback-only. No port-forwarding, external listener, LAN firewall opening or privileged NUT shutdown credentials for PeaNUT.
- PeaNUT can expose UPS command API endpoints; its web login is required, but monitor-only usage alone does not make its API intrinsically read-only.
- A failed PeaNUT, Homepage, or Glances UI must **not** interrupt NUT's independent automatic shutdown.

## Deploy, back up or troubleshoot the dashboards

Use the [complete as-built Server Health runbook](../server-health/README.md), including tested Glances and PeaNUT Compose configurations, HTTPS/AdGuard/UFW, PeaNUT `6.0.0` auth-path and healthcheck workarounds, Homepage file secrets, and rollback.

The former long **Netdata installation runbook has been archived** [here](../../archive/netdata-legacy-setup.md) and must not be used for rebuilding this host. Former URL `server.pirocorp.com` and port `19999` are retired. Netdata was removed only after Homepage + Glances + PeaNUT acceptance.
