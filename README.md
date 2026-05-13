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

#### 2. Create Bootable USB

##### Recommended Settings

- Boot Selection: Ubuntu Server ISO
- Partition Scheme: GPT / if no UEFI support MBR
- File System: Leave default. (FAT32)
- Choose: ISO Mode (recommended)

| Setting          | Value          |
| ---------------- | -------------- |
| Ubuntu ISO       | 26.04 LTS      |
| Partition Scheme | GPT            |
| Target System    | UEFI           |
| File System      | FAT32          |
| Write Mode       | ISO Image Mode |


<img width="466" height="582" alt="image" src="https://github.com/user-attachments/assets/d2d3511d-ebbe-4bd3-b59c-d2e1a3e4a662" /> 

<img width="559" height="283" alt="image" src="https://github.com/user-attachments/assets/05beed4f-5617-45cc-ab62-1d3d5ad6cc62" />

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
| Proxy                | Blank                      |
| Use entire disk      | Selected                   |
| Mirror               | Default                    |
| LVM                  | Disable LVM                |
| Encryption (LUKS)    | Leave OFF                  |


##### Final Disk Layout

| Mount       | Type  | Size    |
| ----------- | ----- | ------- |
| `/`         | ext4  | ~1.8 TB |
| `/boot/efi` | FAT32 | 1 GB    |

##### Linux host

| Field       | Value                       |
| ----------- | --------------------------- |
| Your name   | `Piroman`                   |
| Server name | `piroman-server`            |
| Username    | `piroman`                   |

##### ENABLE OpenSSH server

That enables:

- Remote terminal access
- Windows SSH access
- Future Docker management
- Cockpit/Portainer workflow

##### Ubuntu PRO - Skip it

##### SSH Configuration

| Option                                 | Status |
| -------------------------------------- | ------ |
| Install OpenSSH server                 | ✅     |
| Allow password authentication over SSH | ✅     |
| Import SSH key                         |        |


### Connect through SSH to Ubuntu Server

```bash
ssh piroman@192.168.0.246
```

<img width="1115" height="628" alt="image" src="https://github.com/user-attachments/assets/24cd03b0-5b50-4ffa-8c18-b05d2e1409d8" />

### Initial System Update

```bash
sudo apt update
sudo apt upgrade -y
```

<img width="1115" height="628" alt="image" src="https://github.com/user-attachments/assets/c01112ed-4c05-47b1-9a39-e4108a667327" />

### Remove no longer required packages

```bash
sudo apt autoremove -y
```

<img width="1115" height="628" alt="image" src="https://github.com/user-attachments/assets/1534a926-455f-4f37-88da-b1df2ba8b0cf" />


### Install Essential Packages

```bash
sudo apt install -y \
curl \
git \
htop \
btop \
tmux \
nano \
ncdu \
ufw \
smartmontools \
ca-certificates \
gnupg \
samba \
ntfs-3g \
cockpit
```

| Package           | Purpose                            | Why You Need It                                  |
| ----------------- | ---------------------------------- | ------------------------------------------------ |
| `curl`            | Command-line downloader/API client | Download scripts, test APIs, fetch files         |
| `git`             | Version control system             | Clone GitHub repos and manage configs            |
| `htop`            | Interactive process monitor        | View CPU/RAM/processes easily                    |
| `btop`            | Advanced terminal resource monitor | Beautiful real-time monitoring dashboard         |
| `tmux`            | Persistent terminal sessions       | Keep sessions running after SSH disconnects      |
| `nano`            | Terminal text editor               | Easy editing of config files                     |
| `ncdu`            | Disk usage analyzer                | Find large folders/files quickly                 |
| `ufw`             | Simple firewall manager            | Secure the server with manageable firewall rules |
| `smartmontools`   | Disk health monitoring             | Check SSD/HDD SMART health status                |
| `ca-certificates` | SSL certificate bundle             | Required for secure HTTPS connections            |
| `gnupg`           | GPG key management                 | Needed for trusted repositories like Docker      |
| `samba`           | Windows file sharing               | Share folders between Ubuntu and Windows         |
| `ntfs-3g`         | NTFS filesystem support            | Read/write Windows NTFS drives                   |
| `cockpit`         | Web management interface           | Manage server from browser                       |

<img width="1112" height="628" alt="image" src="https://github.com/user-attachments/assets/b4360f30-6322-4629-8daa-4aed9f9616b7" />


