# SMART Disk Health — Phase 2 deployed operations runbook

Status: **Deployed 2026-10-15; unified Storage v0.8 and single Homepage card accepted 2026-10-16 (CEST)**  
Purpose: Explain the six-drive SMART + filesystem-capacity collector, locked baseline and timer; preserve the v0.6/v0.7 rollout history and link to the current unified v0.8 Homepage deployment.  
Depends on: [Phase-1 Server Health](../server-health/README.md), [Homepage](../homepage/README.md), existing host `smartctl 7.5`, mounted storage.  
Design history: [original SMART extension roadmap](../../roadmaps/server-health-dashboard/smart-extension.md).

> **As-built reference, not an executable installer.** The collector's Python code and actual disk configuration live **only on the Ubuntu host**, under `/srv/docker/disk-health/`; this repository intentionally does not contain device serials, by-id symlinks, baseline data, or an unverified reproduction of the current Python source. Keep host copies and backups for disaster recovery. Do not run code from this document without checking the live machine.

## Current deployment: unified Storage v0.8

**Current as-built instructions, JSON schema, exact Homepage `services.yaml`/`settings.yaml`, active systemd override, verification and rollback are in [Unified Storage v0.8](./storage-v08-unified.md).** As of 2026-10-16 CEST, **one** Storage card replaces the former two, showing **six combined disk tiles plus LAST UPDATED**. The runbook sections below preserve the original v0.6/v0.7 deployment as **historical acceptance evidence**; their old references to two cards and the v0.7 drop-in are not current configuration instructions. v0.8 retains the same locked baseline, private state, redacted status publishing and ten-minute timer.

## Implementation and acceptance status

- **Confirmed live:** six physical devices identified by validated ATA `/dev/disk/by-id/` links and mount-point cross-check; `smartctl -j -d sat -H -A` works on six drives; initial baseline created and protected. Collector **v0.7** passed `--preview`, isolated staging `--write`, and real `systemctl start disk-health.service` at **2026-10-15 22:31:24–22:31:25 UTC** (00:31 CEST, 16 Oct). The journal showed six Healthy SMART drives, six Available filesystem readings, `STATE + STATUS WRITTEN. Baseline unchanged.` and successful oneshot completion. HTTP GET inside Homepage returned 200 with six `disks` and six `space` entries; the user-confirmed Homepage screenshot shows both working `Storage` cards.
- **Confirmed systemd:** `disk-health.service` linked, loaded and ran successfully via manual `systemctl start` (2026-10-15, 21:33 UTC). `disk-health.timer` is `enabled` and `active`; its **first fully automatic run was independently observed** at 21:40:02–21:40:03 UTC (23:40 CEST): journal shows all six Healthy samples, `STATE + STATUS WRITTEN. Baseline unchanged.` and successful service completion. `systemctl list-timers` reported LAST 21:40:02 UTC and NEXT 21:50 UTC; the open Homepage dashboard updated LAST UPDATED to 23:40.
- **Simulated:** SMART standby skip -> Sleeping; unsupported power-mode -> Unknown; SMART 188 counter increase -> Warning; Warning retained through subsequent Sleeping; last successful temperature/time retained through Sleeping or Unknown; failed second-file publish leaves previous complete public JSON in place; file permissions and atomic individual replacement.
- **Confirmed automatic v0.7:** first timer-triggered cycle at **2026-10-15 22:40:06–22:40:07 UTC (2026-10-16 00:40 CEST)**; journal shows all six SMART disks Healthy, six disk-space mounts Available, `STATE + STATUS WRITTEN. Baseline unchanged.`, and successful oneshot completion. Homepage LAST UPDATED advanced from 00:31 to 00:40 without manual collector invocation.
- **Pending:** reboot persistence; real standby hardware test (the observed Seagate modes IDLE_A/IDLE_B do not prove true standby); calibrated drive-specific temperature and Samsung SSD wear thresholds; full fault/critical escalation tests and acknowledgement workflow; iPhone layout acceptance.

The host remains the only SMART diagnostic reader. No new Docker container, HTTP listener, reverse-proxy host, DNS rewrite, Docker socket access, device mount or firewall rule was added for this feature. Host disks and NTFS/Samba shares were not modified by the monitoring work.

## Six monitored physical devices

Current `/dev/sdX` mappings below are **observations only**, not stable selectors:

| Public label / ID | Host mount | Physical device at acceptance | Media |
| --- | --- | --- | --- |
| SYSTEM SSD / `system` | `/` | `/dev/sde` | Samsung SSD 870 QVO 2 TB |
| IAC / `iac` | `/mnt/iac` | `/dev/sda` | Seagate ST4000VX007 4 TB HDD |
| LP / `lp` | `/mnt/lp` | `/dev/sdb` | Seagate ST8000DM004 8 TB HDD |
| DATA / `data` | `/mnt/data` | `/dev/sdc` | Seagate ST8000DM004 8 TB HDD |
| IA / `ia` | `/mnt/ia` | `/dev/sdd` | Seagate ST4000VX000 4 TB HDD |
| COMP / `comp` | `/mnt/comp` | `/dev/sdf` | Seagate ST4000DM004 4 TB HDD |

Host-local `collector/config.json` maps each ID to `mount`, `kind`, `by_id`, and `smart_type: sat`. The collector verifies that the configured physical symlink agrees with the mounted device and the SHA-256 identity stored in the baseline. **Never commit `config.json` or copy actual disk serials to public JSON.**

SMART attributes tracked on Seagate HDDs: 5 (reallocated), 187 (reported uncorrectable), 188 (Seagate command timeout; three-component `raw16`), 197 (pending), 198 (offline uncorrectable), 199 (interface CRC). Samsung SSD: 5 and 199 error baseline plus normalized 177 wear index and 241 total LBAs written for reference. Normalized wear index `96` is **not a verified percentage of SSD life remaining**; it is not published as a health percentage.

The initial baseline deliberately includes historical IAC SMART 188 `[0,1,1]`, IA SMART 188 `[0,0,3]`, and SSD CRC 199 `6`. These are existing counts, **not evidence of a new incident** by themselves.

## Runtime architecture

```text
Ubuntu smartctl 7.5 + stable by-id config; host os.statvfs for six mounts
   -> /srv/docker/disk-health/collector/collect-v0.7.py --write
      ^ systemd disk-health.service (oneshot, root, journal)
      ^ systemd disk-health.timer (every 10 minutes)
   -> /srv/docker/disk-health/state/baseline.json      [private, immutable by policy]
   -> /srv/docker/disk-health/state/last-observed.json [private, updated on valid reads]
   -> /srv/docker/disk-health/data/status.json         [public to Homepage, atomic replace]
      -> read-only Docker bind mount: /app/public/disk-health/
      -> Homepage HTTP http://127.0.0.1:3000/disk-health/status.json
      -> native Homepage customapi widgets (30-second browser refresh)
      -> Disk Health (six SMART tiles + timestamp)
      -> Disk Space (six fixed-identity filesystem tiles)
      -> https://home.pirocorp.com
```

Host paths and rights:

| File | Mode | Contract |
| --- | --- | --- |
| `collector/config.json` | `600 root:root` | Private ATA by-id identities |
| `state/` | `700 root:root` | Private baseline/observations/alerts |
| `state/baseline.json` | `600 root:root` | Explicitly initialized ONCE; **no periodic reset** |
| `state/last-observed.json` | `600 root:root` | Last successful reading per disk, plus sticky alert records |
| `data/status.json` | `644 root:root` | Redacted display data for Homepage |
| `systemd/disk-health.service`, `disk-health.timer` | `644 root:root` | Host unit source files |

**Baseline policy:** `collect-v0.2.py --init-baseline` was explicitly run once after a validated six-drive preview. Never schedule it or re-run it to suppress alerts. A disk replacement or intentionally accepted new reference requires separate manual review, identity validation and history handling. Collectors v0.6 and v0.7 read (do not change) the baseline.

**Sample vs publication times:** `generated_at` records the completed collector cycle in UTC and `last_updated` formats that time for `Europe/Brussels`. Each disk retains its own `last_smart_at`. A sleeping or unreadable disk must not be represented as a fresh temperature/SMART reading; the card can use `last_temperature_c` with an explicit Sleeping/Unknown label. Under the approved native-widget design, if the entire timer stops, the last published Healthy tiles may remain visible: users must inspect the displayed **LAST UPDATED** time. There is no independent automatic stale-state evaluator.

**Alert policy (v0.6 unchanged in v0.7):** compare tracked SMART counters against the fixed baseline, flag increases/decreases and nonzero 197/198, classify SMART overall-health failure as Critical, preserve unresolved Warning/Critical across future reads including standby/Unknown. Alert acknowledgements are **not implemented**; do not edit the private JSON to hide a problem. Temperature and SSD wear **threshold alerts are not yet implemented**. `SMART PASSED` is not a substitute for counter trend checks.

**File consistency:** both state and status writes use temporary files, `fsync`, permissions and atomic `os.replace`. The private state is written first and public status last; **the pair is not a single cross-file transaction**. On a failed status publish, the prior public status remains complete with its older timestamp; the private state may be newer. Investigate journal errors and re-run after correcting the cause.

## Historical Homepage integration (v0.7, superseded by unified v0.8)

Homepage Compose (`/srv/docker/homepage/compose.yaml`) has **one extra read-only volume on the existing `homepage` service**:

```yaml
volumes:
  - ./config:/app/config
  - ./secrets:/app/secrets:ro
  - /srv/docker/disk-health/data:/app/public/disk-health:ro
```

Keep the other Compose keys, container, networks, Docker Socket Proxy, secrets and Glances/PeaNUT configurations unchanged.

At `/srv/docker/homepage/config/settings.yaml`, the two relevant groups are:

```yaml
layout:
  Storage:
    style: row
    columns: 2
  Infrastructure:
    style: row
    columns: 5
```

`services.yaml` now has two service cards under `- Storage:`: **Disk Space** (two Custom API widgets reading v0.7 `space` by stable ID, replacing Phase-1 positional Glances mappings) and **Disk Health** (three native Custom API widgets reading existing `disks` and `last_updated`). `- Infrastructure:` starts with UPS, followed by Portainer, Nginx Proxy Manager, AdGuard Home and NetAlertX; all other groups keep their previous layout. Disk Health is a **3 x 2 block**, plus the timestamp tile:

```yaml
    - Disk Health:
        icon: mdi-harddisk
        href: https://home.pirocorp.com/disk-health/status.json
        description: SMART health · 6 drives
        widgets:
          - type: customapi
            url: http://127.0.0.1:3000/disk-health/status.json
            display: block
            refreshInterval: 30000
            mappings:
              - field: "disks.0.label"
                label: SYSTEM SSD
              - field: "disks.1.label"
                label: IAC
              - field: "disks.2.label"
                label: LP
          - type: customapi
            url: http://127.0.0.1:3000/disk-health/status.json
            display: block
            refreshInterval: 30000
            mappings:
              - field: "disks.3.label"
                label: DATA
              - field: "disks.4.label"
                label: IA
              - field: "disks.5.label"
                label: COMP
          - type: customapi
            url: http://127.0.0.1:3000/disk-health/status.json
            display: block
            refreshInterval: 30000
            mappings:
              - field: last_updated
                label: LAST UPDATED
                format: text
```

The **Disk Health** `disks.N.label` mappings still depend on the collector's deliberately configured row order `system, iac, lp, data, ia, comp`. **Disk Space no longer depends on the Glances FS-array order**; it resolves fixed keys `space.system`, `space.lp`, `space.data`, `space.ia`, `space.comp`, `space.iac` from the same published JSON.

**Static file discovery caveat (observed):** existing `poc.json` responded to atomic replacements without restart, but newly created `status.json` initially returned HTTP 404 despite being present in the bind mount. A **Homepage-only container restart** made it return HTTP 200. After that, an ordinary v0.6 manual run updated LAST UPDATED on the open Homepage screen without a container restart or browser reload. Do not schedule Homepage restarts on every SMART cycle.

**Publication/privacy caveat:** everything under `/app/public/disk-health/` is served via the Homepage hostname to anyone able to reach that route, independently of the private state files. Keep it free of device paths, serials, hashes, secrets, raw smartctl JSON and sensitive alerts. The feature is intended for the already established trusted LAN/VPN route; the read-only mount does *not* add authentication to static files.

## Historical systemd v0.6/v0.7 unit and drop-in (v0.8 adds a later override)

Unit sources are kept at `/srv/docker/disk-health/systemd/` and linked into `/etc/systemd/system/`. The **original v0.6 unit source remains unchanged**; the current v0.7 command is selected through a host-local drop-in override. The original service was validated with `systemd-analyze verify` and manually executed successfully:

```ini
# disk-health.service
[Unit]
Description=PiroCorp SMART Disk Health Collector
After=local-fs.target

[Service]
Type=oneshot
User=root
Group=root
WorkingDirectory=/srv/docker/disk-health
ExecStart=/usr/bin/python3 -B /srv/docker/disk-health/collector/collect-v0.6.py --write
TimeoutStartSec=240
UMask=0077
StandardOutput=journal
StandardError=journal
```

```ini
# disk-health.timer
[Unit]
Description=PiroCorp SMART Disk Health Timer

[Timer]
OnCalendar=*:0/10
AccuracySec=1min
Persistent=true
Unit=disk-health.service

[Install]
WantedBy=timers.target
```

The active deployment has this additional systemd drop-in **`/etc/systemd/system/disk-health.service.d/10-collector-v07.conf`**:

```ini
[Service]
ExecStart=
ExecStart=/usr/bin/python3 -B /srv/docker/disk-health/collector/collect-v0.7.py --write
```

The blank `ExecStart=` clears the v0.6 line in the original unit; the next line selects v0.7. This was confirmed with `sudo systemctl cat disk-health.service` after `daemon-reload`. No change to `disk-health.timer` was made; the 10-minute schedule still applies. Registered originally with `systemctl link`, `systemctl daemon-reload`, and activated with `systemctl enable --now disk-health.timer`. **Only the timer is enabled**, not the oneshot service. `Persistent=true` allows a missed timer event to be triggered once after the timer becomes active again; systemd calendar times are interpreted in the host's local time zone.

## Day-to-day validation

```bash
# Timer enabled/active and most recent/next schedule:
systemctl is-enabled disk-health.timer
systemctl is-active disk-health.timer
systemctl list-timers --all disk-health.timer --no-pager

# Last result of the oneshot:
systemctl show disk-health.service --property=Result,ExecMainStatus,ActiveState --no-pager
sudo journalctl -u disk-health.service -n 35 --no-pager

# Compare last modified time with homepage:
sudo stat -c '%y %a %U:%G %n' /srv/docker/disk-health/data/status.json
sudo python3 -m json.tool /srv/docker/disk-health/data/status.json >/dev/null
sudo docker exec homepage wget -S -O /dev/null http://127.0.0.1:3000/disk-health/status.json
```

Expected oneshot result after completion: `Result=success`, `ExecMainStatus=0`, `ActiveState=inactive`; journal contains six SMART disk lines, six `space/...` lines and `STATE + STATUS WRITTEN. Baseline unchanged.`. Homepage's LAST UPDATED advances after a successful collector run, subject to its 30-second widget refresh. The first **automatic** journal invocation is verified (2026-10-15, 21:40 UTC). Keep checking subsequent runs and LAST UPDATED for failures.

A deliberate one-shot check can be launched with `sudo systemctl start disk-health.service`. Preview without file writes uses `sudo python3 -B /srv/docker/disk-health/collector/collect-v0.7.py --preview`; **preview still reads the drives**.

## Historical stable Disk Space integration (v0.7, deployed 2026-10-16 CEST)

Previously Phase-1 Homepage used Glances `/api/4/fs` with `0.percent` through `5.percent`. This was **not a stable mapping**: Glances returns an array whose position can change after restart or mount changes, causing a percentage to appear under the wrong disk label. Read-only checks matched all six Glances mount-point records exactly once, and host `os.statvfs()` matched their capacity percentages for all six disks (`11.1`, `51.9`, `46.2`, `80.0`, `33.9`, `51.4` at acceptance).

v0.7 measures filesystem space **on the Ubuntu host**, independently of SMART collection and without querying Glances. It uses the fixed mount mapping: `system:/`, `lp:/mnt/lp`, `data:/mnt/data`, `ia:/mnt/ia`, `comp:/mnt/comp`, `iac:/mnt/iac`. For each mount it checks `os.path.ismount()`, then calculates used percentage using `os.statvfs()` with user-available blocks (matching Glances/psutil semantics). If a mount is missing or invalid, it publishes `state: Unavailable`, `percent: null`, `label: Unavailable` **instead of silently measuring the parent filesystem or inventing 0%**. A filesystem failure must not suppress SMART output or reset the SMART baseline.

The **additive** `space` object in public `status.json` leaves the existing `schema_version: 1`, `disks`, `generated_at`, `last_updated`, private SMART state, sticky alerts and atomic publication order unchanged. Illustrative shape (numbers vary):

```json
{
  "space": {
    "system": {"state": "Available", "percent": 11.1, "label": "11.1%"},
    "lp": {"state": "Available", "percent": 51.9, "label": "51.9%"},
    "data": {"state": "Available", "percent": 46.2, "label": "46.2%"},
    "ia": {"state": "Available", "percent": 80.0, "label": "80.0%"},
    "comp": {"state": "Available", "percent": 33.9, "label": "33.9%"},
    "iac": {"state": "Available", "percent": 51.4, "label": "51.4%"}
  }
}
```

The deployed **Disk Space** section in `/srv/docker/homepage/config/services.yaml` keeps two `customapi` widgets of three blocks each; both use `url: http://127.0.0.1:3000/disk-health/status.json`, `refreshInterval: 30000`, and **no Glances credentials**:

```yaml
- Storage:
    - Disk Space:
        icon: mdi-database-outline
        href: https://glances.pirocorp.com
        description: Disk space · 6 filesystems
        widgets:
          - type: customapi
            url: http://127.0.0.1:3000/disk-health/status.json
            display: block
            refreshInterval: 30000
            mappings:
              - { field: space.system.label, label: SYSTEM SSD, format: text }
              - { field: space.lp.label, label: LP, format: text }
              - { field: space.data.label, label: DATA, format: text }
          - type: customapi
            url: http://127.0.0.1:3000/disk-health/status.json
            display: block
            refreshInterval: 30000
            mappings:
              - { field: space.ia.label, label: IA, format: text }
              - { field: space.comp.label, label: COMP, format: text }
              - { field: space.iac.label, label: IAC, format: text }
```

`format: text` is intentional: labels include `%` or `Unavailable`; a numeric percent formatter could misrepresent failures. Homepage re-reads every 30 seconds, but **new measurements are generated every 10 minutes** by the systemd timer. The screen at 2026-10-16 00:31 CEST confirmed the six correct percentages and unchanged healthy SMART tiles. A subsequent Homepage screenshot at **00:40 CEST** showed an advanced LAST UPDATED timestamp, independently corroborated by the **22:40:06–22:40:07 UTC systemd journal** recording the first successful automatically fired v0.7 cycle.

**Acceptance evidence:** v0.7 `--preview`: six healthy SMART and six available space readings; isolated staging `--write` with cloned private history: `STAGING WRITE TEST PASSED`, original status/state untouched, unknown mount yields Unavailable; manual `systemctl start disk-health.service`: journal 22:31:24–22:31:25 UTC shows all twelve samples and successful publication; `docker exec homepage node ... fetch('http://127.0.0.1:3000/disk-health/status.json')`: HTTP 200, six SMART records, six keyed space objects; screenshot displays both Storage cards normally. The SMART baseline was **not reinitialized**. **Automatic scheduling evidence:** `journalctl -u disk-health.service` at 22:40:06–22:40:07 UTC reports six Healthy SMART readings, six Available filesystem readings, `STATE + STATUS WRITTEN. Baseline unchanged.`, and successful service completion; Homepage's LAST UPDATED also advanced to 16.10.2026 00:40 CEST.

**Safe v0.7 rollback sequence:** first restore the backed-up **Phase-1 Disk Space** widget configuration in Homepage (two Glances `customapi` blocks using array indexes), and verify the display. Then remove **only** `/etc/systemd/system/disk-health.service.d/10-collector-v07.conf` and run `sudo systemctl daemon-reload` to fall back to the untouched v0.6 service; `sudo systemctl cat disk-health.service` must show only the original v0.6 `ExecStart`. Do not delete other drop-ins, do not change the timer, baseline or history; note that v0.6 publications omit `space` and would otherwise break the migrated Disk Space widget. Restore from the timestamped Homepage `services.yaml.bak-space-*` backup only after verifying its contents and current changes, to avoid discarding later configuration edits.

## Troubleshooting, rollback and guardrails

| Symptom | Check / safe response |
| --- | --- |
| File exists in container but URL is 404 | Inspect `sudo docker exec homepage ls -l /app/public/disk-health/`; restart only Homepage once for a newly added filename, then retest HTTP |
| LAST UPDATED does not advance | Compare `list-timers`, `journalctl`, `stat`, and the JSON timestamp; verify the widget endpoint |
| Sleeping / Unknown | Inspect latest journal and last observation; do not force HDD spin-up merely to refresh the card |
| New counter warning | Compare against private baseline and prior observations; investigate the drive/cabling; never reset baseline automatically |
| Systemd validation emits unrelated XFS CPUAccounting warnings | Check service's own errors and the verify **exit code**; the observed warnings came from pre-existing XFS units |

To **pause monitoring without deleting data**, run `sudo systemctl disable --now disk-health.timer`; this stops future scheduling but does not alter Homepage, SMART baseline or UPS/NUT. To roll back only the new display, restore the timestamped backup of Homepage `services.yaml` and `settings.yaml` (or remove the two Storage changes), and allow Homepage to reload; roll back Compose's read-only mount only if the static endpoint is no longer required. Preserve `state/` during rollback for investigation. Verify restored YAML and other service cards afterward.

Do not use `--init-baseline` as a maintenance command, commit local JSON data, grant Homepage raw block-device permissions, or change SATA/NTFS/Samba/NUT settings to make the dashboard appear healthy.

## Further work (not yet delivered)

1. Verify timer persistence and successful scheduled SMART + Disk Space checks **after a later host reboot**; first automatic v0.6 run at 21:40 UTC and first automatic v0.7 run at 22:40 UTC on 2026-10-15 are both confirmed.
2. Validate the standby protection with an HDD actually reporting STANDBY.
3. Agree per-drive temperature warning/critical bands and SSD wear semantics; implement and test threshold alerts.
4. Implement an explicit reviewed alert acknowledgement/reference-reset workflow.
5. Verify narrow-screen/iPhone appearance of the now-unified v0.8 Storage card; desktop acceptance is complete.
6. Establish a **private backup** for the host-only collector code, disk identity config and state; these are not recoverable from this documentation PR alone.

## References

- [Homepage v2.4.0 Custom API widget](https://github.com/gethomepage/homepage/blob/v2.4.0/docs/widgets/services/customapi.md)
- [Homepage v2.4.0 groups/layout](https://github.com/gethomepage/homepage/blob/v2.4.0/docs/configs/settings.md)
- [smartmontools](https://www.smartmontools.org/)
