# Home Energy Monitoring — Home Assistant Container

Status: Planned — **Architecture GO approved 2026-10-16; PoC/deployment not authorized**
Purpose: Record the selected, ready-made self-hosted solution for local electricity and gas monitoring, Belgian cost estimates and capacity tariff on the existing Ubuntu/Docker homelab.
Depends on: Existing Docker/Compose host, working HomeWizard P1 Meter connected to the enabled Fluvius P1 port, local LAN connectivity, and the existing trusted-only reverse-proxy/DNS model.
Related docs: [Roadmaps index](../README.md), [Previous OmniMeter evaluation](../omnimeter/README.md), [Services index](../../services/README.md), [Homepage runbook](../../services/homepage/README.md), [Docker and Portainer](../../platform/docker-and-portainer.md), [Networking and reverse proxy](../../platform/networking-and-reverse-proxy.md).

## Locked Architecture Decision and AI Implementation Brief

**Decision:** Home Assistant Container is the preferred and approved architecture for the planned home energy project.  
**Phase:** Architecture Evaluation **CLOSED — GO**; PoC Planning is next, installation requires separate explicit approval.  
**Version:** 1.0  
**Decision date:** 2026-10-16

This document is a **plan**, not an as-built runbook or a deployment authorization. Do not install software, change the host, modify DNS/NPM, import data, or change existing devices simply because the architecture was approved.

## 1. Goal and success criterion

Deliver a practical, browser-based, privately hosted electricity and gas dashboard that shows current P1 measurements, consumption trends, realistic costs in EUR, and Flemish capacity-tariff estimates without building a bespoke application.

**Optimize for implementation simplicity and existing integrations**, not the smallest theoretical memory footprint or invoice-grade cent-level reconciliation. A useful working dashboard without historical imports is already a successful first release.

The new system supplements rather than replaces the HomeWizard phone app; it is separate from existing server/UPS/SMART monitoring and from all home automation work.

## 2. Locked architecture

```text
Landis+Gyr E360 smart meter / gas meter
  -> enabled Fluvius P1 port
  -> HomeWizard P1 Meter (local LAN, validated API v2 device-side)
  -> official Home Assistant HomeWizard integration
  -> Home Assistant Container (one Docker application service)
       -> built-in Recorder / persistent SQLite
       -> native Energy Dashboard + selected energy cards
       -> Belgian Electricity Prices custom integration
       -> Belgian Gas Prices custom integration
  -> existing AdGuard local DNS + Nginx Proxy Manager HTTPS
  -> LAN and trusted Tailscale browser clients only

Optional later, best effort:
Existing Fluvius exports -> reviewed one-off conversion/import -> HA statistics
```

### LOCKED decisions

1. Use **Home Assistant Container**, not Home Assistant OS or Supervised.
2. Use its default **SQLite** Recorder with persisted configuration/data; no additional database service.
3. Connect the working **HomeWizard P1 Meter** with the official HomeWizard integration. API v2 pairing/measurement were validated independently on 2026-10-16; verify the actual Home Assistant integration connection during the PoC rather than assuming an API transport.
4. Use [Belgian Electricity Prices](https://github.com/renaudallard/homeassistant_be_electricity_prices) and [Belgian Gas Prices](https://github.com/renaudallard/homeassistant_be_gas_prices) as the first ready-made cost integrations, subject to PoC compatibility and configuration validation.
5. Use Home Assistant's **Energy Dashboard** as the starting UI. Only add simple custom cards if necessary; no Grafana dashboards.
6. Include **current Flemish capacity tariff estimates** using the real P1 monthly-peak sensor where supported. Historical peak seeding is optional.
7. The electricity meter has **T1/T2 registers**, but the signed supply contract has **one price** for both. Preserve meaningful counters while billing both at the same rate; do not model an imaginary day/night price differential.
8. Estimated electricity/gas costs in **EUR** are sufficient. Include regulated/network/capacity/fixed components where the chosen integrations can calculate them; do not require exact replication of MEGA invoices or promotional discounts.
9. Attempt historical Fluvius import **best effort after the live system is working**, without letting it block deployment. Hourly aggregation of old 15-minute electricity history is acceptable. Importing historical monthly peaks is optional.
10. Do not introduce Grafana, InfluxDB, PostgreSQL, MariaDB, MQTT, Telegraf, a continuously running import bridge, or custom application development unless a **new architecture decision** explicitly approves a justified need.
11. Use the existing **LAN/trusted Tailscale + explicit AdGuard DNS + NPM HTTPS** publishing model; no new public router forwarding or Cloudflare Tunnel by default.
12. Keep the separate **Belgian Invoice Audit** commercial idea out of this project. Previously collected Fluvius exports and contract cards are reference inputs only, not a shared runtime component or cross-project development scope.

## 3. Confirmed hardware and source data

- Electricity meter: Landis+Gyr E360-1P.
- Working HomeWizard P1 Meter on the homelab LAN; Local API v2 was successfully paired and authenticated on 2026-10-16.
- Available measurements already observed: import totals T1/T2, total power, gas meter readings, 15-minute average power and current-month peak.
- Gas readings can update at a different timestamp from electricity.
- Existing Fluvius exports cover historical electric/gas measurements and official monthly peaks. The 15-minute electrical dataset is substantial; preserve the original CSVs as the source of truth.
- The agreed supply contract starts **2026-10-21**, with MEGA **Zen Fixed 3Y** for electricity and gas on the **09/2026** tariff cards. Earlier Ecofix history does not require retrospective cost reconstruction as part of phase 1.
- The HomeWizard app already holds approximate all-in rates starting on the new contract date. **Do not reuse those averaged rates together with componentized Belgian integration fees**, which would double-count costs.

**Public repository privacy boundary:** Do not commit EANs, meter serial numbers, supplier account identifiers, access tokens, user-specific readings, consumption exports, actual invoices or other private billing records. Exact cost configuration belongs in the future local deployment, with private operational records as appropriate.

## 4. Planned components and responsibilities

| Component | Role | Deployment |
| --- | --- | --- |
| Home Assistant Container | Host integrations, Recorder, Energy Dashboard and browser UI | One proposed Docker Compose service, not deployed |
| SQLite Recorder | Store current energy readings and long-term aggregate statistics | Inside HA's persisted config storage |
| Official HomeWizard integration | Read electric counters, gas, power, 15-minute mean and peak sensors exposed by the device | Built into Home Assistant |
| Belgian Electricity Prices | Estimate Belgian electricity price, distribution/network/tax and capacity components in EUR | HA custom integration, not separate container |
| Belgian Gas Prices | Estimate gas energy, network/tax/fixed components and m³-to-kWh conversion in EUR | HA custom integration, not separate container |
| Existing NPM + AdGuard + Tailscale | Trusted HTTPS/browser access | Reuse current platform; no new public endpoint |
| Fluvius historical importer | Optional, one-time historical backfill | Temporary/manual task only, **not** a permanent service |

HACS is **not** a required additional runtime service. Prefer supported manual installation of the two custom integrations if that keeps the PoC simpler; HACS may be reconsidered for maintenance convenience, not assumed as a prerequisite.

## 5. Tariff and dashboard behavior

- Treat meter registers **T1 + T2** as consumption under a **single contractual electricity price**. Ensure any HA energy source configuration does not accidentally price both counters twice.
- Configure the agreed MEGA **Zen Fixed 3Y** tariff from its signing-month **09/2026** card with supply effective from **2026-10-21**. Confirm that the community integration respects the signing card rather than automatically substituting a later offer.
- Prefer the componentized Belgian integration for the estimated full electricity/gas cost. Do **not** add its capacity/fixed/network fees a second time through another sensor or the Energy Dashboard.
- Separate consumption/commodity-only displays from more complete estimated spending. If the built-in Energy Dashboard and a provider's `current_year_cost` differ in scope, label that difference explicitly instead of claiming invoice accuracy.
- For gas, preserve m³ volume and use an appropriate changing calorific conversion or an available published factor when needed. Avoid treating `€/kWh` and `€/m³` as interchangeable.
- Capacity tariff: consume HomeWizard's **measured monthly peak** (rather than sampled instantaneous power), apply the Belgian integration's rolling 12-month methodology, and accept an estimate/temporary fixed baseline until sufficient months are collected.
- Use the existing dashboard first: real-time W, electric kWh history, gas m³ history, historical comparisons, peak and EUR cost views. Avoid introducing an elaborate dashboard stack.

The integration's cost sensors are **estimates**, not a promise to match a supplier invoice cent-for-cent. In particular, promotional discounts and historical cost recreation are not release gates.

## 6. Historical Fluvius import — best effort, non-blocking

**Desired:** Bring historical electricity and gas consumption into the self-hosted dashboard, preferably after live collection, initial costs, persistence and backups are validated.

First evaluate the existing [Home Assistant Import Energy Data](https://github.com/patrickvorgers/Home-Assistant-Import-Energy-Data) tool and its Fluvius converter. A limited **one-off** adaptation of a converter is acceptable; a new always-running ingestion application or custom billing service is not.

Safety/correctness checks before importing:

1. Work only against an isolated/test HA database or a backup that can be restored; preserve untouched source CSVs and do not overwrite live history blindly.
2. Map the **actual Dutch Fluvius export column names** (e.g. parenthesized date/time columns), units and validation statuses; do not assume the upstream sample's schema matches.
3. Treat `Europe/Brussels` DST transitions explicitly: avoid silently merging the repeated autumn hour or shifting the spring gap.
4. Normalize electricity T1/T2 and handle gas **m³ and kWh as alternative unit representations**, not two volumes to add. Some gas exports use day boundaries from **06:00**.
5. Check imported totals and timeline against the source exports; avoid gaps, duplicate readings and spikes where historical and current cumulative series meet.
6. Prefer correct **hourly long-term statistics** over trying to keep the original 15-minute detail permanently in HA.
7. If worthwhile, evaluate seeding the historical monthly peak bank from official Fluvius values separately. Do **not** mutate the community integration's private persistence format without an approved, low-risk method.
8. Validate rollback and re-run/idempotence before touching real data.

**Stop condition:** If the converter is unreliable, requires intrusive database manipulation, or becomes disproportionately complex, skip the import. The original Fluvius files remain available outside this project, and the live Home Assistant system remains a valid success.

## 7. Infrastructure and security boundaries

- Future suggested stack location: `/srv/docker/homeassistant` (**proposal, not created**).
- Pin a reviewed Home Assistant image version; choose a suitable restart policy, persistent config volume and local timezone `Europe/Brussels`.
- Keep SQLite, custom integrations and settings within the backed-up persistent storage. Plan a consistent backup and at least one restore check.
- Check HomeWizard discovery/connectivity from the selected Docker networking mode. Do not add host networking, mDNS sidecars or additional collectors unless a real test demonstrates the need.
- Future access hostname could be `energy.pirocorp.com` (**proposal only, not an existing DNS/NPM configuration**). Confirm HA reverse-proxy `trusted_proxies`, headers and application access control before publishing through NPM.
- LAN/Tailscale only; no direct internet exposure, router forwarding or automatic cloud dependency for local measurements.
- No credentials or meter data in Git; keep secrets/config with appropriate local permissions.
- Do not touch Philips Hue, smart appliances, UPS/NUT, Homepage widgets or other existing homelab workloads as part of energy phase 1.

## 8. PoC and implementation plan (not authorization)

### Phase 0 — PoC planning (next)

- Confirm a pinned Home Assistant Container image, Compose layout, volumes, restore method, update approach and appropriate network mode.
- Decide how the two custom integrations will be installed/updated without adding avoidable maintenance complexity.
- Choose whether the dashboard will be HA's stock Energy view plus a small supplemental peak/cost view.
- Define an isolated or reversible PoC boundary before any actual deployment.
- Review NPM reverse-proxy and local authentication requirements; do not reserve DNS or ports yet.

### Phase 1 — Live core (requires separate approval)

1. Bring up the single planned HA container and persistent SQLite Recorder.
2. Verify that the official HomeWizard integration exposes reliable live electricity, gas and peak sensor values.
3. Configure the Energy Dashboard without double-counting T1/T2.
4. Add the two Belgian price integrations, contract start/tariff settings and network/capacity estimates.
5. Verify visible trends and EUR estimates, current monthly peak, restart persistence and missing-data behavior.

### Phase 2 — Trusted access and operations (requires separate approval)

1. Validate backups and a restore drill.
2. Reuse NPM/AdGuard/Tailscale under an approved hostname after confirming access controls.
3. Optionally add a Homepage navigation/status entry **only if separately approved**.
4. Document measured versions, final stack location, access URL and as-built operations.

### Phase 3 — Optional historical enrichment

1. Test Fluvius export conversion and hourly history import offline/reversibly.
2. Compare totals/time ranges and validate DST and gas boundaries.
3. Import only if it remains safe and simple; otherwise mark the task deferred/skipped with the reason.
4. Attempt historical monthly peak seeding only if there is an established safe method.

## 9. Acceptance checklist

**Architecture decisions — accepted 2026-10-16**

- [x] Home Assistant Container selected over OmniMeter.
- [x] SQLite, one application container, no Grafana/InfluxDB/database server.
- [x] Official HomeWizard integration and ready-made Belgian electricity/gas price integrations selected for evaluation in PoC.
- [x] Preserve T1/T2 data but use one contract price for both.
- [x] Current Belgian capacity tariff estimates required; exact invoice matching is not.
- [x] Historical Fluvius import stays planned **best effort** and **non-blocking**.
- [x] Architecture approval **does not authorize deployment**.

**Implementation verification — all pending**

- [ ] Separate authorization to deploy an isolated/reversible PoC.
- [ ] Live electricity, gas and peak entities verified in HA.
- [ ] T1/T2 energy totals are consistent, with no double count.
- [ ] Belgian cost sensors display credible EUR estimates and capacity costs.
- [ ] User-friendly browser dashboard works on phone and desktop.
- [ ] Recorder history persists across restarts and backup/restore succeeds.
- [ ] Access is limited to trusted LAN/Tailscale clients with appropriate authentication.
- [ ] Historical import attempted and documented as successful, partial or skipped (**not a blocking acceptance gate**).
- [ ] No unrelated services or equipment changed.

## 10. Previous candidate and documentation transition

The earlier [OmniMeter roadmap](../omnimeter/README.md) is retained as a **historical architecture evaluation**, not a second active energy-monitoring deployment proposal. Its older “candidate” wording records the state when it was written, not the current approved decision. The gap was insufficient ready-made Belgian cost/capacity support relative to Home Assistant.

[Draft PR #30](https://github.com/pirocorp/Ubuntu-Media-Server-Homelab-Setup-Guide/pull/30) separately records independent HomeWizard API v2 testing and OmniMeter field-mapping gaps. This roadmap does not modify that PR or its file; reconcile/archive that draft separately so historical evidence is not lost.

Only **after implementation is validated**: create `docs/services/home-energy-monitoring/README.md`; record actual installed image/version, Compose service, persistence, network/authentication, backup/restore and costs/peak limitations; then move this roadmap and indexes from Planned to Implemented. Do not create an as-built service entry before deployment.

## 11. References

- [Home Assistant Container](https://www.home-assistant.io/installation/linux#install-home-assistant-container)
- [Home Assistant HomeWizard integration](https://www.home-assistant.io/integrations/homewizard/)
- [Home Assistant Energy management](https://www.home-assistant.io/docs/energy/)
- [Belgian Electricity Prices integration](https://github.com/renaudallard/homeassistant_be_electricity_prices)
- [Belgian Gas Prices integration](https://github.com/renaudallard/homeassistant_be_gas_prices)
- [Historical import candidate](https://github.com/patrickvorgers/Home-Assistant-Import-Energy-Data)
- [HomeWizard Local API documentation](https://api-documentation.homewizard.com/)

**Guardrail:** Documentation-only approved architecture; no host, GitHub deployment files, live DNS, meter or HomeWizard app changes are approved by this decision.
