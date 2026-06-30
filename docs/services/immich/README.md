# Immich

Status: Implemented
Purpose: Snapshot of the deployed Immich stack until a full runbook is added.
Depends on: [Docker and Portainer](../../platform/docker-and-portainer.md), [Storage and Samba](../../platform/storage-and-samba.md)
Related docs: [Services index](../README.md), [Current state](../../overview/current-state.md)

## Current Deployment Snapshot

| Item | Value |
| --- | --- |
| Stack directory | `/srv/docker/immich` |
| Main containers | `immich_server`, `immich_machine_learning`, `immich_postgres`, `immich_redis` |
| App image | `ghcr.io/immich-app/immich-server:v2.7.5` |
| Direct port | `192.168.0.10:2283 -> 2283` |
| Public hostname | Not yet documented in this repo |

## Notes

- The stack is currently deployed on the host and was verified from the running container inventory.
- Add a dedicated runbook later for reverse proxy, storage, backup, and mobile-client details.
