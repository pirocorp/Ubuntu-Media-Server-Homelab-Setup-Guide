# Docker And Portainer

Status: Implemented
Purpose: Document the container runtime conventions used across the homelab.
Depends on: [Base server setup](./base-server-setup.md)
Related docs: [Infrastructure HowTo](./infrastructure-howto.md), [Services index](../services/README.md), [Common commands](../operations/common-commands.md), [Legacy build history](../archive/legacy-root-readme.md)

Use the [Infrastructure HowTo](./infrastructure-howto.md) for the ordered build path. This page records the steady-state runtime conventions after Docker and Portainer are in place.

## Runtime Model

- Docker Engine and Docker Compose are the standard deployment method.
- Each stack lives under `/srv/docker/<app>`.
- Portainer provides centralized visibility and container management.

## Current Conventions

| Convention | Value |
| --- | --- |
| Stack root | `/srv/docker` |
| Stack layout | one folder per application or service |
| Config persistence | local stack folders and Docker-managed state |
| Portainer URL | `https://portainer.pirocorp.com` |

## Current Stack Roots

- `adguard-home`
- `audiobookshelf`
- `bitmagnet`
- `immich`
- `kavita`
- `nextcloud`
- `nginx-proxy-manager`
- `plex`
- `portainer`
- `qbittorrent`
- `shadowbroker`

## Operational Notes

- Prefer pinned image versions for stable services instead of floating `latest` tags.
- Keep compose files, environment files, and persistent stack data grouped together.
- Use Portainer for inspection and Docker CLI for lower-level operations when needed.

For the original Docker, Portainer, and upgrade walkthroughs, see the [legacy build history](../archive/legacy-root-readme.md).
