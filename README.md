# Ubuntu Media Server Homelab Setup Guide

## Goal

Build a headless Ubuntu Server homelab/media server with:

- Docker
- Docker Compose
- SSH remote management
- Samba (Windows RW shares)
- Plex
- Immich
- Kavita
- Stash
- bitmagnet
- NTFS media disks
- SSD system drive
- Cockpit web UI
- Portainer
- Lazydocker

## Architecture

```text
NVMe SSD (ext4)
├── Ubuntu Server
├── Docker
├── Docker volumes
├── Databases
├── Plex metadata
└── Configs

NTFS HDDs
├── Movies
├── TV
├── Music
├── Books
├── Photos
└── Downloads
```

## Steps

### Prepare For Ubuntu Server Installation

#### 1. Prepare Bootable USB Flash Drive

- Download the latest [Ubuntu Server LTS](https://ubuntu.com/download/server?utm_source=chatgpt.com)
- USB Writing Tool (Windows) [Rufus](https://rufus.ie/?utm_source=chatgpt.com)

| Setting          | Value          |
| ---------------- | -------------- |
| Ubuntu ISO       | 26.04 LTS      |
| Partition Scheme | GPT            |
| Target System    | UEFI           |
| File System      | FAT32          |
| Write Mode       | ISO Image Mode |


<img width="466" height="582" alt="image" src="https://github.com/user-attachments/assets/d2d3511d-ebbe-4bd3-b59c-d2e1a3e4a662" /> 

<img width="559" height="283" alt="image" src="https://github.com/user-attachments/assets/05beed4f-5617-45cc-ab62-1d3d5ad6cc62" />



#### 2. Create Bootable USB

##### Recommended Settings

- Boot Selection: Ubuntu Server ISO
- Partition Scheme: GPT / if no UEFI support MBR
- File System: Leave default. (FAT32)
- Choose: ISO Mode (recommended)

#### 3. BIOS Preparation

- Enable:
  - UEFI boot
  - AHCI mode for SATA
 
- Disable:
  - Fast Boot (optional)

#### 4. Install Ubuntu Server

- Tutorials, shell commands, Docker/YAML assume US layout

| Setting              | Value                      |
| -------------------- | -------------------------- |
| Layout               | English (US)               |
| Variant              | English (US)               |
| Installation         | Ubuntu Server (minimized)  |

| File System      | FAT32          |
| Write Mode       | ISO Image Mode |







Use:
- Ubuntu Server LTS
- Minimal installation
- Install OpenSSH server

Do NOT install:
- Desktop environment
- Extra unnecessary packages




