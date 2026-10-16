# Unified Storage v0.8 — deployed as-built (2026-10-16 CEST)

Status: **Accepted on the Ubuntu host and visually in Homepage, 2026-10-16 CEST**.
Scope: Host-native SMART and capacity diagnostics, the published redacted JSON, systemd selector and the single Homepage **Storage** card.
History: [Phase-2 v0.6/v0.7 acceptance](./README.md); [Homepage operations](../homepage/README.md).

> This documents the **actually deployed** behavior, not a new installer. Collector source, private configuration and SMART baseline remain only on the server. No disk serials, by-id selectors, private state or credentials belong in GitHub.

## Final user-facing design

- Homepage group: `Storage`; **one service card** named `Storage`, `mdi-harddisk` icon, link to authenticated `https://glances.pirocorp.com`.
- Row 1: `SYSTEM SSD`, `LP`, `DATA`. Row 2: `IA`, `COMP`, `IAC`.
- Each of the **six tiles** combines **filesystem used percentage · disk temperature · SMART status**. Missing filesystems or an unreadable/sleeping SMART drive must be represented explicitly, not mislabeled as healthy or 0% usage.
- A **seventh, full-width tile** below the six shows the separate `LAST UPDATED` publication time; it is not duplicated inside each disk tile.
- `Storage` uses `style: row`, `columns: 1`, so the card is full width. Other Homepage groups, widgets, NUT, PeaNUT, Glances info strip, Docker network, and proxy configuration are unchanged.
- Dashboard main-service-card count changed from **17 to 16** by consolidating the prior Disk Space and Disk Health cards; there remain **three native `customapi` widget blocks inside this single card**.

## Collection and data contract

```text
Host smartctl 7.5 + fixed private disk identities + os.statvfs (six mount points)
    -> /srv/docker/disk-health/collector/collect-v0.8.py --write
    -> disk-health.service (oneshot, root)
       ^ disk-health.timer (every 10 minutes; unchanged)
    -> private state under /srv/docker/disk-health/state/ (locked baseline, sticky alarms)
    -> /srv/docker/disk-health/data/status.json (redacted; atomic publication)
    -> Homepage read-only volume at /app/public/disk-health/
    -> Homepage local HTTP http://127.0.0.1:3000/disk-health/status.json
    -> native customapi widgets (refreshInterval: 30000 ms)
```

v0.8 **adds** `overview.<id>.label` and `overview.<id>.dated_label` for stable IDs `system`, `lp`, `data`, `ia`, `comp`, `iac`, while retaining `schema_version: 1`, `disks`, `space`, `last_updated`, `generated_at` and the v0.7 private-state logic. `label` is the **short** combined value and is used by Homepage; `dated_label` adds the publication time and is **not** used because the seventh tile shows the timestamp separately. The `overview` keys avoid reliance on either Glances positional filesystem indices or `disks` list ordering.

Example, **observed** in the final 11:40 CEST JSON check (other fields including `dated_label` omitted in this excerpt):

```json
{
  "overview": {
    "system": {"label": "11.0% · 27°C · Healthy"},
    "lp": {"label": "51.9% · 36°C · Healthy"},
    "data": {"label": "46.2% · 37°C · Healthy"},
    "ia": {"label": "80.0% · 33°C · Healthy"},
    "comp": {"label": "33.9% · 36°C · Healthy"},
    "iac": {"label": "51.4% · 32°C · Healthy"}
  },
  "last_updated": "16.10.2026 11:40"
}
```

These temperatures/capacity readings are a **snapshot**, not constants or acceptance thresholds. A healthy SMART status does **not** imply zero risk. The publicly served JSON must contain **no private disk selectors, serials, raw SMART dumps, secrets, baseline or hidden alert history**.

## Active systemd override

The original root-owned service file still selects `collect-v0.6.py --write`; the original timer remains unchanged. The previous `/etc/systemd/system/disk-health.service.d/10-collector-v07.conf` remains in place and the new, later-sorted override selects v0.8:

```ini
# /etc/systemd/system/disk-health.service.d/20-collector-v08.conf
[Service]
ExecStart=
ExecStart=/usr/bin/python3 -B /srv/docker/disk-health/collector/collect-v0.8.py --write
```

An empty `ExecStart=` resets the earlier command; the last drop-in takes precedence. At acceptance, `systemctl show disk-health.service -p ExecStart -p Result -p ExecMainStatus` confirmed v0.8 with `Result=success` and `ExecMainStatus=0`. The source `collect-v0.8.py` was normalized to `root:root`, mode `0644`, without the unnecessary inherited `+` ACL. The systemd oneshot service is normally inactive between successful runs; the timer remains enabled and active.

## Exact Homepage configuration

Current `/srv/docker/homepage/config/settings.yaml` layout excerpt:

```yaml
layout:
  Storage:
    style: row
    columns: 1
  Infrastructure:
    style: row
    columns: 5
```

Current `/srv/docker/homepage/config/services.yaml` **complete Storage section**:

```yaml
- Storage:
    - Storage:
        icon: mdi-harddisk
        href: https://glances.pirocorp.com
        description: Disk space & SMART health · 6 drives
        widgets:
          - type: customapi
            url: http://127.0.0.1:3000/disk-health/status.json
            display: block
            refreshInterval: 30000
            mappings:
              - field: overview.system.label
                label: SYSTEM SSD
                format: text
              - field: overview.lp.label
                label: LP
                format: text
              - field: overview.data.label
                label: DATA
                format: text

          - type: customapi
            url: http://127.0.0.1:3000/disk-health/status.json
            display: block
            refreshInterval: 30000
            mappings:
              - field: overview.ia.label
                label: IA
                format: text
              - field: overview.comp.label
                label: COMP
                format: text
              - field: overview.iac.label
                label: IAC
                format: text

          - type: customapi
            url: http://127.0.0.1:3000/disk-health/status.json
            display: block
            refreshInterval: 30000
            mappings:
              - field: last_updated
                label: LAST UPDATED
                format: text
```

All three widgets read the same redacted file from within Homepage. `format: text` ensures status strings such as `Healthy`, `Warning`, `Unknown` and `Unavailable` are shown without numeric coercion. Capacity measurement updates are controlled by the **ten-minute systemd timer**, not by the 30-second Homepage fetch rate. No new Docker container, static endpoint, port, DNS rewrite or external data store was needed.

## Acceptance evidence and limits

1. v0.8 `--preview` on the actual host: all six SMART records `Healthy`, all six capacity records `Available` and all six combined preview values; preview made **no writes**.
2. Isolated staging `--write` against a temporary copy of `last-observed.json`: `STAGING WRITE TEST PASSED`, six each of SMART, capacity and combined labels; `last_updated` remained separate and **production files were not modified** by the test.
3. Production systemd override activated using `daemon-reload` and a manual `systemctl start disk-health.service`. Journal at **2026-10-16 09:36:06 UTC** confirms six Healthy SMART, six Available capacity results, `STATE + STATUS WRITTEN. Baseline unchanged.` and successful completion (`Result=success`, `ExecMainStatus=0`).
4. Production `status.json` inspection showed six matching `overview.<id>.label` entries plus `LAST UPDATED: 16.10.2026 11:40`.
5. Homepage visual acceptance screenshot at **16.10.2026 11:50 CEST** shows the correct **3×2 + full-width LAST UPDATED** layout and the other dashboard groups intact. The two YAML backups were created and the change was applied without a Docker restart.
6. **Not separately proved for v0.8 in captured logs:** first fully automatic journal invocation or post-reboot persistence. Historical v0.6/v0.7 automatic timer evidence is in the parent runbook. The screenshot timestamp advanced to 11:50 but is not a substitute for a dedicated post-reboot test. No real-standby, calibrated temperature-threshold, alert-acknowledgement or iPhone-layout acceptance was claimed.

## Checks (read-only unless noted)

```bash
systemctl show disk-health.service -p ExecStart -p Result -p ExecMainStatus --no-pager
systemctl list-timers --all disk-health.timer --no-pager
sudo journalctl -u disk-health.service -n 30 --no-pager
sudo python3 -m json.tool /srv/docker/disk-health/data/status.json >/dev/null

python3 - <<'PY'
import json
with open("/srv/docker/disk-health/data/status.json") as f:
    status = json.load(f)
for name in ("system", "lp", "data", "ia", "comp", "iac"):
    print(name, status["overview"][name]["label"])
print("LAST UPDATED:", status["last_updated"])
PY
```

Do not reinitialize the baseline for a cosmetic dashboard issue; do not run `smartctl` repeatedly just to make the card look fresh.

## Safe rollback — revert Homepage **before** Collector

The v0.7 collector **does not publish `overview`**. Therefore, **never deactivate the v0.8 override while leaving Homepage configured for `overview.*.label`**.

1. Inspect and reconcile the paired pre-v0.8 Homepage backups; do not blindly overwrite unrelated subsequent changes:
   - `/srv/docker/homepage/config/services.yaml.bak-20261016-095334`
   - `/srv/docker/homepage/config/settings.yaml.bak-20261016-095334`
2. Restore the former two Storage cards (Disk Space reading `space.*.label`; Disk Health reading `disks.N.label` plus `last_updated`) and `Storage.columns: 2`. Verify they render while **v0.8 is still active** (it keeps both previous JSON fields).
3. Remove **only** `/etc/systemd/system/disk-health.service.d/20-collector-v08.conf` after verifying its content, then `sudo systemctl daemon-reload`. The retained `10-collector-v07.conf` now selects v0.7. Confirm `ExecStart` before allowing the next timed execution.
4. Do not delete private state or change the baseline, timer, host NUT, Docker storage/network mounts or UPS configuration.

The original v0.7 → v0.6 rollback is separately documented as **historical** in [the Phase-2 runbook](./README.md); it requires a compatible Glances-backed Homepage configuration before removing the v0.7 override.
