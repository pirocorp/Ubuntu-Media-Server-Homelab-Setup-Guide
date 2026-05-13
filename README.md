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
