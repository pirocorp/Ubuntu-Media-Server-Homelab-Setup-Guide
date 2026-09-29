# Homepage Homelab Dashboard Architecture

Status: Implemented baseline; retained as architecture decision record
Purpose: Preserve the approved Homepage design and record how the deployed implementation differs from the pre-implementation plan.
Depends on: Existing Docker runtime, Nginx Proxy Manager, AdGuard Home, Tailscale, and deployed homelab services
Related docs: [Homepage runbook](../../services/homepage/README.md), [Roadmaps index](../README.md), [Current state](../../overview/current-state.md), [Service inventory](../../overview/service-inventory.md), [Docker and Portainer](../../platform/docker-and-portainer.md), [Networking and reverse proxy](../../platform/networking-and-reverse-proxy.md)

## Implementation Outcome

**Status:** Deployed on 2026-09-29; canonical Homepage route operational. Two final validation items remain: trusted-Tailscale access and authoritative/public apex-DNS resolution for `pirocorp.com`.  
**Original design version:** 1.0  
**Original design date:** 2026-09-25

The architecture below is retained as the original decision record. The authoritative live implementation is documented in the [Homepage deployment and operations runbook](../../services/homepage/README.md).

The deployed implementation preserves the major locked decisions with these explicit outcomes and deviations:

- Homepage runs under `/srv/docker/homepage`.
- Homepage is pinned to `ghcr.io/gethomepage/homepage:v2.4.0`.
- Docker Socket Proxy is pinned to `ghcr.io/tecnativa/docker-socket-proxy:v0.4.2`.
- Homepage does not mount `/var/run/docker.sock` directly.
- the socket proxy is private to Docker, has no host-published `2375`, and uses `POST=0`.
- the planned host port `3000` conflicted with the deployed AdGuard Home web UI, so Homepage uses `192.168.0.10:3002 -> container:3000`.
- Nginx Proxy Manager therefore forwards `home.pirocorp.com` to `http://192.168.0.10:3002`.
- `https://home.pirocorp.com` is operational and is the canonical dashboard URL.
- AdGuard has explicit rewrites for `home.pirocorp.com`, `pirocorp.com`, and `www.pirocorp.com`.
- NPM has a `301` redirection host for both `pirocorp.com` and `www.pirocorp.com` to `https://home.pirocorp.com`.
- Cloudflare has DNS-only private-address records for `*.pirocorp.com` and the apex `pirocorp.com`; final authoritative/public validation of the newly-created apex record is still pending.
- the dashboard contains the agreed native widgets plus Stremio and AIOStreams Docker-status cards.
- all service credentials are host-local file-backed secrets referenced through `HOMEPAGE_FILE_*` substitutions.
- the NetAlertX v2 widget required a narrow UFW rule from `172.31.0.0/16` to `192.168.0.10:20212/tcp`.

## Locked Decision Record and Implementation Brief

The remainder of this document preserves the pre-implementation design language for traceability. Where it conflicts with the implementation outcome above, the implementation outcome and live runbook are authoritative.

## 1. Purpose

Homepage was designed to become the central operational landing page for the homelab. It must not be implemented as a simple bookmark grid. The dashboard should provide a compact view of service health and useful service-specific state while leaving deep operational monitoring to the tools that already own it, especially Netdata and Portainer.

This document preserves the architecture decisions already agreed. Items marked **LOCKED** were requirements for implementation unless explicitly reopened.

## 2. Existing environment

The implementation had to fit the current homelab rather than introduce a parallel platform model.

Original assumptions:

- Ubuntu host: `piroman-server`
- LAN IP: `192.168.0.10`
- Docker Compose stacks live under `/srv/docker/<service>`
- Nginx Proxy Manager is the reverse proxy and HTTPS entry point
- AdGuard Home provides explicit local DNS rewrites; wildcard DNS rewrites are intentionally not used
- the wildcard certificate already covers `pirocorp.com` and `*.pirocorp.com`
- Tailscale advertises `192.168.0.0/24` to trusted remote clients
- existing applications normally expose host ports and are proxied by Nginx Proxy Manager through `192.168.0.10:<port>`

## 3. Scope boundary

### In scope

- deploy Homepage under `/srv/docker/homepage`
- provide Docker container status and on-demand container resource statistics
- use native Homepage widgets where they add useful operational information
- use HTTP status monitoring where no useful native widget exists
- publish Homepage at `https://home.pirocorp.com`
- make `https://pirocorp.com` redirect permanently to the canonical Homepage URL
- add the required explicit AdGuard DNS rewrites
- add the required Nginx Proxy Manager proxy and redirect configuration
- keep credentials out of committed YAML where practical
- document the final deployed state after implementation is validated

### Out of scope for the initial implementation

- redesigning the existing LAN or Tailscale topology
- changing existing services from host-port publishing to shared reverse-proxy Docker networks
- introducing VLANs or a dedicated management network
- replacing Netdata, Portainer, AdGuard Home, Nginx Proxy Manager, or NetAlertX
- adding new monitoring products only to enrich Homepage
- exposing Homepage directly to the public Internet

## 4. Locked high-level architecture

Original target:

```text
LAN / trusted Tailscale client
          |
          v
      AdGuard Home
          |
          v
  192.168.0.10 / NPM
          |
          | HTTPS proxy
          v
  192.168.0.10:3000
          |
          v
       Homepage
          |
          | private Docker network only
          v
 Docker Socket Proxy
          |
          | /var/run/docker.sock:ro
          v
     Docker daemon
```

Actual host port is `3002`, not `3000`, because `3000` is already occupied by AdGuard Home.

### LOCKED decisions

1. Homepage runs as its own Docker Compose stack under `/srv/docker/homepage`.
2. Homepage does **not** receive `/var/run/docker.sock` directly.
3. A Docker Socket Proxy sidecar is the only container in the Homepage stack with the Docker socket mounted.
4. Homepage and the Docker Socket Proxy share a dedicated private Docker network used only for Docker API access.
5. The Docker Socket Proxy does not publish port `2375` to the host or LAN.
6. Docker API mutation through the proxy is disabled with `POST=0`, and only read endpoints required by Homepage are enabled.
7. The planned host port was `3000`; implementation uses `3002` because AdGuard already owns `3000`.
8. Nginx Proxy Manager forwards `home.pirocorp.com` to `http://192.168.0.10:3002`.
9. `https://home.pirocorp.com` is the canonical Homepage URL.
10. `https://pirocorp.com` is configured for permanent `301` redirection to `https://home.pirocorp.com`; `www.pirocorp.com` was added to the same redirect host.
11. AdGuard Home has explicit rewrites for `home.pirocorp.com`, `pirocorp.com`, and `www.pirocorp.com` to `192.168.0.10`.
12. Homepage service definitions are centralized in `services.yaml`; Docker labels are not spread across existing service stacks for dashboard configuration.
13. Docker statistics are not permanently expanded. Service cards stay compact, with CPU, memory, and network statistics available on demand through the Docker status indicator.
14. Homepage complements Netdata and Portainer; it does not replace them.

## 5. Docker access security model

Direct Docker socket access was rejected because a process with access to the Docker daemon API can potentially gain broad control of the Docker host. A read-oriented Docker Socket Proxy provides a smaller interface and lets the deployment explicitly disable mutating API methods.

```text
Homepage
    |
    | HTTP Docker API on private Docker network
    v
Docker Socket Proxy
    |
    | mounted socket
    v
/var/run/docker.sock
```

The proxy network does not contain unrelated application containers. The proxy itself does not expose a host port.

This protects primarily against compromise of the Homepage application container. It is not intended to protect against an attacker who already controls the Ubuntu host or Docker daemon.

## 6. Ingress and domain model

### Canonical route

```text
https://home.pirocorp.com
    -> AdGuard / DNS compatibility fallback: 192.168.0.10
    -> Nginx Proxy Manager :443
    -> http://192.168.0.10:3002
    -> Homepage
```

### Redirect routes

```text
https://pirocorp.com
https://www.pirocorp.com
    -> Nginx Proxy Manager
    -> 301 https://home.pirocorp.com
```

The existing certificate covering both `pirocorp.com` and `*.pirocorp.com` is reused.

The deployed design uses the same host-port ingress model as the other deployed applications. A dedicated NPM-to-Homepage Docker network was not introduced.

## 7. Dashboard information model

### 7.1 Infrastructure

| Card | Status source | Native widget | Deployed fields |
| --- | --- | --- | --- |
| Server / Netdata | HTTP monitor | Netdata | `warnings`, `criticals` |
| Portainer | Docker | Portainer | `running`, `stopped`, `total` |
| Nginx Proxy Manager | Docker | NPM | `enabled`, `disabled`, `total` |
| AdGuard Home | Docker | AdGuard | `queries`, `blocked`, `filtered`, `latency` |
| NetAlertX | Docker | NetAlertX v2 | `total`, `connected`, `new_devices`, `down_alerts` |

Netdata remains the authoritative destination for detailed host CPU, RAM, disk, network, and UPS monitoring. Homepage surfaces alert counts instead of duplicating the Netdata dashboard.

### 7.2 Media and libraries

| Card | Status source | Native widget | Deployed fields |
| --- | --- | --- | --- |
| Plex | Docker | Plex | `streams`, `movies`, `tv` |
| Immich | Docker | Immich v2 | `users`, `photos`, `videos`, `storage` |
| Audiobookshelf | Docker | Audiobookshelf | `books`, `booksDuration` |
| Kavita | Docker | Kavita | `seriesCount`, `totalFiles` |
| Stremio | Docker | none | Docker status only |
| AIOStreams | Docker | none | Docker status only |

Tautulli is not part of the initial implementation.

### 7.3 Downloads and discovery

| Card | Status source | Native widget | Deployed fields |
| --- | --- | --- | --- |
| qBittorrent | Docker | qBittorrent | `leech`, `download`, `seed`, `upload` |
| Bitmagnet | Docker + HTTP monitor | none | compact service health only |

Bitmagnet does not have a custom API widget in the initial implementation. Deep resource and database monitoring remains in Netdata and Portainer.

### 7.4 Cloud and applications

| Card | Status source | Native widget | Deployed fields |
| --- | --- | --- | --- |
| Nextcloud | Docker | Nextcloud | `activeusers`, `numfiles`, `numshares`, `freespace` |
| ShadowBroker | Docker + HTTP monitor | none | frontend Docker state plus frontend HTTP reachability |

ShadowBroker remains one logical card. Its frontend, backend, and I2P containers are not separate top-level dashboard cards.

## 8. Status strategy

Services with useful native widgets use:

```text
Docker container state + native service widget
```

Services without useful native widgets use:

```text
Docker container state + HTTP siteMonitor
```

This applies to Bitmagnet and ShadowBroker. Stremio and AIOStreams intentionally use Docker status only.

Netdata is a host service and uses HTTP reachability plus the native Netdata alert widget.

## 9. Initial dashboard layout

```text
PIROCORP HOMELAB

Infrastructure
  Server / Netdata
  Portainer
  Nginx Proxy Manager
  AdGuard Home
  NetAlertX

Media & Libraries
  Plex
  Immich
  Audiobookshelf
  Kavita
  Stremio
  AIOStreams

Downloads & Discovery
  qBittorrent
  Bitmagnet

Cloud & Applications
  Nextcloud
  ShadowBroker
```

The deployed visual policy is compact cards with dot-style status indicators and Docker statistics hidden until explicitly expanded.

## 10. Credential and secret boundaries

Implementation uses least-privileged or dedicated credentials where supported:

- Portainer: API key and verified environment ID `3`
- Nginx Proxy Manager: dedicated non-admin/view-oriented user
- AdGuard Home: authenticated API credentials
- NetAlertX: regenerated API token
- Plex: Plex token
- Immich: dedicated API key restricted to `server.statistics`
- Audiobookshelf: API key acting as a non-admin `homepage` user
- Kavita: dedicated authorization key
- qBittorrent 5.2+: API key rather than WebUI password
- Nextcloud: `NC-Token` rather than account password

Homepage secrets are not committed in clear text. The deployment uses `HOMEPAGE_FILE_*` file-backed substitutions with host-local secret files mounted read-only.

## 11. Components intentionally not added initially

- Glances
- PeaNUT
- Tautulli
- Tailscale API widget
- Uptime Kuma
- custom Bitmagnet API integration
- custom ShadowBroker API integration

These may be reconsidered later if they solve a demonstrated operational need.

## 12. Rejected alternatives

### Direct Docker socket mount

Rejected and not used.

### Dedicated NPM-to-Homepage Docker ingress network

Rejected for the initial deployment in favor of consistency with the current host-port ingress model.

### Serving Homepage directly on both `pirocorp.com` and `home.pirocorp.com`

Rejected. The apex redirects to the canonical origin; `www` was also added as a redirect alias.

### Docker-label-driven Homepage configuration

Rejected. Dashboard configuration remains centralized.

### Always-visible Docker resource statistics

Rejected. Resource telemetry remains available on demand and in Netdata/Portainer.

### One card per implementation container

Rejected. Database, Redis, worker, machine-learning, I2P, and sidecar containers remain implementation details.

## 13. Implementation-resolved decisions

- Homepage version: `v2.4.0`
- Docker Socket Proxy version: `v0.4.2`
- Homepage UID/GID: `1000:1000`
- Homepage host port: `3002`
- Portainer environment ID: `3`
- AdGuard widget URL: `http://192.168.0.10:3000`
- NetAlertX widget v2 backend: `http://192.168.0.10:20212`
- NetAlertX firewall rule: `172.31.0.0/16 -> 192.168.0.10:20212/tcp`
- secret strategy: `/srv/docker/homepage/secrets` + `HOMEPAGE_FILE_*`
- qBittorrent progress/size is not shown in the initial widget; compact transfer/seeding fields are used

## 14. Implementation guardrails outcome

The implementation follows the intended guardrails: pinned images, no direct Homepage Docker socket, no published socket-proxy port, `POST=0`, host-port/NPM ingress, explicit AdGuard rewrites, canonical `home.pirocorp.com`, incremental widget validation, file-backed secrets, and no additional monitoring product introduced solely for Homepage.

## 15. Acceptance criteria

- [x] Homepage runs under `/srv/docker/homepage`.
- [x] Homepage does not mount `/var/run/docker.sock` directly.
- [x] Docker Socket Proxy is not reachable through a published host port.
- [x] Docker container state appears in Homepage through the proxy.
- [x] Docker CPU, RAM, and network statistics can be expanded on demand.
- [x] `https://home.pirocorp.com` works on the LAN.
- [ ] `https://home.pirocorp.com` works from a trusted Tailscale client using the existing subnet-routing model.
- [ ] `https://pirocorp.com` redirects permanently to `https://home.pirocorp.com` end-to-end; NPM redirect is configured, but final public apex DNS validation is pending.
- [x] AdGuard uses explicit rewrites for `home.pirocorp.com`, `pirocorp.com`, and `www.pirocorp.com`.
- [x] Nginx Proxy Manager terminates TLS and proxies Homepage to `192.168.0.10:3002`.
- [x] Infrastructure cards show the agreed native widget fields.
- [x] Media/library cards show the agreed native widget fields.
- [x] qBittorrent shows useful transfer/seeding information.
- [x] Bitmagnet and ShadowBroker show compact operational health without unnecessary custom integrations.
- [x] No committed file contains real service credentials or API secrets.
- [x] The dashboard remains readable without permanently expanded container statistics.

## 16. Post-implementation documentation transition

This transition is implemented by the documentation PR that introduced the live Homepage runbook:

1. `docs/services/homepage/README.md` is the operating/deployment reference;
2. Homepage is listed in the implemented service inventory and services index;
3. `https://home.pirocorp.com` is listed in current access URLs;
4. `/srv/docker/homepage` is listed in active Docker stack roots;
5. architecture and current-state documents describe the deployed service;
6. actual pinned versions, container names, port mapping, secret strategy, widgets, UFW exception, and operational commands are recorded;
7. Homepage is removed from Planned Work; and
8. this roadmap is retained as the architecture decision record.

## 17. Official implementation references

- Homepage installation: https://gethomepage.dev/installation/
- Homepage Docker installation: https://gethomepage.dev/installation/docker/
- Homepage Docker integration: https://gethomepage.dev/configs/docker/
- Homepage services configuration: https://gethomepage.dev/configs/services/
- Homepage settings: https://gethomepage.dev/configs/settings/
- Homepage service widgets: https://gethomepage.dev/widgets/services/
- Docker Socket Proxy: https://github.com/Tecnativa/docker-socket-proxy

## 18. Historical implementation prompt

The original implementation prompt called for Homepage under `/srv/docker/homepage`, host port `3000`, Docker access through a dedicated non-published Docker Socket Proxy with POST operations disabled, canonical publishing at `https://home.pirocorp.com`, an apex redirect, centralized `services.yaml`, collapsed Docker resource statistics, least-privileged credentials, and file-backed secrets. The deployed implementation follows that design except that host port `3002` was selected after preflight identified AdGuard Home already using port `3000`; `www.pirocorp.com` was also added as a redirect alias.
