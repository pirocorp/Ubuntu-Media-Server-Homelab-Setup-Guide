# OmniMeter Local Energy Monitoring

Status: Planned — candidate evaluation only; deployment not approved
Purpose: Preserve the proposed lightweight self-hosted electricity and gas history project, using HomeWizard P1 as the local source and OmniMeter as the application candidate.
Depends on: Existing Ubuntu Docker host, active Fluvius P1 user port, operational HomeWizard P1 Meter, working LAN access, and operational HomeWizard local API v2 (validated 2026-10-16). OmniMeter compatibility and deployment remain unapproved.
Related docs: [Roadmaps index](../README.md), [Services index](../../services/README.md), [Homepage runbook](../../services/homepage/README.md), [Docker and Portainer](../../platform/docker-and-portainer.md), [Networking and reverse proxy](../../platform/networking-and-reverse-proxy.md)

## Decision Record and AI Implementation Brief

**Status:** Proposed concept; not selected for deployment yet  
**Version:** 0.1  
**Date:** 2026-10-14

The existing discussion is an architecture evaluation, **not** authorization to install or modify services. OmniMeter is a promising candidate rather than an irrevocable product choice. Treat the user requirements and scope boundaries below as established; treat all unverified product capabilities and deployment details as validation gates.

## 1. Objective

Keep a local, user-owned, long-term record of household electricity and gas consumption from the existing HomeWizard P1 Meter, with a simple dedicated browser UI. Avoid operating a generic observability stack only for household energy.

Minimum desired experience:

- current imported/exported electrical power (as available);
- electricity consumption history by hour, day, month, and year;
- differentiated day/night electricity counters and tariffs;
- gas consumption history (m³), alongside electricity;
- usable historical graphs and consumption comparisons;
- estimated spending in EUR, with honest separation between commodity rates and regulated/fixed costs;
- persistent data, controlled backup/restore, and no dependence on the HomeWizard cloud for locally recorded history.

Useful features to **evaluate**, not assume:

- Belgian 15-minute average power/peak and capacity-tariff analysis;
- importing historical HomeWizard CSV exports;
- reliable detection/visibility of gaps, device outages, or missed readings.

The application must be simple enough to justify its upkeep.

## 2. Verified local environment and facts

- Ubuntu Docker/Compose homelab host: `192.168.0.10`.
- Existing convention: independent stacks under `/srv/docker/<service>`.
- HomeWizard P1 Meter: `192.168.0.222` on the LAN at time of discussion (a DHCP reservation/static allocation is to be checked before implementation).
- The Fluvius user P1 port has been enabled, and the P1 device now receives power directly from the meter.
- HomeWizard Energy already shows live electrical power and receives electricity/gas readings.
- HomeWizard **Local API v1 is enabled**; `http://192.168.0.222/api/v1/data` was tested successfully. **API v2 subsequently validated on 2026-10-16**, including physical pairing and an authenticated `GET /api/measurement` returning HTTP 200; see validation record below.
- HomeWizard records cumulative electricity and gas data. The effective rate start date for the current supplier contract is not yet confirmed; `2026-01-01` has been used as a **temporary placeholder in the HomeWizard app**, not as a verified contract effective date.
- Existing local publishing model: explicit AdGuard Home rewrites (no wildcard DNS rewrite) -> Nginx Proxy Manager (NPM) with existing wildcard Let's Encrypt certificate, for LAN/trusted Tailscale clients.
- Homepage at `https://home.pirocorp.com` is the central homelab landing page.

Do not put meter identifiers, EANs, supplier account identifiers, credentials, or personal billing records into this public repository.

### 2026-10-16 — HomeWizard local API v2 validation (completed)

This is **device-side validation only**. It does not constitute an OmniMeter deployment or prove end-to-end collector/dashboard behavior.

- Unauthenticated `GET https://192.168.0.222/api` with `X-Api-Version: 2` returned HTTP **401** and `user:unauthorized`: API v2 is reachable over HTTPS and requires authentication.
- Before physical confirmation, `POST /api/user` with `name=local/omnimeter-validation` returned HTTP **403** and `user:creation-not-enabled`: pairing security works.
- After pressing the device button, a one-shot `POST /api/user` returned HTTP **200**. The issued Bearer token was saved on the Ubuntu host at `~/.config/homewizard/omnimeter-validation.json` with restrictive filesystem permissions. **Never commit or print the token.**
- Using that token, authenticated `GET /api/measurement` returned HTTP **200**. Observed available fields: `timestamp`, `protocol_version` (50 / DSMR 5.0), `tariff`, `power_w`, `energy_import_kwh`, `energy_import_t1_kwh`, `energy_import_t2_kwh`, `energy_export_kwh`, `average_power_15m_w`, `monthly_power_peak_w`, `monthly_power_peak_timestamp`, and `external[]` with `type=gas_meter`, cumulative `value`, `unit=m3`, and gas timestamp.
- Tariff-register sum was internally consistent with the combined import total. The gas measurement timestamp can differ from the electricity telegram timestamp.
- No household cumulative readings, serials, meter identifiers, EANs, bills, or secrets are included in this public record.
- Diagnostic requests used `curl -k` / an unverified Python SSL context; **certificate chain and hostname verification remain an explicit deployment gate** (using HomeWizard's CA and device identity).
- The meter directly reports 15-minute average demand and the current monthly peak, so the *hardware/API* supports relevant Belgian capacity-tariff inputs. The actual billed capacity charge depends on monthly/rolling-average rules and network tariff; an instantaneous monthly peak is not itself the invoice total.

**Upstream OmniMeter source review, 2026-10-16:** `src/meter_ingest.py` currently maps the P1's import/export cumulative counters, phase power values and `external[].gas_meter.value` into SQLite readings, but the P1 field map / persistence targets **do not include** `power_w`, `average_power_15m_w`, `monthly_power_peak_w` or `monthly_power_peak_timestamp`. Consequently do **not** claim native preservation or graphing of those fields. Decide whether native peak tracking is essential and assess upstream capability or alternatives before approving deployment. Upstream links: [meter ingestion](https://github.com/t-philip/omnimeter/blob/main/src/meter_ingest.py), [collector loop](https://github.com/t-philip/omnimeter/blob/main/src/meter_poller.py).

**Remaining acceptance gates:** actual OmniMeter collection, field mapping, dashboard quality, Belgian cost model, validated TLS/hostname, retention/gaps, hosting/access controls, backup/restore. No software deployment approved.

## 3. Candidate project: OmniMeter

Upstream: https://github.com/t-philip/omnimeter

As reviewed on 2026-10-14, the upstream README documents:

- Docker Compose and a dedicated web dashboard;
- local HomeWizard API v2 integration via HTTPS with device pairing/Bearer token;
- electricity, gas, and other optional measurement categories;
- SQLite persistent storage;
- a long-running collector polling roughly every 20 seconds (not WebSocket streaming);
- CSV import, consumption/cost views, and an example backup service with nightly backups;
- an English-only interface and no built-in multi-user accounts;
- opt-in optional network features that should remain disabled by default;
- AGPL-3.0 licensing.

**Upstream risks:** small/new project with a narrow maintainer base; API compatibility, reliability, updates, migrations, backfill, Belgium-specific tariff support, and restore procedures require a real evaluation. Do not equate advertised capabilities with validated capabilities.

## 4. Proposed architecture (subject to acceptance)

```text
HomeWizard P1 Meter (192.168.0.222)
    -> local HTTPS API v2 + per-device authentication (verify availability)
    -> OmniMeter poller / application
       -> persistent SQLite history
       -> local backups stored independently of the active database
       -> built-in energy web dashboard
    -> internal Docker/host port (to be verified)
    -> Nginx Proxy Manager HTTPS
    -> energy.pirocorp.com (proposed, NOT reserved/approved yet)
       -> explicit AdGuard DNS rewrite for LAN users
       -> trusted Tailscale access using existing model
```

- Plan a dedicated `/srv/docker/omnimeter` stack only **after** approval.
- Prefer OmniMeter's native collector/database/UI; do not introduce Telegraf, InfluxDB, Grafana, Home Assistant, or a custom .NET service without a proven need.
- Persist SQLite files, configuration, pairing credentials, and backups on the host, with distinct backup location and tested restoration.
- Pin a reviewed upstream release/commit/image rather than blindly deploying an unpinned moving branch.
- Keep the database and management interfaces off the public Internet. Do not add router forwarding or Cloudflare Tunnel for this service by default.
- NPM proxy and hostname are proposals only, not live configuration.
- If API v2 is unavailable on this exact meter/firmware, **stop and revisit the choice**; do not silently downgrade to v1 or install another collector.

## 5. Scope boundaries

### In scope

- Evaluate OmniMeter's actual behavior with the existing HomeWizard P1 hardware.
- Confirm API v2 availability, activation/pairing, token persistence and TLS behavior.
- Deploy one dedicated local energy dashboard only after decision/approval.
- Record and visualize electricity and gas history in a user-controlled persistent store.
- Configure meaningful tariffs in EUR, with explicit effective dates and documented assumptions.
- Validate backup/restore and upgrades against the collected data.
- Add a compact link/status card in existing Homepage if useful, without replacing Homepage.

### Out of scope

- Installing Home Assistant, Grafana, InfluxDB, or a broad observability stack.
- Replacing the HomeWizard app, altering the electrical meter, or changing Fluvius port settings.
- Reconfiguring Philips Hue Bridge, smart buttons, appliances, or home automation.
- Replacing Netdata or redesigning UPS monitoring (a separate homelab dashboard/monitoring discussion).
- Building a bespoke energy app or ingestion bridge before proving the need.
- Exposing the energy application on the open Internet.
- Assuming the project supports every Belgian fee, tax or capacity-tariff calculation without validation.

## 6. Meter measurements, tariffs and historical accuracy

### Data acquisition

- Verify whether the actual meter exposes API v2 in HomeWizard Energy, its required firmware/version, and whether pairing works on the local network.
- Confirm which fields are available: instantaneous W, import/export totals, tariff 1/2 kWh, gas m³, voltage, 15-minute average and monthly peak if reported.
- Measure polling interval and gaps in practice. An approximately 20-second poll gives sampled history, **not** every second-level change.
- Energy totals must be computed from cumulative readings without double-counting after a restart or import.
- Newly self-hosted history begins when ingestion starts unless a historical CSV import succeeds. Device API availability does not imply that past readings can be recovered automatically.

### Tariff accounting

- The supplier contract is a day/night fixed-rate electricity plan with a separate gas rate.
- Effective start date is **pending confirmation**. The existing `2026-01-01` setting is only a temporary app placeholder and must not be silently treated as confirmed.
- Separate supplier energy prices from VAT, grid/transport fees, fixed charges and the Belgian capacity tariff.
- Gas meter readings arrive in m³ whereas the supplier may price energy in kWh; conversion factors can vary. Avoid hardcoding the earlier invoice's conversion factor as universally constant.
- Establish whether OmniMeter can handle rate schedules, two electricity tariffs and gas cost units for this exact contract.
- If any costs cannot be computed accurately, label output **estimated / energy component only** rather than presenting it as a full invoice.

### Belgian peak tracking

- Check whether OmniMeter stores/displays the P1's actual 15-minute average and monthly peak, if reported by the meter.
- Do not assume 20-second sampled instantaneous power is sufficient to reconstruct the regulated 15-minute settlement peak exactly.
- Any capacity-tariff visualization must show its methodology and limitations, or be explicitly deferred.

## 7. Security, availability and backup

- Use a LAN-restricted HomeWizard API endpoint, credentials in untracked host files/env, and verified TLS behavior for v2.
- OmniMeter's own access model must be assessed; NPM TLS alone is not equivalent to user authentication.
- Restrict the web UI to LAN/trusted Tailscale paths and apply suitable access protection if the app does not provide user accounts.
- Disable optional outbound weather, update-check and notification integrations unless explicitly requested.
- Ensure backups are separate from the active SQLite database, scheduled and retained according to an agreed policy.
- Perform at least one restore drill, and verify upgrades preserve DB, configuration, tokens and historic samples.
- Meter or server disconnections should not corrupt data; missing collection windows must be visible rather than silently interpolated.

## 8. Open decisions / validation gates

1. **Product decision:** does OmniMeter meet the desired quality/feature bar versus other focused applications?
2. **API v2 — verified 2026-10-16:** this P1 Meter exposes v2, accepts physical pairing, and returns authenticated electricity/gas measurements (HTTP 200). Certificate verification and OmniMeter integration remain open.
3. **Belgian pricing:** day/night electricity, gas units, VAT/network charges and effective-dated rates.
4. **Capacity tariff — meter confirmed / app gap:** P1 reports actual 15-minute average and monthly peak, but currently reviewed OmniMeter ingestion does not persist these fields. Are native graphs/retention required, and can the app be extended without excessive complexity?
5. **History:** sample interval, retention, gap handling, CSV backfill and cost of database growth.
6. **Operations:** exact version/pinning strategy, Docker service composition, volumes and rollback/restore.
7. **Access:** proposed `energy.pirocorp.com` hostname, local port, NPM configuration and authentication model.
8. **Backups:** storage destination, retention, and periodic verification.

Unresolved decisions are **not** approvals to implement additional components.

## 9. Proposed implementation order (later, upon approval)

1. ~~Verify HomeWizard API v2 availability, pairing, and authenticated measurements~~ — completed 2026-10-16. Before deployment, verify device firmware/identity and TLS certificate chain + hostname without changing the working v1 setup.
2. Review a specific upstream OmniMeter release and its documentation/dependencies/security model.
3. Prove electricity and gas capture, day/night counters, gaps and timestamps in an isolated trial.
4. Verify cost/tariff support and 15-minute peak requirements; choose whether limitations are acceptable.
5. Finalize network exposure, hostname, service port, persistence, backup policy and version.
6. Only then deploy the stack under `/srv/docker/omnimeter` using the established Docker/NPM/AdGuard pattern.
7. Verify readings, persistence after restart, backup/restore, access controls and non-dependence on the vendor cloud for stored history.
8. Optionally add the app to Homepage and write the deployed-service runbook.

## 10. Acceptance checklist

- [ ] User explicitly chooses OmniMeter after validation.
- [x] API v2 confirmed and authenticated with this exact P1 Meter (2026-10-16); TLS validation and application integration are still pending.
- [ ] Real-time electricity and gas ingestion verified against HomeWizard readings.
- [ ] Tariff 1/2 and gas consumption recorded without unit mistakes.
- [ ] Historical data survives container restart/recreation.
- [ ] Missing data periods and sampling cadence are understood.
- [ ] Cost calculation displays its limitations and uses effective-dated contract settings.
- [ ] Belgian 15-minute peak capability assessed and documented (supported or explicitly deferred).
- [ ] No unnecessary monitoring/database stack was installed.
- [ ] Web UI restricted to trusted access, with secrets kept out of Git.
- [ ] Backup and restore of real accumulated history tested.
- [ ] Simple dashboard remains usable from a phone and desktop.
- [ ] Existing Homepage, Hue, UPS/NUT and other homelab functions remain unaffected.

## 11. Documentation transition after deployment

Only when deployed and validated:

1. Add `docs/services/omnimeter/README.md` with real pinned version, stack path, volumes, ports, credentials-handling strategy, operational checks and restore steps.
2. Add OmniMeter to implemented service inventory, current-state docs and service index.
3. Update access URL list with the **actual** confirmed hostname.
4. Move the roadmap from Planned to Implemented while retaining this record.
5. Keep any future replacement for Netdata / server-and-UPS dashboard in a **separate** project.

## 12. Official references

- OmniMeter source/docs: https://github.com/t-philip/omnimeter
- HomeWizard Local API docs: https://api-documentation.homewizard.com/

**Guardrail:** This roadmap stores a candidate and requirements so the idea is not forgotten. It does not install OmniMeter or authorize broader architecture changes.
