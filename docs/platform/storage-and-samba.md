# Storage And Samba

Status: Implemented
Purpose: Describe the current storage layout, mount strategy, and Windows share model.
Depends on: [Base server setup](./base-server-setup.md)
Related docs: [Plex](../services/plex/README.md), [Common commands](../operations/common-commands.md), [Legacy build history](../archive/legacy-root-readme.md)

## Storage Strategy

| Data type | Preferred location |
| --- | --- |
| OS and core host state | NVMe SSD |
| App configs and compose stacks | `/srv/docker/<app>` |
| Media and shared files | mounted drives under `/mnt/*` |
| Large application data | domain-specific mounted drives |

## Current Mounted Domains

| Mount point | Purpose |
| --- | --- |
| `/mnt/iac` | domain-oriented shared storage |
| `/mnt/lp` | domain-oriented shared storage |
| `/mnt/data` | main media and shared application data |
| `/mnt/ia` | domain-oriented shared storage |
| `/mnt/comp` | domain-oriented shared storage |

## Samba Model

- Samba provides Windows read/write access to the mounted shared storage.
- Stable Linux mount points are used so Docker and Samba reference the same paths.
- Plex and other media-oriented services consume data from the shared mounted storage rather than from ephemeral container state.

## Historical Detail

The original drive detection, `fstab`, NTFS, and Samba walkthrough remains preserved in the [legacy build history](../archive/legacy-root-readme.md).
