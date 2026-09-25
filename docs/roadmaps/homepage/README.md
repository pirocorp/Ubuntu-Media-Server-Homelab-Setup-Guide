# Homepage Homelab Dashboard Architecture

Status: Planned
Purpose: Record the approved Homepage dashboard architecture before implementation begins.
Depends on: Existing Docker runtime, Nginx Proxy Manager, AdGuard Home, Tailscale, and deployed homelab services
Related docs: [Roadmaps index](../README.md), [Current state](../../overview/current-state.md), [Service inventory](../../overview/service-inventory.md), [Docker and Portainer](../../platform/docker-and-portainer.md), [Networking and reverse proxy](../../platform/networking-and-reverse-proxy.md)

## Locked Decision Record and Implementation Brief

**Status:** Approved architecture; implementation not yet started  
**Version:** 1.0  
**Date:** 2026-09-25

## 1. Purpose

Homepage will become the central operational landing page for the homelab. It must not be implemented as a simple bookmark grid. The dashboard should provide a compact view of service health and useful service-specific state while leaving deep operational monitoring to the tools that already own it, especially Netdata and Portainer.

This document preserves the architecture decisions already agreed. Items marked **LOCKED** are requirements for implementation and should not be redesigned unless explicitly reopened.

## 2. Existing environment

The implementation must fit the current homelab rather than introduce a parallel platform model.

Current assumptions:

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

### LOCKED decisions

1. Homepage will run as its own Docker Compose stack under `/srv/docker/homepage`.
2. Homepage will **not** receive `/var/run/docker.sock` directly.
3. A Docker Socket Proxy sidecar will be the only container in the Homepage stack with the Docker socket mounted.
4. Homepage and the Docker Socket Proxy will share a dedicated private Docker network used only for Docker API access.
5. The Docker Socket Proxy will not publish port `2375` to the host or LAN.
6. Docker API mutation through the proxy will be disabled with `POST=0`, and only the read endpoints required by Homepage will be enabled.
7. Homepage will publish host port `3000`, following the current homelab ingress pattern used by the existing services.
8. Nginx Proxy Manager will forward `home.pirocorp.com` to `http://192.168.0.10:3000`.
9. `https://home.pirocorp.com` is the canonical Homepage URL.
10. `https://pirocorp.com` will return a permanent `301` redirect to `https://home.pirocorp.com` rather than serving a second Homepage origin.
11. AdGuard Home will receive explicit rewrites for both `home.pirocorp.com` and `pirocorp.com` to `192.168.0.10`.
12. Homepage service definitions will be centralized in `services.yaml`; Docker labels will not be spread across the existing service stacks for dashboard configuration.
13. Docker statistics will not be permanently expanded. Service cards should stay compact, with CPU, memory, and network statistics available on demand through the Docker status indicator.
14. Homepage will complement Netdata and Portainer, not replace them.

## 5. Docker access security model

Direct Docker socket access is rejected because a process with access to the Docker daemon API can potentially gain broad control of the Docker host. A read-oriented Docker Socket Proxy provides a smaller interface and lets the deployment explicitly disable mutating API methods.

Target relationship:

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

The proxy network must not contain unrelated application containers. The proxy itself must not expose a host port.

This protects primarily against compromise of the Homepage application container. It is not intended to protect against an attacker who already controls the Ubuntu host or Docker daemon.

## 6. Ingress and domain model

### Canonical route

```text
https://home.pirocorp.com
    -> AdGuard: 192.168.0.10
    -> Nginx Proxy Manager :443
    -> http://192.168.0.10:3000
    -> Homepage
```

### Apex route

```text
https://pirocorp.com
    -> AdGuard: 192.168.0.10
    -> Nginx Proxy Manager
    -> 301 https://home.pirocorp.com
```

The existing certificate covering both `pirocorp.com` and `*.pirocorp.com` should be reused.

The initial design deliberately uses the same host-port ingress model as the other deployed applications. A dedicated NPM-to-Homepage Docker network was considered but rejected for the initial deployment because it would introduce a second ingress convention for limited practical benefit while the rest of the homelab remains reachable through published host ports.

## 7. Dashboard information model

The dashboard is organized around operational questions rather than application categories alone.

### 7.1 Infrastructure

| Card | Status source | Native widget | Planned fields |
| --- | --- | --- | --- |
| Server / Netdata | HTTP monitor | Netdata | `warnings`, `criticals` |
| Portainer | Docker | Portainer | `running`, `stopped`, `total` |
| Nginx Proxy Manager | Docker | NPM | `enabled`, `disabled`, `total` |
| AdGuard Home | Docker | AdGuard | `queries`, `blocked`, `filtered`, `latency` |
| NetAlertX | Docker | NetAlertX v2 | `total`, `connected`, `new_devices`, `down_alerts` |

Netdata remains the authoritative destination for detailed host CPU, RAM, disk, network, and UPS monitoring. Homepage should surface alert counts, not duplicate the Netdata dashboard.

### 7.2 Media and libraries

| Card | Status source | Native widget | Planned fields |
| --- | --- | --- | --- |
| Plex | Docker | Plex | `streams`, `movies`, `tv` |
| Immich | Docker | Immich v2 | `users`, `photos`, `videos`, `storage` |
| Audiobookshelf | Docker | Audiobookshelf | `books`, `booksDuration` |
| Kavita | Docker | Kavita | `seriesCount`, `totalFiles` |

The initial Plex integration uses the native Plex widget. Tautulli is not part of the first implementation.

### 7.3 Downloads and discovery

| Card | Status source | Native widget | Planned fields |
| --- | --- | --- | --- |
| qBittorrent | Docker | qBittorrent | `leech`, `download`, `seed`, `upload`; enable progress/size if useful |
| Bitmagnet | Docker + HTTP monitor | none | compact service health only |

Bitmagnet will not receive a custom API widget in the first implementation. Deep resource and database monitoring remains in Netdata and Portainer.

### 7.4 Cloud and applications

| Card | Status source | Native widget | Planned fields |
| --- | --- | --- | --- |
| Nextcloud | Docker | Nextcloud | `activeusers`, `numfiles`, `numshares`, `freespace` |
| ShadowBroker | Docker + HTTP monitor | none | backend container state plus frontend HTTP reachability |

ShadowBroker remains one logical card. Its frontend, backend, and I2P containers must not become separate top-level dashboard cards.

## 8. Status strategy

The dashboard should avoid redundant health signals.

### Services with useful native widgets

Use:

```text
Docker container state + native service widget
```

The Docker state answers whether the selected container is running. The native widget verifies that the application API can respond and supplies useful domain information.

### Services without useful native widgets

Use:

```text
Docker container state + HTTP siteMonitor
```

This applies initially to Bitmagnet and ShadowBroker.

### Host services

Netdata is not treated as a Docker container in this model. Use HTTP reachability plus the native Netdata alert widget.

Homepage `siteMonitor` URLs may differ from the user-facing `href` when an internal endpoint is more appropriate.

## 9. Initial dashboard layout

The planned groups are:

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

Downloads & Discovery
  qBittorrent
  Bitmagnet

Cloud & Applications
  Nextcloud
  ShadowBroker
```

The initial visual policy is compact cards with dot-style status indicators and Docker statistics hidden until explicitly expanded.

## 10. Credential and secret boundaries

Implementation should prefer the least-privileged credential supported by each application.

Planned credential choices:

- Portainer: API key and verified environment ID
- Nginx Proxy Manager: dedicated view-only user
- AdGuard Home: dedicated Homepage account rather than the primary administrator credential where supported by the deployed version
- NetAlertX: API token
- Plex: Plex token
- Immich: API key restricted to `server.statistics`
- Audiobookshelf: API token
- Kavita: API key where practical
- qBittorrent 5.2+: API key rather than WebUI password
- Nextcloud: `NC-Token` rather than the account password

Homepage secrets must not be committed in clear text. Prefer `HOMEPAGE_FILE_*` file-backed substitutions or another equivalent local secret mechanism supported by the deployed Homepage version.

## 11. Components intentionally not added initially

The first implementation must not add these only to enrich the dashboard:

- Glances
- PeaNUT
- Tautulli
- Tailscale API widget
- Uptime Kuma
- custom Bitmagnet API integration
- custom ShadowBroker API integration

These may be reconsidered after the first stable deployment if they solve a demonstrated operational need.

## 12. Rejected alternatives

### Direct Docker socket mount

Rejected. Homepage should not mount `/var/run/docker.sock` directly.

### Dedicated NPM-to-Homepage Docker ingress network

Technically valid and would avoid publishing `3000` on the host, but rejected for the initial deployment in favor of consistency with the current homelab ingress model.

### Serving Homepage directly on both `pirocorp.com` and `home.pirocorp.com`

Rejected. The apex domain redirects to one canonical origin.

### Docker-label-driven Homepage configuration

Rejected. Dashboard configuration remains centralized instead of modifying every existing Compose stack.

### Always-visible Docker resource statistics

Rejected. Resource telemetry should remain available on demand and in Netdata/Portainer rather than dominate the landing page.

### One card per implementation container

Rejected. Database, Redis, worker, and sidecar containers remain implementation details. Portainer is the proper place for full container inventory.

## 13. Decisions intentionally left for implementation

The following must be verified during deployment and are not architecture changes:

- current stable Homepage image/version to pin
- current stable Docker Socket Proxy image/version to pin
- exact host ownership and `PUID`/`PGID` values for the Homepage config directory
- Homepage authentication mode and final authentication settings
- exact Homepage secret-file layout
- Portainer environment ID
- actual AdGuard Home web/API port and widget-compatible URL
- exact internal/native widget URLs for every service
- required firewall rule for the NetAlertX backend/API port used by widget v2
- exact API tokens, keys, service accounts, and account permissions
- whether qBittorrent download progress/size is visually useful after real data is displayed
- final icons, descriptions, group column counts, and minor visual styling

## 14. Implementation guardrails

When implementing this roadmap:

1. Start from this document and preserve every LOCKED decision.
2. Verify current official Homepage and Docker Socket Proxy documentation before generating the final Compose file.
3. Pin explicit image versions rather than using floating `latest` tags in the final deployed state.
4. Do not mount the Docker socket into Homepage.
5. Do not expose the Docker Socket Proxy to the host or LAN.
6. Keep Docker API access read-oriented and disable POST operations.
7. Use the existing host-port/NPM ingress model for Homepage.
8. Use explicit AdGuard rewrites; do not introduce wildcard DNS rewrites.
9. Keep `home.pirocorp.com` canonical and redirect the apex domain to it.
10. Add integrations incrementally and validate each API credential before adding the next one.
11. Do not expose secrets in repository documentation, Compose files, screenshots, or committed YAML.
12. Do not add extra monitoring products unless the existing tools demonstrably cannot supply a required signal.
13. Validate access from both the LAN and a trusted Tailscale client.
14. After successful deployment, update the repository to describe the actual implemented state rather than leaving this roadmap as the only source.

## 15. Acceptance criteria

The initial implementation is accepted when all applicable checks pass:

- [ ] Homepage runs under `/srv/docker/homepage`.
- [ ] Homepage does not mount `/var/run/docker.sock` directly.
- [ ] Docker Socket Proxy is not reachable through a published host port.
- [ ] Docker container state appears in Homepage through the proxy.
- [ ] Docker CPU, RAM, and network statistics can be expanded on demand.
- [ ] `https://home.pirocorp.com` works on the LAN.
- [ ] `https://home.pirocorp.com` works from a trusted Tailscale client using the existing subnet-routing model.
- [ ] `https://pirocorp.com` redirects permanently to `https://home.pirocorp.com`.
- [ ] AdGuard uses explicit rewrites for both names.
- [ ] Nginx Proxy Manager terminates TLS and proxies to `192.168.0.10:3000`.
- [ ] Infrastructure cards show the agreed native widget fields.
- [ ] Media/library cards show the agreed native widget fields.
- [ ] qBittorrent shows useful transfer/seeding information.
- [ ] Bitmagnet and ShadowBroker show compact operational health without unnecessary custom integrations.
- [ ] No committed file contains real service credentials or API secrets.
- [ ] The dashboard remains readable without permanently expanded container statistics.

## 16. Post-implementation documentation transition

After the deployment is validated, create a follow-up documentation PR that:

1. adds `docs/services/homepage/README.md` as the operating/deployment reference;
2. adds Homepage to the implemented service inventory and services index;
3. adds `https://home.pirocorp.com` to the current access URLs;
4. adds `/srv/docker/homepage` to the active Docker stack roots;
5. updates the architecture and current-state documents where needed;
6. records the actual pinned versions, container names, port mappings, secret strategy, and operational commands;
7. removes Homepage from Planned Work; and
8. retains or reclassifies this roadmap as an architecture decision record if it remains useful.

## 17. Official implementation references

Verify these again during implementation because Homepage behavior and supported widget fields can change:

- Homepage installation: https://gethomepage.dev/installation/
- Homepage Docker installation: https://gethomepage.dev/installation/docker/
- Homepage Docker integration: https://gethomepage.dev/configs/docker/
- Homepage services configuration: https://gethomepage.dev/configs/services/
- Homepage settings: https://gethomepage.dev/configs/settings/
- Homepage service widgets: https://gethomepage.dev/widgets/services/
- Docker Socket Proxy: https://github.com/Tecnativa/docker-socket-proxy

## 18. One-paragraph implementation prompt

Implement the approved Homepage architecture in this document on `piroman-server`. Deploy Homepage under `/srv/docker/homepage`, expose Homepage on host port `3000`, and integrate Docker through a dedicated non-published Docker Socket Proxy with POST operations disabled instead of mounting the Docker socket directly into Homepage. Publish the canonical dashboard at `https://home.pirocorp.com` through the existing AdGuard and Nginx Proxy Manager model, and redirect `https://pirocorp.com` permanently to it. Build the dashboard from centralized `services.yaml` configuration using the native widgets and status strategy defined above, keep Docker resource statistics collapsed by default, use least-privileged service credentials and file-backed secrets where supported, and do not introduce extra monitoring products or redesign the existing network. Validate LAN and trusted Tailscale access before updating the repository with the final implemented state.
