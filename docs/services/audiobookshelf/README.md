# Audiobookshelf

Status: Implemented
Purpose: Snapshot of the deployed Audiobookshelf stack until a full runbook is added.
Depends on: [Docker and Portainer](../../platform/docker-and-portainer.md), [Storage and Samba](../../platform/storage-and-samba.md)
Related docs: [Services index](../README.md), [Current state](../../overview/current-state.md)

## Current Deployment Snapshot

| Item | Value |
| --- | --- |
| Stack directory | `/srv/docker/audiobookshelf` |
| Main container | `audiobookshelf` |
| Image | `ghcr.io/advplyr/audiobookshelf:2.35.1` |
| Direct port | `192.168.0.10:13378 -> 80` |
| Public hostname | Not yet documented in this repo |

## Notes

- The stack is currently deployed on the host and was verified from the running container inventory.
- Add a dedicated runbook later for storage mappings, reverse proxy settings, and client access details.
