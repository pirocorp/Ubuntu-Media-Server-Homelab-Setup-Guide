# Plex

Status: Implemented
Purpose: Document the Plex deployment, storage mappings, and its relationship to the broader media workflow.
Depends on: [Storage and Samba](../../platform/storage-and-samba.md), [Docker and Portainer](../../platform/docker-and-portainer.md)
Related docs: [qBittorrent](../qbittorrent/README.md), [Service inventory](../../overview/service-inventory.md), [Legacy build history](../../archive/legacy-root-readme.md)

## Access

| Path | Value |
| --- | --- |
| Direct web UI | `http://192.168.0.10:32400/web` |
| Published route | `https://plex.pirocorp.com` |

## Storage Mappings

| Host path | Container path |
| --- | --- |
| `/mnt/data/Plex/Movies` | `/movies` |
| `/mnt/data/Plex/Series` | `/series` |
| `/mnt/data/Plex/Transcode` | `/transcode` |

## Role In The Homelab

- Plex is the media-consumption service for the shared library.
- Media lives on mounted storage, not inside the container filesystem.
- qBittorrent is documented as one of the content-ingest paths into the Plex library.

For the original deployment walkthrough and screenshots, see the [legacy build history](../../archive/legacy-root-readme.md).
