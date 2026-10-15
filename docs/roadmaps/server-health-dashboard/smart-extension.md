# Homepage Disk Health — Phase 2 SMART Extension

Status: **Implemented core monitoring on 2026-10-15**; this file preserves the original 2026-10-14 pre-deployment plan. See the [Phase-2 as-built runbook](../../services/disk-health/README.md) for the deployed system and remaining acceptance gates.
Purpose: Add a compact, readable per-drive SMART Health overview to the existing Homepage without installing a standalone disk-monitoring platform.
Depends on: [Completed Phase 1 Server Health](../../services/server-health/README.md), existing Homepage v2.4.0, host `smartctl`, existing physical drives and mount points
Related docs: [Roadmaps index](../README.md), [Homepage runbook](../../services/homepage/README.md), [Storage and Samba](../../platform/storage-and-samba.md)

## Historical Decision Record and Original Implementation Brief

**Status:** Planned extension, implementation not started
**Version:** 0.1
**Date:** 2026-10-14

This section is the **original proposal written before implementation**, not a description of the current runtime. Phase 1 (CPU, RAM, file-space and UPS monitoring through Glances/PeaNUT) was completed and accepted on 2026-10-14; [see its runbook](../../services/server-health/README.md). Phase 2 was deployed separately on 2026-10-15; this documentation PR records the *as-built* result and does not itself deploy anything. The original suggestions (thin HTTP server, possible iframe, four-column design, stale handling) were superseded by the deployed native Homepage approach, documented in the [Phase-2 runbook](../../services/disk-health/README.md).

## 1. User-approved goal

A single, legible **Disk Health** card within the **existing** Homepage, for all **six physical drives**, with:

- **Disk identification** using useful labels (System SSD, iac, lp, data, ia, comp), not just anonymous percentages.
- **Used space**, percent (with free capacity optionally shown).
- **Drive temperature**, °C when available.
- **SMART status**: `Healthy` / `Warning` / `Critical` / `Unknown`.
- **Health/wear percentage only if a meaningful manufacturer/device-provided indicator is available**; no invented health percentages for hard disks.
- Correct treatment of disks sleeping, SMART scan failure, unavailable metrics and stale data.

No requirement for long-term SMART time-series database, elaborate charts, a standalone monitoring dashboard or public exposure.

The previously discussed four-column **Disk / Used / Temp / Health** layout is the desired mobile/desktop presentation. The confirmed standard Homepage v2.4.0 `customapi` `dynamic-list` is only **two columns**, not a pixel-identical substitute. A UI PoC will determine whether its compact two-column summary is acceptable; if not, evaluate a **small HTML iframe card** before considering larger changes. The final presentation must be approved based on **actual iPhone screenshots**.

## 2. Verified host environment and reference measurements

From the user's `lsblk` / `df` / `smartctl` outputs (October 2026):

| Physical device (current name) | Model / media | Host filesystem | Used snapshot | Temperature snapshot | SMART |
| --- | --- | --- | --- | --- | --- |
| `/dev/sda` | Seagate ST4000VX007 4 TB HDD | `/mnt/iac` (NTFS) | 52% | 33°C | PASSED |
| `/dev/sdb` | Seagate ST8000DM004 8 TB HDD | `/mnt/lp` (NTFS) | 52% | 37°C | PASSED |
| `/dev/sdc` | Seagate ST8000DM004 8 TB HDD | `/mnt/data` (NTFS) | 47% | 37°C | PASSED |
| `/dev/sdd` | Seagate ST4000VX000 4 TB HDD | `/mnt/ia` (NTFS) | 80% | 33°C | PASSED |
| `/dev/sde` | Samsung SSD 870 2 TB SATA SSD | `/` (ext4) | 11% | 28°C | PASSED |
| `/dev/sdf` | Seagate ST4000DM004 4 TB HDD | `/mnt/comp` (NTFS) | 34% | 36°C | PASSED |

- The host `smartctl --scan` sees all six drives.
- All six returned `SMART overall-health self-assessment test result: PASSED` at the observation time.
- On the five Seagate HDDs, inspected attributes 5 (`Reallocated_Sector_Ct`), 197 (`Current_Pending_Sector`), and 198 (`Offline_Uncorrectable`) were each zero.
- The Samsung SSD returned normalized SMART attribute 177 (`Wear_Leveling_Count`) value **96**. **This is not a confirmed "96% life remaining" figure**; verify the device-specific meaning and display as "wear index 96" if that is all that can be established.
- Disk temperatures came from SMART attributes; no temperatures or health values should be hardcoded.
- These `/dev/sdX` names are **current observations, not stable identities**. Resolve actual device identities / filesystem mapping during implementation (e.g. by-id + `lsblk`), and do not commit hardware serial numbers to the public repo.
- Five data partitions are NTFS (`fuseblk`) and exported over Samba to Windows clients. Their configuration is **out of scope**.

## 3. Proposed minimal architecture

```text
Ubuntu host (scheduled, read-only collector)
  |-> smartctl --json for physical disks, respecting standby
  |-> filesystem free/used from mount points
  |-> normalize results into atomic status.json (timestamp + six disk items)
  |
  v
Local readonly JSON endpoint (thin static server, only if needed)
  |
  v
Homepage v2.4.0 Custom API widget (internal Docker network)
  |
  v
Disk Health card within existing Homepage
```

Architecture choices:

1. Execute the SMART collector **on the Ubuntu host** where `smartctl` already works. Avoid granting any new Docker container direct block-device access, `privileged: true`, or a host Docker socket.
2. Keep the collector small: a scheduled process (systemd timer preferred over another always-running daemon); candidate cadence **5–10 minutes**, tune only after checking wake/sleep behavior.
3. Use `smartctl -j` structured output and explicit checks; do not parse localized free-text SMART output with a brittle `grep` pipeline. Test `smartctl -n standby` semantics for the five HDDs **before enabling scheduling**.
4. Read filesystem space through host `statvfs`/`df` for the approved mount points. Use read-only information; no scans/writes of directory contents.
5. Write JSON **atomically**, with `observed_at`, per-device `checked_at` and last error/mode; never present stale data as freshly healthy.
6. If standard Homepage `customapi` needs HTTP, expose only a **read-only, internal** endpoint (e.g. a minimal static-file server on a private Docker network). Avoid a new Internet/LAN endpoint or a dedicated dashboard/database. Verify feasibility before choosing the server implementation.
7. Do not mount a host device or NTFS share as writable in an application container, change Samba shares, or modify filesystem formats.
8. SMART collection and display are strictly **observational**. No disk commands, self-tests, firmware updates, shutdowns or filesystem modifications are part of the dashboard.

### Suggested JSON schema (proposal; subject to PoC)

```json
{
  "observed_at": "2026-10-14T16:00:00+02:00",
  "disks": [
    {
      "name": "HDD · ia",
      "mount": "/mnt/ia",
      "usage_percent": 80,
      "free_bytes": 803158884352,
      "temperature_c": 33,
      "health": "Healthy",
      "wear_index": null,
      "wear_unit": null,
      "updated_at": "2026-10-14T16:00:00+02:00",
      "state": "ok",
      "summary": "80% · 33°C · Healthy"
    }
  ]
}
```

Example values are **illustrative only**, not a real generated status file.

`summary` is a preformatted string for Homepage's default two-column `dynamic-list` fallback, not a complete UI API. For a richer view retain structured fields rather than encoding everything into text.

## 4. Status/health calculation

### HDDs (five Seagate)

- `Critical`: a failed SMART overall-health result or other clearly fatal indicator.
- `Warning`: material nonzero pending or uncorrectable sectors, newly reallocated sectors, significant changes from previously observed attributes, or a validated manufacturer-specific warning.
- `Healthy`: SMART passes and the required relevant checks have **actually succeeded**, with no warning thresholds met.
- `Unknown`: SMART unavailable, device missing, data too old, unsupported or unparseable; **never** convert "command failed" into "Healthy".
- Temperature outside the agreed safe band should be explicitly flagged, but threshold values must be checked against the actual models/environment rather than invented in this roadmap.
- Hard disks have no reliable standardized "% life remaining"; **do not synthesize a "100% Health" indicator from PASSED**.

### Samsung SATA SSD

- Same basic `Healthy/Warning/Critical/Unknown` rules; include SSD-relevant media errors and wear attributes where supported.
- Show a percent **only if its semantics and unit have been established for the specific firmware/model**; normalized SMART value 96 alone is not proof of 96% remaining life.
- If percent meaning remains unclear, show `Wear index 96` or omit the numeric badge; never label an unsupported value "Health 96%".

### Sleep, gaps, and stale data

- If `smartctl -n standby` intentionally skips a sleeping drive, avoid waking it merely to repaint Homepage. Display `Sleeping / last observed status at <time>` where practical, with a clearly visible freshness indicator.
- If a drive is not checked for longer than the agreed freshness window, surface `Stale`/`Unknown` rather than a misleading live green check.
- If an entire collection run fails, the endpoint must not silently keep an old status with a new timestamp.
- Distinguish a drive that is genuinely unhealthy from one whose data source is unavailable.

## 5. Homepage visual contract

Target conceptual table/card:

| Disk | Used | Temperature | Health |
| --- | --- | --- | --- |
| SSD · System | 11% | 28°C | Healthy · wear index 96* |
| HDD · iac | 52% | 33°C | Healthy |
| HDD · lp | 52% | 37°C | Healthy |
| HDD · data | 47% | 37°C | Healthy |
| HDD · ia | 80% | 33°C | Healthy |
| HDD · comp | 34% | 36°C | Healthy |

*Example from inspected SMART output, not a confirmed percent of SSD remaining life.

**Homepage v2.4.0 constraint:** `customapi` with `display: dynamic-list` only has left `name` and right `label` fields. It can show `80% · 33°C · Healthy` for each drive, but **cannot directly render the proposed four-column table**. Do not claim that a ready-made Homepage widget reproduces it.

- **First PoC:** native `Custom API dynamic-list` with a single six-row card, no custom frontend, available in v2.4.0.
- **Optional UI upgrade:** a small static HTML table embedded through Homepage `iframe`, *only if* mobile presentation is materially better. Test HTTPS/same-origin restrictions, internal exposure and authentication. Keep code/security complexity bounded.
- Do not modify Homepage source code or upgrade Homepage just to implement the disk card without separate approval.
- Keep the existing Homepage infrastructure/media cards and one-screen usability intact.

## 6. Scope / no-regression requirements

**In scope:** six-drive diagnostics and card only; read-only measurements; limited private JSON transport; visual warnings/unknown; documentation and rollback.

**Out of scope:** Phase-1 CPU/RAM/Glances/PeaNUT deployment; Netdata reinstallation; drive writes and filesystem changes; NTFS conversion; Samba changes; disk firmware operations; all-in-one monitoring databases; commercial or proprietary health-score algorithms.

This is a **phase-2** roadmap: deploying it before the baseline monitoring has been accepted would contradict the intended sequence.

## 7. Future implementation order

1. Confirm the deployed phase-1 Glances + PeaNUT baseline is still working and backed up (phase 1 accepted 2026-10-14).
2. Confirm `smartctl -j` fields for all six actual drives without triggering unintended spin-up.
3. Map labels and mount points to stable hardware identifiers without leaking serial numbers.
4. Build a small host-only read collector with explicit timeout/error handling and manufacturer-specific wear interpretation where needed.
5. Test its JSON output against the accepted schema, health-state rules and filesystem measurements.
6. Run it manually on representative states, including failed read and sleeping-drive conditions; choose interval/freshness limits.
7. Select minimal safe JSON transport for Homepage (no public API/extra dashboard).
8. Test compact six-row `dynamic-list` on desktop and iPhone. Implement an HTML/iframe view **only if needed and approved**.
9. Configure an on-host timer; verify no disk wake-up side effect, reasonable CPU/IO overhead and no disturbance to Samba/NTFS.
10. Record rollback, failure modes and actual setup in a service/operations runbook.

## 8. Acceptance checklist

- [ ] Phase-1 Homepage + Glances + PeaNUT monitoring remains operational throughout any future SMART changes (baseline accepted 2026-10-14).
- [ ] All six physical disks are correctly identified and labelled after restart.
- [ ] Each drive displays actual used percent, available free space, temperature when supplied, and a meaningful SMART status.
- [ ] SSD wear percent is displayed only if device-specific interpretation is supported; no invented HDD health percentages.
- [ ] `Healthy` requires fresh valid SMART readings; failed/missing/old readings become `Unknown`/`Stale`.
- [ ] Sleeping HDDs are not spun up solely for dashboard refresh.
- [ ] All SMART commands and file reads are observational, no filesystem writes/remounts, NTFS/Samba access unaffected.
- [ ] The JSON service, if introduced, has no router forward/public DNS/public exposure and minimal permissions.
- [ ] The card is legible on iPhone, and the final compact vs four-column view is explicitly approved.
- [ ] Homepage existing cards and Docker Socket Proxy remain unchanged.
- [ ] Rollback is documented and tested.
- [ ] No services or live config were changed by this documentation PR.

## 9. References

- [Phase-1 Server Health roadmap](./README.md)
- [Homepage v2.4.0 Custom API widget](https://github.com/gethomepage/homepage/blob/v2.4.0/docs/widgets/services/customapi.md)
- [Homepage v2.4.0 iframe widget](https://github.com/gethomepage/homepage/tree/v2.4.0/docs/widgets/services)
- [smartmontools / smartctl](https://www.smartmontools.org/)

**Historical guardrail:** The 2026-10-14 roadmap deferred implementation until after Phase 1 acceptance. That prerequisite was met; for the implementation actually performed on 2026-10-15, use the [as-built runbook](../../services/disk-health/README.md).