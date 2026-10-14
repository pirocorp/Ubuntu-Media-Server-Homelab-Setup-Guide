# Services

Status: Implemented
Purpose: Entry point for deployed application documentation, service overviews, and service-specific runbooks.
Related docs: [Overview](../overview/README.md), [Platform](../platform/README.md), [Operations](../operations/README.md), [Roadmaps](../roadmaps/README.md)

These service docs assume the base environment from the [Infrastructure HowTo](../platform/infrastructure-howto.md) is already in place.

## Implemented Services

The original [phase-1 decision record](../roadmaps/server-health-dashboard/README.md) is retained for history. The [Server Health runbook](./server-health/README.md) is now the source of truth for the deployed implementation.

- [Homepage](./homepage/README.md)
- [Plex](./plex/README.md)
- [Nextcloud](./nextcloud/README.md)
- [qBittorrent](./qbittorrent/README.md)
- [Bitmagnet](./bitmagnet/README.md)
- [Audiobookshelf](./audiobookshelf/README.md)
- [Immich](./immich/README.md)
- [Kavita](./kavita/README.md)
- [UPS monitoring / host NUT](./ups-monitoring/README.md)
- [Server Health Dashboard (Glances + PeaNUT)](./server-health/README.md)
- [ShadowBroker](./shadowbroker/README.md)
- [NetAlertX](./netalertx/README.md)
- [AIOStreams](./aiostreams/README.md)
- [stremio-libtorrent-server](./stremio-libtorrent-server/README.md)

## Runbooks And How-To Docs

- [Homepage deployment and operations runbook](./homepage/README.md)
- [Server Health Dashboard — reproducing the deployed phase-1 configuration](./server-health/README.md)
- [Nextcloud update runbook](./nextcloud/update-runbook.md)
- [Immich update and backup runbook](./immich/update-runbook.md)
- [qBittorrent seedbox runbook](./qbittorrent/README.md)
- [Bitmagnet runbook](./bitmagnet/README.md)
- [ShadowBroker deployment and operations runbook](./shadowbroker/README.md)
- [NetAlertX deployment and operations runbook](./netalertx/README.md)
- [AIOStreams deployment and operations runbook](./aiostreams/README.md)
- [stremio-libtorrent-server deployment and operations runbook](./stremio-libtorrent-server/README.md)
- [Stremio Web UI publishing through AdGuard and Nginx Proxy Manager](./stremio-libtorrent-server/web-ui-publishing.md)

## Service READMEs Used As Main How-To

These services currently use their main `README.md` as the primary operating reference:

- [Homepage](./homepage/README.md)
- [Plex](./plex/README.md)
- [Audiobookshelf](./audiobookshelf/README.md)
- [Kavita](./kavita/README.md)
- [UPS monitoring](./ups-monitoring/README.md)
- [Server Health Dashboard](./server-health/README.md)
- [NetAlertX](./netalertx/README.md)
- [AIOStreams](./aiostreams/README.md)
- [stremio-libtorrent-server](./stremio-libtorrent-server/README.md)

## Planned Work

- [Homepage Disk Health — phase-2 SMART roadmap](../roadmaps/server-health-dashboard/smart-extension.md) — separate future SMART card; phase-1 Glances + PeaNUT deployment completed and Netdata retired.
- [OmniMeter local energy monitoring roadmap](../roadmaps/omnimeter/README.md) — candidate under evaluation; not deployed.
- [Taiwan–Russia Risk Watch Discord Bridge](../roadmaps/riskwatch-discord-bridge/README.md)
- [AIOStreams multi-source P2P expansion](../roadmaps/stremio-aiostreams/multi-source-p2p-expansion.md)
- [Usenet roadmap](../roadmaps/usenet/README.md)
- [ShadowBroker OpenClaw integration roadmap](../roadmaps/shadowbroker-openclaw-integration.md)

The Homepage dashboard baseline is deployed; its original architecture roadmap is retained as a decision record. The original Stremio/AIOStreams seven-phase implementation baseline is complete, including the explicit `6882/TCP+UDP` router-forward follow-up. Remote access is also implemented as a platform capability. Use the [Tailscale remote access runbook](../operations/tailscale-remote-access-runbook.md) for operations.
