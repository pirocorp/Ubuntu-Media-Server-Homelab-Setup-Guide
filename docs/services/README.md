# Services

Status: Implemented
Purpose: Entry point for deployed application documentation, service overviews, and service-specific runbooks.
Related docs: [Overview](../overview/README.md), [Platform](../platform/README.md), [Operations](../operations/README.md), [Roadmaps](../roadmaps/README.md)

These service docs assume the base environment from the [Infrastructure HowTo](../platform/infrastructure-howto.md) is already in place.

## Implemented Services

- [Plex](./plex/README.md)
- [Nextcloud](./nextcloud/README.md)
- [qBittorrent](./qbittorrent/README.md)
- [Bitmagnet](./bitmagnet/README.md)
- [Audiobookshelf](./audiobookshelf/README.md)
- [Immich](./immich/README.md)
- [Kavita](./kavita/README.md)
- [UPS monitoring](./ups-monitoring/README.md)
- [ShadowBroker](./shadowbroker/README.md)
- [NetAlertX](./netalertx/README.md)
- [AIOStreams](./aiostreams/README.md)
- [stremio-libtorrent-server](./stremio-libtorrent-server/README.md)

## Runbooks And How-To Docs

- [Nextcloud update runbook](./nextcloud/update-runbook.md)
- [Immich update and backup runbook](./immich/update-runbook.md)
- [qBittorrent seedbox runbook](./qbittorrent/README.md)
- [Bitmagnet runbook](./bitmagnet/README.md)
- [ShadowBroker deployment and operations runbook](./shadowbroker/README.md)
- [NetAlertX deployment and operations runbook](./netalertx/README.md)
- [AIOStreams deployment and operations runbook](./aiostreams/README.md)
- [stremio-libtorrent-server deployment and operations runbook](./stremio-libtorrent-server/README.md)

## Service READMEs Used As Main How-To

These services currently use their main `README.md` as the primary operating reference:

- [Plex](./plex/README.md)
- [Audiobookshelf](./audiobookshelf/README.md)
- [Kavita](./kavita/README.md)
- [UPS monitoring](./ups-monitoring/README.md)
- [NetAlertX](./netalertx/README.md)
- [AIOStreams](./aiostreams/README.md)
- [stremio-libtorrent-server](./stremio-libtorrent-server/README.md)

## Planned Work

- [Stremio + AIOStreams remaining implementation phases](../roadmaps/stremio-aiostreams/README.md)
- [Usenet roadmap](../roadmaps/usenet/README.md)
- [ShadowBroker OpenClaw integration roadmap](../roadmaps/shadowbroker-openclaw-integration.md)

Remote access is now implemented as a platform capability. Use the [Tailscale remote access runbook](../operations/tailscale-remote-access-runbook.md) for operations.
