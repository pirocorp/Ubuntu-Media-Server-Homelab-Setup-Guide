# Homepage Server Health — Phase 1 (without SMART)

Status: Planned — architecture recorded; not deployed
Purpose: Replace the heavy Netdata viewing workflow with a compact, low-maintenance server/UPS status display inside the existing Homepage.
Depends on: Existing Ubuntu Docker host, deployed Homepage, NUT UPS monitoring, existing Docker Socket Proxy, and the current AdGuard/NPM/Tailscale ingress.
Related docs: [Roadmaps index](../README.md), [Homepage deployment runbook](../../services/homepage/README.md), [Homepage original architecture decision record](../homepage/README.md), [UPS / Netdata runbook](../../services/ups-monitoring/README.md), [Docker and Portainer](../../platform/docker-and-portainer.md)

## Decision Record and Implementation Brief

**Status:** Agreed phase-1 direction; runtime configuration and acceptance tests pending  
**Version:** 0.1  
**Date:** 2026-10-14

**This PR documents a planned change only.** It neither deploys Glances/PeaNUT nor alters Homepage, NUT, Netdata, storage, Samba, firewall rules, or live access paths.

## 1. Goal and implementation order

User goal: one **simple** dashboard for CPU, RAM, CPU temperature, disk free space, and basic UPS condition. Historical time-series analysis and sophisticated observability are not required.

**Phase 1 — baseline (this roadmap):** Homepage + Glances + PeaNUT, with server metrics and UPS values verified on desktop and iPhone.

**Phase 2 — optional extension, separately implemented:** SMART Health, physical-drive temperature, and SSD wear indicator for the six drives. The separate SMART solution previously discussed is host \`smartctl\` -> normalized read-only JSON -> Homepage card. Do **not** make its implementation a prerequisite for phase 1.

**Phase 3 — only after phase-1 sign-off:** decide whether to retire Netdata; document cleanup and monitoring/alert trade-offs separately.

## 2. Existing environment — do not disturb

- Ubuntu host \`piroman-server\` at \`192.168.0.10\`.
- Existing independent Docker Compose stacks typically under \`/srv/docker/<service>\`.
- Homepage is deployed at \`https://home.pirocorp.com\`, image \`ghcr.io/gethomepage/homepage:v2.4.0\`, listening on \`192.168.0.10:3002\` via Nginx Proxy Manager.
- Homepage uses Docker Socket Proxy on private network \`homepage_docker_api\`; direct Docker-socket mounting in Homepage remains forbidden.
- Homepage's regular (non-Docker-API) network is \`homepage_default\`. Verify this **live** before attaching other containers.
- Existing Homepage service cards, categories, credentials and reverse proxy setup work; avoid layout churn.
- NUT already manages UPS monitoring/shutdown on the host, with UPS identifier **\`powerwalker\`**. Observed \`upsd\` listens only on \`127.0.0.1:3493\` and \`[::1]:3493\`.
- Observed UPS fields include \`battery.charge\`, \`ups.status\`, \`ups.load\` (and optional runtime). Example observed \`ups.status: OL CHRG\` means online and charging; do not treat \`OL CHRG\` as unknown.
- Netdata remains installed and currently supports the existing Homepage \`Server / Netdata\` widget at \`https://server.pirocorp.com\`. Do not remove it until the replacement has passed tests and any current alerts have been reviewed.
- Five data drives are NTFS mounted using \`fuseblk\` and shared to Windows through Samba. No filesystem conversion, remount, Samba change or disk write is part of this project.

### Six filesystem paths to report

| Drive | Mount point | Observed used | Observed free |
| --- | --- | --- | --- |
| Samsung 870 SATA SSD ~2 TB | \`/\` | 11% | 1.6 TB |
| Seagate ~4 TB | \`/mnt/iac\` | 52% | 1.8 TB |
| Seagate ~8 TB | \`/mnt/lp\` | 52% | 3.6 TB |
| Seagate ~8 TB | \`/mnt/data\` | 47% | 4.0 TB |
| Seagate ~4 TB | \`/mnt/ia\` | 80% | 748 GB |
| Seagate ~4 TB | \`/mnt/comp\` | 34% | 2.5 TB |

Values are **reference snapshots**, not configuration constants or live values. Refresh them via the monitoring source. \`/mnt/ia\` is the most occupied filesystem at the time of observation.

## 3. Phase-1 requirements — accepted

| Area | Visible data | Proposed source |
| --- | --- | --- |
| Host CPU | Usage (%) | Glances |
| Host RAM | Used/free/total | Glances |
| CPU temperature | Celsius, if an actual CPU sensor can be detected | Glances |
| Filesystems | Used %, free and total for the six named mount points | Glances |
| UPS battery | Charge (%) | PeaNUT -> existing NUT |
| UPS state | Online / on battery, preserving compound status like \`OL CHRG\` | PeaNUT -> existing NUT |
| UPS load | Load (%) | PeaNUT -> existing NUT |
| Docker | Existing running/stopped states and on-demand container metrics | **Existing** Homepage + Docker Socket Proxy |

Only one user-facing main dashboard: **Homepage**. The UPS shutdown path stays independent from any dashboard application.

No requirement for second-level raw history, database, Grafana, InfluxDB, Home Assistant, Beszel or a custom monitoring web application.

## 4. Selected topology and network/security boundaries

\`\`\`text
Trusted browser (LAN or Tailscale)
  -> HTTPS home.pirocorp.com
  -> existing Nginx Proxy Manager
  -> existing Homepage container (:3000; host 192.168.0.10:3002)
       |-> Glances v4 HTTP API on a shared *non-socket-proxy* Docker network
       |     -> read-only visibility into Ubuntu host sensors / file systems
       |
       |-> PeaNUT HTTP API on a chosen host-local/LAN interface & protected port
             -> existing NUT on 127.0.0.1:3493
                   -> physical PowerWalker UPS + independent shutdown
\`\`\`

### Fixed architecture boundaries

1. Preserve the existing Homepage and its Docker Socket Proxy. **Do not** attach Glances/PeaNUT to \`homepage_docker_api\` or grant direct Docker socket access.
2. Glances is proposed as a standalone, pinned **4.x** container attached to the regular Homepage Docker network only. Homepage v2.4.0 expects \`version: 4\` for Glances v4; do not upgrade to v5 or select \`latest\` without a new API-compatibility test.
3. NUT stays on the Ubuntu host and remains solely responsible for safe shutdown. Do not move NUT into a container or reconfigure it to listen on the entire LAN simply to enable visualization.
4. PeaNUT in \`network_mode: host\` is the chosen **candidate** way to reach the host's \`127.0.0.1:3493\`. It needs a distinct, confirmed-free HTTP listener on \`192.168.0.10\` to let the Homepage container reach it. No Docker \`ports:\` directive in host-network mode.
5. The PeaNUT HTTP endpoint is **not** a public service. Bind only to the necessary host interface, enable its built-in authentication, and restrict ingress from Homepage at the firewall as feasible. Avoid exposing the UI/API from the open Internet or creating NPM/public DNS routes.
6. Treat PeaNUT's HTTP API as a management interface: upstream supports UPS commands, so a NUT account used for this project must not be a privileged/shutdown-capable NUT identity. Prefer unauthenticated NUT read queries when supported, with protected PeaNUT HTTP credentials.
7. No privileged Glances container, no broad host-network exposure of Glances, no Docker socket for Glances, and no writes to mounted host filesystems.
8. Images must be reviewed and **pinned** before actual deployment. Actual HTTP ports, run UID, volume visibility, API version and auth details must pass preflight; no arbitrary port assumptions.

## 5. Glances data-source implementation proposal

Use a separately managed Docker stack \`/srv/docker/glances\` in web/API mode (\`GLANCES_OPT=-w\`), on the existing \`homepage_default\` network (after verification).

**Important caveat:** \`pid: host\` alone does not make container namespaces report the host's complete file systems/sensors. Before fixing the Compose volumes, test correct host CPU/RAM, CPU temperature, and all six *named* filesystem usage values against \`/proc\`/\`df\`/\`sensors\` on Ubuntu. Provide only explicit read-only mounts needed to make data visible.

- Candidate release: stable Glances v4.5.8 (review/pull exact v4 Docker image tag at deployment; do not use an unpinned floating tag).
- API must listen where the Homepage container can connect but not be host-published to the LAN unnecessarily.
- Mount host filesystem paths read-only as required, including \`/\`, \`/mnt/iac\`, \`/mnt/lp\`, \`/mnt/data\`, \`/mnt/ia\`, \`/mnt/comp\`; do **not** grant write access to NTFS or Samba shares.
- Check that these mounts are exposed in the Glances \`fs\` API under the names used by Homepage. A different in-container path is acceptable if correctly mapped in Homepage and labelled for the user.
- Temperature must come from a real host sensor (e.g. Package/Core/Tctl), not a fake zero or an unrelated container/drive sensor. If unavailable, omit the CPU temperature rather than inventing it.
- Validate host CPU/RAM readings versus \`top\`/\`free\` in a controlled load/idle comparison, including host memory visibility with the chosen Docker configuration.

### Homepage Glances information widget — example (not applied)

In \`/srv/docker/homepage/config/widgets.yaml\`, after confirming DNS/network reachability:

\`\`\`yaml
- glances:
    url: http://glances:61208
    version: 4
    cpu: true
    mem: true
    cputemp: true
    uptime: true
    label: Server
    expanded: true
\`\`\`

The \`disk\` option supports a list of mount points, **but in v2.4.0 that information widget renders each with a generic "free" label, not its mount name**. Therefore do not approve an unlabeled six-disk strip as the final UI. Try compact **named** Glances service widgets as a baseline, e.g.:

\`\`\`yaml
- Storage:
    - HDD · ia:
        widget:
          type: glances
          url: http://glances:61208
          version: 4
          metric: fs:/mnt/ia
          chart: false
\`\`\`

Repeat only for mounts actually returned by Glances. Test the six labels and mobile grouping; prefer fewer legible items to an indistinguishable wall of cards. The definitive placement and layout remain a **UI acceptance gate**, not a fully locked design.

## 6. PeaNUT data-source implementation proposal

Use a separate stack \`/srv/docker/peanut\`; keep \`/config\` persistent under that root with UID/GID permissions matching the chosen image.

- Connect to NUT server \`127.0.0.1:3493\`, device \`powerwalker\`.
- First approach: Docker \`network_mode: host\`; this gives the container access to the host's loopback NUT server without changing NUT's listener.
- Set \`WEB_HOST=192.168.0.10\` (candidate bind interface), \`WEB_PORT=<CHECK_FOR_FREE_PORT>\` and \`AUTH_DISABLED=false\`. \`NUT_HOST=127.0.0.1\` and \`NUT_PORT=3493\` are supported as backward-compatible configuration inputs upstream; confirm these work with the *pinned* release. No \`ports:\` in host-network mode.
- Use an authenticated PeaNUT API for Homepage, storing credentials as existing local Homepage file-backed secrets, not in Git. Verify whether additional PeaNUT user permissions or commands need restriction.
- UFW access must be as narrow as practicable for the Homepage origin. Do not allow the whole Internet or expose the NUT port \`3493\` beyond localhost.
- **Validation:** from Homepage's network, authenticated \`GET /api/v1/devices/powerwalker\` must return charge/load/status without granting UPS control.

### Homepage UPS card — example (not applied)

In \`/srv/docker/homepage/config/services.yaml\`:

\`\`\`yaml
- Infrastructure:
    - PowerWalker UPS:
        widget:
          type: peanut
          url: http://192.168.0.10:<CHECK_FOR_FREE_PORT>
          key: powerwalker
          username: "{{HOMEPAGE_VAR_PEANUT_USER}}"
          password: "{{HOMEPAGE_VAR_PEANUT_PASSWORD}}"
\`\`\`

**Credentials and template variable form are illustrative only:** match the *existing* Homepage file-backed secrets convention and test whether this widget accepts the substitutions. Do not commit passwords.

Homepage v2.4.0 PeaNUT widget supports precisely \`battery_charge\`, \`ups_load\` and \`ups_status\`. Its renderer translates exactly \`OL\`, \`OB\`, \`LB\`; a compound NUT status like \`OL CHRG\` is shown as the raw text. That is valid status, **not** an error; decide whether the raw string is acceptable on iPhone or a later narrow normalization is needed.

## 7. Preflight and rollout plan (future implementation, not this PR)

1. Record existing Netdata alert details, current Homepage screenshots/backup, current resources, active ports/networks and NUT status.
2. Confirm Ubuntu architecture, available CPU temperature sensors and the pinned Glances image/API v4 compatibility.
3. Start isolated Glances PoC with least-privilege mounts/network; test API and host CPU/RAM/sensor/storage readings against known host commands. Do not touch Homepage if not correct.
4. Evaluate disk display in Homepage \`widgets.yaml\` and/or named \`services.yaml\` cards; approve desktop and narrow iPhone layouts before finalizing.
5. Preflight PeaNUT version, bind address/port, local NUT access from host network, nonprivileged UPS read access, authentication, and firewall path from Homepage.
6. Verify the three UPS fields and correct status interpretation at \`ups.status=OL CHRG\`; ensure no change to existing NUT shutdown/monitor service.
7. Add only the minimal necessary Homepage widgets. Keep other Homepage cards, Docker Socket Proxy and existing NPM/AdGuard setup unchanged.
8. Run both monitoring paths in parallel with Netdata for an observation period; compare readings during idle and load, and verify absence of intermittent errors.
9. Investigate the pre-existing Netdata warning, and document any lost alerts/notification functions.
10. After user acceptance, create a **separate** removal/cleanup decision for Netdata and update as-built documentation. If validation fails, revert Homepage widget changes and stop the two experimental containers; Netdata remains available.

## 8. Phase-1 acceptance checklist

- [ ] No configuration or running services changed by this roadmap PR.
- [ ] Glances image is pinned to a tested Homepage-v2.4.0-compatible API v4 release.
- [ ] Homepage shows **host** CPU usage and memory accurately versus Ubuntu measurements.
- [ ] Homepage shows real CPU temperature (if supported) or explicitly documents the unavailable sensor.
- [ ] Each of \`/\`, \`/mnt/iac\`, \`/mnt/lp\`, \`/mnt/data\`, \`/mnt/ia\`, \`/mnt/comp\` shows correctly labelled and accurately measured free/used/total.
- [ ] No NTFS filesystem, mount option or Samba share changed, and disk data remains untouched.
- [ ] PeaNUT reads \`powerwalker\` at host loopback NUT without changing NUT's safety/shutdown role.
- [ ] Homepage shows UPS battery %, load %, and online/on-battery state, including compound statuses.
- [ ] PeaNUT API protected with credentials and restricted access; NUT port remains local-only.
- [ ] No unnecessary host-published Glances API, new NPM route, wildcard DNS rewrite or router forwarding.
- [ ] Existing Homepage Docker Socket Proxy and all prior widgets still work.
- [ ] Layout works on iPhone as well as desktop without overwhelming the initial dashboard.
- [ ] Existing Netdata warning investigated; old Netdata stays operational until user approves decommission.
- [ ] SMART Health/drive temperatures/wear remain separate future work and are **not** a phase-1 blocker.

## 9. After successful deployment

- Record the actual Glances and PeaNUT pinned versions, Compose roots, health checks, networks, port, firewall/access policy and persistent configuration in \`docs/services/\` runbooks.
- Update the **implemented** service inventory and current Homepage runbook with actual measurements/limitations, not proposals.
- Preserve the original \`docs/roadmaps/homepage/README.md\` as a historical architecture decision; this follow-up supersedes only the *future* Netdata-vs-Homepage monitoring direction, not the already-deployed baseline.
- Remove or replace the Netdata service card **only after** user-reviewed, successful parallel testing.
- SMART extension can be tracked and implemented separately using host \`smartctl\` / JSON, without blocking this phase.

## 10. Documentation references

- Homepage v2.4.0 Glances information widget: https://github.com/gethomepage/homepage/blob/v2.4.0/docs/widgets/info/glances.md
- Homepage v2.4.0 Glances service widget: https://github.com/gethomepage/homepage/blob/v2.4.0/docs/widgets/services/glances.md
- Homepage v2.4.0 PeaNUT widget: https://github.com/gethomepage/homepage/blob/v2.4.0/docs/widgets/services/peanut.md
- Glances Docker docs: https://glances.readthedocs.io/en/latest/docker.html
- Glances v4.5.8 release: https://github.com/nicolargo/glances/releases/tag/v4.5.8
- PeaNUT upstream (config, NUT settings and API): https://github.com/Brandawg93/PeaNUT

**Implementation guardrail:** Do not interpret this roadmap or sample YAML as permission to modify a running server. The next action after review is a controlled phase-1 PoC, **not** installation of SMART or immediate Netdata removal.