# Docker And Portainer

Status: Implemented
Purpose: Document the container runtime conventions used across the homelab.
Depends on: [Base server setup](./base-server-setup.md)
Related docs: [Infrastructure HowTo](./infrastructure-howto.md), [Current state](../overview/current-state.md), [Services index](../services/README.md), [Common commands](../operations/common-commands.md), [Legacy build history](../archive/legacy-root-readme.md)

Use the [Infrastructure HowTo](./infrastructure-howto.md) for the ordered build path. This page records the steady-state runtime conventions after Docker and Portainer are in place.

## Runtime Model

- Docker Engine and Docker Compose are the standard deployment method.
- Each stack lives under `/srv/docker/<app>`.
- Portainer provides centralized visibility and container management.
- Homepage provides a compact cross-service operational view but does not replace Portainer.

## Current Conventions

| Convention | Value |
| --- | --- |
| Stack root | `/srv/docker` |
| Stack layout | one folder per application or service |
| Config persistence | local stack folders and Docker-managed state |
| Portainer URL | `https://portainer.pirocorp.com` |
| Homepage URL | `https://home.pirocorp.com` |

## Current Stack Roots

- `adguard-home`
- `aiostreams`
- `audiobookshelf`
- `bitmagnet`
- `homepage`
- `immich`
- `kavita`
- `netalertx`
- `nextcloud`
- `nginx-proxy-manager`
- `plex`
- `portainer`
- `qbittorrent`
- `shadowbroker`
- `stremio-libtorrent-server`

Keep this list aligned with the live inventory in [Current state](../overview/current-state.md) whenever a deployed stack is added or removed.

## Homepage Docker Integration

Homepage runs under `/srv/docker/homepage` and uses a dedicated Docker Socket Proxy sidecar. Homepage itself does not mount `/var/run/docker.sock`.

Current security model:

```text
homepage
  -> homepage_docker_api (internal Docker network)
  -> homepage-dockerproxy:2375
  -> /var/run/docker.sock:ro
```

The socket proxy is not published on the host or LAN, and `POST=0` disables Docker API mutation through the proxy. Homepage's published host mapping is `192.168.0.10:3002 -> 3000/tcp` because AdGuard Home already occupies host port `3000`.

See [Homepage dashboard](../services/homepage/README.md) for the deployed Compose shape and validation commands.

## Operational Notes

- Prefer pinned image versions for stable services instead of floating `latest` tags.
- Keep compose files, environment files, and persistent stack data grouped together.
- Use Portainer for inspection and Docker CLI for lower-level operations when needed.
- Do not expose the Homepage Docker Socket Proxy port to the host or LAN.

For the original Docker, Portainer, and upgrade walkthroughs, see the [legacy build history](../archive/legacy-root-readme.md).
