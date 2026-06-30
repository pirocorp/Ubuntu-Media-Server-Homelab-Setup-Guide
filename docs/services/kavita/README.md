# Kavita

Status: Implemented
Purpose: Snapshot of the deployed Kavita stack until a full runbook is added.
Depends on: [Docker and Portainer](../../platform/docker-and-portainer.md), [Storage and Samba](../../platform/storage-and-samba.md)
Related docs: [Services index](../README.md), [Current state](../../overview/current-state.md)

## Current Deployment Snapshot

| Item | Value |
| --- | --- |
| Stack directory | `/srv/docker/kavita` |
| Main container | `kavita` |
| Image | `jvmilazzo/kavita:0.9.0.2` |
| Direct port | `192.168.0.10:5000 -> 5000` |
| Public hostname | Not yet documented in this repo |

## Notes

- The stack is currently deployed on the host and was verified from the running container inventory.
- Add a dedicated runbook later for library paths, user access, and reverse proxy details.
