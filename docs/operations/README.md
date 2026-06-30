# Operations

Status: Implemented
Purpose: Entry point for shared operational references, command guides, and practical how-to material.
Related docs: [Overview](../overview/README.md), [Services](../services/README.md), [Platform](../platform/README.md)

## Bootstrap And Shared How-To Docs

- [Infrastructure HowTo](../platform/infrastructure-howto.md)
- [Common commands](./common-commands.md)
- [Let's Encrypt public-domain guide](../platform/lets-encrypt-public-domain.md)
- [Docker and Portainer](../platform/docker-and-portainer.md)
- [Storage and Samba](../platform/storage-and-samba.md)
- [Networking and reverse proxy](../platform/networking-and-reverse-proxy.md)

## Service Runbooks

- [Nextcloud update runbook](../services/nextcloud/update-runbook.md)
- [Immich update and backup runbook](../services/immich/update-runbook.md)
- [qBittorrent seedbox runbook](../services/qbittorrent/README.md)
- [Bitmagnet runbook](../services/bitmagnet/README.md)
- [ShadowBroker deployment and operations runbook](../services/shadowbroker/README.md)

## Notes

Use the infrastructure guide first when building or rebuilding the base environment. After the platform exists, use `common-commands.md` and the service runbooks for day-to-day maintenance.

Some simpler services do not have a separate runbook file yet. In those cases, the service `README.md` remains the main operating reference.
