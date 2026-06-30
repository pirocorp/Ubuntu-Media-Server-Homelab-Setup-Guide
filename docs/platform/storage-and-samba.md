# Storage And Samba

Status: Implemented
Purpose: Describe the current storage layout, mount strategy, and Windows share model.
Depends on: [Base server setup](./base-server-setup.md)
Related docs: [Infrastructure HowTo](./infrastructure-howto.md), [Plex](../services/plex/README.md), [Common commands](../operations/common-commands.md), [Legacy build history](../archive/legacy-root-readme.md)

Use the [Infrastructure HowTo](./infrastructure-howto.md) for the ordered setup sequence. This page summarizes the resulting storage and Samba model after the platform is built.

## Storage Strategy

| Data type | Preferred location |
| --- | --- |
| OS and core host state | NVMe SSD |
| App configs and compose stacks | `/srv/docker/<app>` |
| Media and shared files | mounted drives under `/mnt/*` |
| Large application data | domain-specific mounted drives |

## Current Mounted Domains

| Mount point | Label | Filesystem | Size | Used | Available | Use |
| --- | --- | --- | --- | --- | --- | --- |
| `/mnt/iac` | `IAC` | `ntfs` | `3.7T` | `1.8T` | `1.9T` | `50%` |
| `/mnt/lp` | `LP` | `ntfs` | `7.3T` | `3.7T` | `3.7T` | `51%` |
| `/mnt/data` | `DATA` | `ntfs` | `7.3T` | `3.3T` | `4.1T` | `45%` |
| `/mnt/ia` | `IA` | `ntfs` | `3.7T` | `2.8T` | `894G` | `77%` |
| `/mnt/comp` | `COMP` | `ntfs` | `3.7T` | `1.1T` | `2.7T` | `28%` |

## System Volumes

| Mount point | Filesystem | Size | Used | Available | Use |
| --- | --- | --- | --- | --- | --- |
| `/` | `ext4` | `1.8T` | `125G` | `1.6T` | `8%` |
| `/boot/efi` | `vfat` | `1.1G` | `6.4M` | `1.1G` | `1%` |

## Samba Model

- Samba provides Windows read/write access to the mounted shared storage.
- Stable Linux mount points are used so Docker and Samba reference the same paths.
- Plex and other media-oriented services consume data from the shared mounted storage rather than from ephemeral container state.

## Historical Detail

The original drive detection, `fstab`, NTFS, and Samba walkthrough remains preserved in the [legacy build history](../archive/legacy-root-readme.md).
