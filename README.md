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

## Applications, Services, Media Persistence storage

| Purpose       | Storage Type   |
| ------------- | -------------- |
| App configs   | Docker volumes |
| Media library | Bind mounts    |
| Photos/videos | Bind mounts    |
| Databases     | Docker volumes |

## Storage Strategy

| Data Type   | Recommended              |
| ----------- | ------------------------ |
| App configs | `/srv/docker/appname`    |
| Media       | `/mnt/media`             |
| Downloads   | `/mnt/downloads`         |
| Databases   | `/srv/docker/appname/db` |
| Photos      | `/mnt/media/photos`      |


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


### Configure Firewall

#### First check current firewall status:

```bash
sudo ufw status
```

<img width="1112" height="628" alt="image" src="https://github.com/user-attachments/assets/7fc7dd47-25cf-424d-8254-879f99d07902" />

#### Run the following commands to enable those services

```bash
sudo ufw allow OpenSSH
sudo ufw allow Samba
sudo ufw allow 9090/tcp
```

| Rule       | Purpose                     |
| ---------- | --------------------------- |
| `OpenSSH`  | Allows SSH remote access    |
| `Samba`    | Allows Windows file sharing |
| `9090/tcp` | Allows Cockpit web UI       |


<img width="1112" height="628" alt="image" src="https://github.com/user-attachments/assets/b3d13ae2-676a-4253-9200-41d75923e4a6" />

#### Enable firewall

```bash
sudo ufw enable
```

<img width="1112" height="628" alt="image" src="https://github.com/user-attachments/assets/9c754d4d-0f3a-4f1f-a7d1-a827e56c52bc" />

#### Verify firewall status

```bash
sudo ufw status verbose
```

<img width="1112" height="628" alt="image" src="https://github.com/user-attachments/assets/fe9f6f88-9f57-459c-ab48-c532b56e9367" />

Current state:

| Service          | Status             |
| ---------------- | ------------------ |
| SSH              | Allowed            |
| Samba            | Allowed            |
| Cockpit (9090)   | Allowed            |
| Incoming traffic | Blocked by default |
| Outgoing traffic | Allowed            |


#### Cockpit web management

Open this in your Windows browser: https://192.168.0.246:9090

<img width="962" height="1032" alt="image" src="https://github.com/user-attachments/assets/c32aee7a-251c-4754-9536-d76f80eea52d" />

<img width="948" height="867" alt="image" src="https://github.com/user-attachments/assets/86b04fd6-2e1e-4142-a70d-15c4a0a26f45" />


#### Turn on administrative access in Cockpit

<img width="937" height="131" alt="image" src="https://github.com/user-attachments/assets/c7b22f6a-6df2-4f35-9ce2-56689083250e" />

<img width="951" height="727" alt="image" src="https://github.com/user-attachments/assets/997d3e22-b845-413a-a8a0-f678afd82fb5" />


#### Install Docker

```bash
# Install Docker Engine + Docker Compose
sudo apt install docker.io docker-compose-v2 -y

# Enable Docker at startup - Linux equivalent of Windows Services + Startup Type
sudo systemctl enable docker

# Start Docker now
sudo systemctl start docker

# Verify Docker works
sudo docker run hello-world

# Allow your user to run Docker without sudo
sudo usermod -aG docker piroman
```

| Package / Component          | Purpose                       | Why You Need It                                                          |
| ---------------------------- | ----------------------------- | ------------------------------------------------------------------------ |
| `docker.io`                  | Docker Engine                 | Runs containers (Plex, Immich, Kavita, Portainer, etc.)                  |
| `docker-compose-v2`          | Docker Compose plugin         | Lets you define and start multi-container apps using `compose.yml` files |
| `systemctl enable docker`    | Startup service configuration | Makes Docker automatically start after every reboot                      |
| `systemctl start docker`     | Starts Docker service now     | Immediately launches the Docker daemon without rebooting                 |
| `hello-world` container      | Docker test image             | Confirms Docker is installed and working correctly                       |
| `usermod -aG docker piroman` | Adds user to Docker group     | Allows `piroman` to use Docker without typing `sudo` every time          |

<img width="1115" height="818" alt="image" src="https://github.com/user-attachments/assets/a7e294b4-ffa8-4a46-82cd-94e38ec98a94" />

Log out of SSH and reconnect. After reconnecting, test:

```bash
docker ps
```
If no permission error appears, Docker is configured correctly.

<img width="1115" height="647" alt="image" src="https://github.com/user-attachments/assets/ab14dec1-56b7-4c3a-8291-355d3239f002" />


### Important Concept

On Linux, almost everything important is a service. Examples: Docker, SSH, Samba, Cockpit, Plex, Databases. And all are managed similarly with:

```bash
systemctl
```


### Install Portainer

Portainer becomes your graphical Docker management UI. You’ll be able to:

- Start/Stop Containers
- View logs
- Manage stacks
- Update containers
- Inspect networks/volumes
- Deploy compose files visually

#### Create Docker-managed persistent storage volume for Portainer

A Docker-managed persistent storage volume is permanent storage that is separate from the container itself. It keeps application data safe even if containers are updated, recreated, restarted, or deleted, ensuring settings, databases, and configurations persist independently of the running container.

A Docker container is the running application instance itself — temporary and replaceable — while a Docker volume is persistent storage that holds the application’s important data, such as configurations, databases, and files. Containers can be recreated at any time, but volumes preserve the data independently, so nothing important is lost.

Docker stores volumes here by default: `/var/lib/docker/volumes/`, Portainer volume specifically: `/var/lib/docker/volumes/portainer_data/`.

1. Create Portainer Directory

```bash
sudo mkdir -p /srv/docker/portainer
```

<img width="1115" height="647" alt="image" src="https://github.com/user-attachments/assets/a62871b2-2581-407a-8e93-fc877918ab66" />


2. Give Your User Ownership

```bash
sudo chown -R piroman:piroman /srv/docker
```

<img width="1115" height="647" alt="image" src="https://github.com/user-attachments/assets/ff6a1e84-c38e-4af2-ad03-58a4b8b899be" />

This allows Docker containers and your user to manage files cleanly.

3. Create Compose File

```bash
nano /srv/docker/portainer/compose.yml
```

```yaml
services:
  portainer:
    image: portainer/portainer-ce:lts
    container_name: portainer
    restart: unless-stopped

    ports:
      - "8000:8000"
      - "9443:9443"

    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /srv/docker/portainer/data:/data
```

What Happens After Reboot

```mermaid
flowchart TD
    A[Ubuntu boots] --> B[Docker service starts]
    B --> C[Docker reads restart policies]
    C --> D[Portainer container starts automatically]
```

<img width="1115" height="647" alt="image" src="https://github.com/user-attachments/assets/1940dcbc-9771-447d-b17d-49511ae7efd7" />


4. Check and validate the YAML
   
```bash
cat /srv/docker/portainer/compose.yml
docker compose -f /srv/docker/portainer/compose.yml config
```

<img width="1115" height="647" alt="image" src="https://github.com/user-attachments/assets/938e41f5-c3f6-4b5f-9e78-6b52695dce17" />

<img width="1115" height="799" alt="image" src="https://github.com/user-attachments/assets/cdecaa30-c7d5-43c8-b8d2-15cc8e3239bd" />

4. Run the Docker container

```bash
cd /srv/docker/portainer
docker compose up -d
```

<img width="1115" height="799" alt="image" src="https://github.com/user-attachments/assets/44737ddc-4841-469a-bf7f-055fda1466b3" />


5. Then, Verify Running

```bash
docker ps
```

<img width="1115" height="799" alt="image" src="https://github.com/user-attachments/assets/20106668-53c0-44ec-8afa-20825f71f1c8" />

Open the following URI: https://192.168.0.246:9443

If you see that your instance timed out, just restart the container.

<img width="1055" height="250" alt="image" src="https://github.com/user-attachments/assets/318efe57-94cd-4b10-b780-01b17f64595e" />

Why This Happens. On first boot:

```mermaid
flowchart TD
    A[Portainer starts first time] --> B[Waits for admin creation]
    B --> C[Timeout expires]
    C --> D[Locks setup for security]
    D --> E[Requires restart]
```

**Portainer waits for the initial admin setup for only a limited time.**

5a. Restart the container.

```bash
cd /srv/docker/portainer
docker compose restart
```

<img width="1117" height="151" alt="image" src="https://github.com/user-attachments/assets/81bd6c0c-f53d-4946-af04-2f45ee57742f" />

Then refresh: https://192.168.0.246:9443 and create an admin account

<img width="1307" height="863" alt="image" src="https://github.com/user-attachments/assets/06bbe7e7-7ac4-45c1-83c8-3200467eeb4c" />


6. Connect Portainer to your local Docker environment

Clicking “Get Started” connects Portainer to your local Docker environment, allowing it to manage the Docker Engine running on the server. Once connected, Portainer provides a centralized web interface for viewing and managing containers, images, networks, volumes, and Compose stacks without using terminal commands for every operation.


<img width="1909" height="589" alt="image" src="https://github.com/user-attachments/assets/2cdc669e-1e0d-4f2f-8b53-85b6b39dfd10" />


### Current Infrastructure

| Component              | Status  |
| ---------------------- | ------  |
| Ubuntu Server          | ✅      |
| SSH                    | ✅      |
| UFW Firewall           | ✅      |
| Cockpit                | ✅      |
| Docker                 | ✅      |
| Docker Compose         | ✅      |
| Portainer              | ✅      |
| Infrastructure-as-Code | ✅      |


### Docker folder architecture

Before deploying containers, it is recommended to create a standardized Docker folder architecture. Organizing applications into dedicated directories provides predictable file paths, simplifies maintenance, and keeps infrastructure clean and scalable. 

This approach makes backups easier because each application’s configuration and persistent data are stored in known locations. It also simplifies migrations to another server, since entire application folders can be copied directly. In addition, structured directories lead to cleaner Docker Compose files and promote infrastructure consistency across all deployed services, which becomes increasingly important as the homelab grows.


#### Structure

```
/srv/docker
├── portainer
├── plex
├── immich
├── kavita
├── stash
├── adguard
└── nginx-proxy-manager
```

Each application should have its own isolated directory containing its Docker Compose file, configuration files, and persistent data. This structure keeps services separate, making management, troubleshooting, backups, and upgrades significantly easier. By following the principle of “one project = one directory,” every application becomes self-contained and portable, keeping the infrastructure organized, scalable, and easier to maintain over time.


##### Create Base Structure

```bash
mkdir -p /srv/docker/{plex,immich,kavita,adguard,nginx-proxy-manager}
```

##### Verify Structure

1. Install `tree` if missing:

```bash
sudo apt install tree -y
```

<img width="1115" height="799" alt="image" src="https://github.com/user-attachments/assets/49e00e5b-4f79-47db-a339-ab964a52b1b4" />

2. Execute `three` Command 

```bash
tree /srv/docker
```

<img width="1115" height="799" alt="image" src="https://github.com/user-attachments/assets/c6e55822-037c-487f-a5c6-9ab97eb09447" />

##### Why This Happens

Portainer container writes some internal application data with elevated permissions for security and isolation. If You Want To Inspect Everything

```bash
sudo tree /srv/docker
```

This temporarily grants root privileges and allows viewing all directories.

<img width="1115" height="799" alt="image" src="https://github.com/user-attachments/assets/10bf3b26-4fe5-446b-ac4a-066152de9bca" />

### Important Concept - Linux file permissions and Docker

Linux file permissions are an important part of container isolation in Docker environments. Containers often create files and directories owned by the root user or by internal service accounts to protect application data and maintain security boundaries between services. As a result, normal users may not always have permission to access certain container-managed files directly, which is expected behavior and helps prevent accidental modification of critical application data.

```mermaid
flowchart TD
    A[Docker Container] --> B[Creates files]
    B --> C[Files owned by root]
    C --> D[Normal user may not access]
```

### Important Concept - Why Floating Tags Can Be Risky

Using floating Docker image tags such as `latest`, `lts`, or `stable` can be risky because they do not point to a fixed application version. Over time, these tags may automatically reference newer releases containing configuration changes, database migrations, removed features, or breaking changes. As a result, running updates or redeploying containers can unexpectedly alter a working environment without any explicit version change in the Compose file. Pinning exact versions provides predictable, reproducible, and more stable infrastructure management.

#### Typical Upgrade Workflow

```mermaid
flowchart TD
    A[Read release notes] --> B[Update compose.yml version]
    B --> C[docker compose pull]
    C --> D[docker compose up -d]
    D --> E[Test]
```

#### Pin the current LTS version

According to the screenshot, the current Portainer LTS line is: **2.39.2 LTS**

<img width="1186" height="358" alt="image" src="https://github.com/user-attachments/assets/219baa40-a3be-48d4-8660-f25f0c0d9a9e" />

So instead of: `image: portainer/portainer-ce:lts` it should be pinned: `image: portainer/portainer-ce:2.39.2` or even `image: portainer/portainer-ce:2.39.2-alpine` if you intentionally want the Alpine variant.

1. Pin the correct version in the YAML file

<img width="1115" height="495" alt="image" src="https://github.com/user-attachments/assets/d15c9e18-c1c0-4bd8-95b7-09fa523a8ab7" />

2. Restart the container

```bash
cd /srv/docker/portainer
docker compose up -d
```

<img width="1115" height="495" alt="image" src="https://github.com/user-attachments/assets/b4573bde-2759-4106-b9f9-f33ba7fb3f75" />


### SSH key authentication

Switching from password-based SSH authentication to SSH keys significantly improves security and convenience. SSH keys are resistant to brute-force attacks, cannot be guessed like passwords, and allow secure authentication without sending your password over the network. They are the standard authentication method used in professional Linux, cloud, and DevOps environments.

#### Goal

```mermaid
flowchart TD
    A[Windows PC] --> B[SSH Private Key]
    B --> C[Ubuntu Server]
    C --> D[Authorized Public Key]
```


1. Step 1 — Generate SSH Key on Windows

```powershell
ssh-keygen -t ed25519 -C "piroman-windows"
```

<img width="1103" height="514" alt="image" src="https://github.com/user-attachments/assets/f0b0828d-2e59-4a46-97e7-3f201860e725" />


`C:\Users\YOUR_USER\.ssh\`

<img width="740" height="235" alt="image" src="https://github.com/user-attachments/assets/94606d7d-01fd-460a-9685-ac855935513f" />

2. Step 2 — Copy the public key

In Windows PowerShell, execute the following command and copy the full output 

```powershell
type $env:USERPROFILE\.ssh\ubuntu-server.pub
```

Then in Ubuntu 

```bash
# Create the .ssh directory if it does not already exist
mkdir -p ~/.ssh

# Set secure permissions so only the current user can access the directory
chmod 700 ~/.ssh

# Open the authorized_keys file to add allowed public SSH keys
nano ~/.ssh/authorized_keys
```

Then paste the output that was copied from Windows into the Ubuntu Server `~/.ssh/authorized_keys` file

3. Limit access to that file only for the current user. SSH requires strict permissions for security.

```bash
chmod 600 ~/.ssh/authorized_keys
```

4. Test SSH Keys

From Windows PowerShell, execute the following command. Now, you should connect without a Linux account password, possibly with an SSH key passphrase if you set one during key creation

```bash
ssh -i $env:USERPROFILE\.ssh\ubuntu-server piroman@192.168.0.246
```

<img width="1060" height="514" alt="image" src="https://github.com/user-attachments/assets/d759eb5a-4ae6-48da-abe2-322d8911edcf" />


#### What Happens During Login Now

```mermaid
flowchart TD
    A[Windows Private Key] --> B[SSH Authentication]
    B --> C[Ubuntu checks authorized_keys]
    C --> D[Access granted]
```

#### Important Security Difference

| Old Method                       | New Method                       |
| -------------------------------- | -------------------------------- |
| Password sent for authentication | Cryptographic key authentication |
| Vulnerable to brute force        | Much more secure                 |
| Manual password typing           | Automatic authentication         |
| Common beginner setup            | Professional standard            |


### Implement Storage Architecture

```mermaid
flowchart TD
    A[Detect NTFS Drives] --> B[Create Mount Structure]
    B --> C[Configure Persistent Mounts]
    C --> D[Set Permissions]
    D --> E[Configure Samba]
    E --> F[Validate Windows Access]
    F --> G[Use In Docker Containers]
```

#### Domain-oriented storage architecture

- Each physical drive has a defined responsibility
- Each mount point represents a storage domain
- Applications consume storage through stable paths

```mermaid
flowchart TD
    A[Storage Domain] --> B[Independent Mount Point]
    B --> C[Independent Backup Strategy]
    B --> D[Independent Docker Bind Mounts]
    B --> E[Independent Samba Share]
```


#### Goal

```
Physical Drive
    ↓
UUID
    ↓
Mount Point
    ↓
Persistent Mount
    ↓
Samba Share
    ↓
Docker Bind Mount
```

1. Step 1 — Connect Your External Drives

Shut down the server and connect the external drives.

```bash
sudo shutdown now
```

2. Step 2 — Detect Drives

```bash
lsblk -f
``` 

<img width="1060" height="514" alt="image" src="https://github.com/user-attachments/assets/69f50912-9331-4b67-a141-1776a46493ed" />

##### Current Drive Mapping

| Domain      | Device | Label  | Filesystem |
| ----------- | ------ | ------ | ---------- |
| `/mnt/iac`  | `sda2` | `IAC`  | NTFS       |
| `/mnt/lp`   | `sdb2` | `LP`   | NTFS       |
| `/mnt/data` | `sdc2` | `DATA` | NTFS       |
| `/mnt/ia`   | `sdd2` | `IA`   | NTFS       |
| `/mnt/comp` | `sdf2` | `COMP` | NTFS       |


3. Step 3 — Create Mount Points

```bash
# Create mount points for domain-oriented storage drives
sudo mkdir -p /mnt/iac
sudo mkdir -p /mnt/lp
sudo mkdir -p /mnt/data
sudo mkdir -p /mnt/ia
sudo mkdir -p /mnt/comp
```
<img width="1060" height="514" alt="image" src="https://github.com/user-attachments/assets/e63063f6-f62f-4169-9b02-9943d2aa4bb4" />


4. Step 4 — Install NTFS Driver Support

```bash
sudo apt install ntfs-3g -y
```

<img width="1060" height="514" alt="image" src="https://github.com/user-attachments/assets/debb323b-c214-4c9e-927e-d4469405d6b7" />


5. Step 5 — Configure Persistent Mounts

Persistent mounts allow Linux to automatically mount storage drives during every system boot without requiring manual intervention. This is configured through the `/etc/fstab` file, where each drive is identified by its UUID — a stable, unique identifier that does not change between reboots. Using persistent mounts ensures that all storage domains are immediately available to Docker containers, Samba shares, media servers, and other services after the server starts.

```bash
sudo nano /etc/fstab
```

Current fstab

The current `/etc/fstab` file contains the default filesystem configuration created during Ubuntu installation. It defines the main Linux system partition (/), the EFI boot partition (/boot/efi), and the swap file used for virtual memory. These entries ensure the operating system and boot components are automatically mounted and available during every system startup.

The /etc/fstab file defines which storage devices Linux should automatically mount during boot. By adding our NTFS drives to fstab, we created persistent mounts that survive reboots and always appear in the same locations, such as /mnt/iac, /mnt/lp, /mnt/data, /mnt/ia, and /mnt/comp. Each entry uses the drive UUID instead of device names like /dev/sda2, making the configuration more reliable even if drive ordering changes.

<img width="1060" height="514" alt="image" src="https://github.com/user-attachments/assets/c34a4d13-0433-4344-8da5-a6bb5a3afe88" />

Extended fstab

The extended `/etc/fstab` configuration now includes persistent NTFS domain drive mounts, allowing all storage drives to be automatically mounted and consistently available after every system reboot.

<img width="1060" height="666" alt="image" src="https://github.com/user-attachments/assets/938d54be-d3c0-40c5-87ce-2749b61af540" />

| Mount Point | UUID in `lsblk -f` | UUID in `fstab`    | Status |
| ----------- | ------------------ | ------------------ | ------ |
| `/mnt/iac`  | `6A3AD04D3AD017C1` | `6A3AD04D3AD017C1` | ✅      |
| `/mnt/lp`   | `8CEEDEBAEEDE9BB0` | `8CEEDEBAEEDE9BB0` | ✅      |
| `/mnt/data` | `A4D09E95D09E6D74` | `A4D09E95D09E6D74` | ✅      |
| `/mnt/ia`   | `7698C86698C82689` | `7698C86698C82689` | ✅      |
| `/mnt/comp` | `D6E66FCAE66FA987` | `D6E66FCAE66FA987` | ✅      |

Initially, the NTFS drives were mounted with `umask=0022`, which made Samba shares appear read-only to Windows. This was solved by changing the mount option to `umask=0000`, which allows full read/write access through Samba while still mapping ownership to the piroman Linux user (`uid=1000`, `gid=1000`). After reloading the mounts and restarting Samba, the shares became fully writable from Windows.

| Property   | Example Value | Purpose                                                                                        |
| ---------- | ------------- | ---------------------------------------------------------------------------------------------- |
| `uid`      | `1000`        | Sets the Linux owner user ID for all files on the NTFS drive                                   |
| `gid`      | `1000`        | Sets the Linux group ID for all files on the NTFS drive                                        |
| `umask`    | `0022`        | Removes write permissions for group/others, resulting in mostly read-only access through Samba |
| `umask`    | `0000`        | Allows full read/write permissions for all users accessing the mounted NTFS drive              |
| `nofail`   | `nofail`      | Prevents boot failure if the drive is disconnected or unavailable                              |
| `defaults` | `defaults`    | Applies standard Linux mount behavior                                                          |
| `ntfs-3g`  | `ntfs-3g`     | Linux NTFS driver providing NTFS read/write support                                            |

Example final mount configuration:

`UUID=A4D09E95D09E6D74 /mnt/data ntfs-3g defaults,uid=1000,gid=1000,umask=0000,nofail 0 0`

This configuration gives Linux and Samba stable ownership, persistent mounting, and full Windows read/write compatibility for the shared NTFS drives.


6. Step 6 - Test the mount configuration safely before rebooting.

The `sudo mount -a` command instructs Linux to immediately mount all filesystems defined in `/etc/fstab`, allowing the configuration to be tested without rebooting the server.

```bash
# “Attempt to mount everything defined inside /etc/fstab right now.”
sudo mount -a
```
<img width="1060" height="780" alt="image" src="https://github.com/user-attachments/assets/d2c65b9e-739c-4f4a-8a36-d69583e4e00f" />


7. Step 7 - Clean NTFS dirty flags

The `ntfsfix` utility is used to safely clear NTFS dirty or hibernation flags left by Windows, allowing the drives to be mounted normally in Linux with full read-write access.

```bash
sudo ntfsfix /dev/sda2
sudo ntfsfix /dev/sdb2
sudo ntfsfix /dev/sdc2
sudo ntfsfix /dev/sdd2
sudo ntfsfix /dev/sdf2
```

<img width="1060" height="1032" alt="image" src="https://github.com/user-attachments/assets/c9e31f3d-d3e9-419b-b970-981854fba5ff" />

8. Step 8 - Test the mount configuration **again** before rebooting.

```bash
sudo mount -a
```

<img width="1060" height="590" alt="image" src="https://github.com/user-attachments/assets/de899950-2575-4d17-bd3a-61bea631904f" />

9. Step 9 - Verify that drives are mounted correctly.

The df -h command displays all currently mounted filesystems, their storage usage, available free space, and mount locations in a human-readable format, such as GB and TB.

```bash
df -h
```

<img width="1060" height="590" alt="image" src="https://github.com/user-attachments/assets/1d7952af-3588-4590-83e8-7a71b1633894" />

Mount points confirmation

| Mount Point | Size | Usage | Status |
| ----------- | ---- | ----- | ------ |
| `/mnt/iac`  | 3.7T | 49%   | ✅      |
| `/mnt/lp`   | 7.3T | 50%   | ✅      |
| `/mnt/data` | 7.3T | 39%   | ✅      |
| `/mnt/ia`   | 3.7T | 85%   | ✅      |
| `/mnt/comp` | 3.7T | 28%   | ✅      |



### Samba network shares

Samba network shares allow Linux directories to be shared over the network using the SMB protocol, making them accessible from Windows systems through File Explorer, like standard network drives. This enables the Ubuntu server to function similarly to a NAS by providing centralized storage access for media, backups, and shared files across the local network.


Step 1 — Install Samba

Samba enables Linux systems to share files and folders over the network using the SMB protocol, allowing Windows devices to access them like standard network drives.

```bash
sudo apt install samba -y
```

<img width="1060" height="590" alt="image" src="https://github.com/user-attachments/assets/f7fa0306-7ee4-473e-ab46-ea0d8c134593" />

Step 2 — Backup Existing Samba Config

```bash
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.backup
```

<img width="1063" height="133" alt="image" src="https://github.com/user-attachments/assets/6195523f-d955-4708-92b1-666af21686c8" />

Step 3 — Open Samba Configuration

```bash
sudo nano /etc/samba/smb.conf
```

<img width="1060" height="590" alt="image" src="https://github.com/user-attachments/assets/9b4c595f-b604-44c3-b6cc-5a300ea1b066" />


Step 4 — Add Your Domain Shares

Append this at the bottom:

```ini

[iac]
   path = /mnt/iac
   browseable = yes
   writable = yes
   guest ok = no
   valid users = piroman

[lp]
   path = /mnt/lp
   browseable = yes
   writable = yes
   guest ok = no
   valid users = piroman

[data]
   path = /mnt/data
   browseable = yes
   writable = yes
   guest ok = no
   valid users = piroman

[ia]
   path = /mnt/ia
   browseable = yes
   writable = yes
   guest ok = no
   valid users = piroman

[comp]
   path = /mnt/comp
   browseable = yes
   writable = yes
   guest ok = no
   valid users = piroman
```

<img width="1060" height="913" alt="image" src="https://github.com/user-attachments/assets/3a5a2955-55c1-4323-aa19-4738884b3612" />

| Option          | Purpose                      |
| --------------- | ---------------------------- |
| `path`          | Linux directory being shared |
| `browseable`    | visible on network           |
| `writable`      | allows write access          |
| `guest ok = no` | requires authentication      |
| `valid users`   | allowed Samba users          |


```bash
# Validate the configuration safely
testparm
```

<img width="1060" height="913" alt="image" src="https://github.com/user-attachments/assets/067b519c-92ba-4a56-9411-6b2ae4d6a658" />

<img width="1060" height="913" alt="image" src="https://github.com/user-attachments/assets/f9931ff1-5ad7-4390-92c0-0fcef4a464e7" />

5. Step 5 - Create Samba User Password

```bash
# This creates the Samba authentication account for: piroman
sudo smbpasswd -a piroman
```
<img width="728" height="100" alt="image" src="https://github.com/user-attachments/assets/da48e63f-25c0-4128-89ed-4513093c1ab1" />


6. Step 6 — Restart Samba

```bash
sudo systemctl restart smbd
```

7. Step 7 — Verify Samba Running

```bash
sudo systemctl status smbd
```
<img width="1060" height="590" alt="image" src="https://github.com/user-attachments/assets/d11cbb72-b0b9-4828-af72-b6c5626cd584" />


8. Step 8 — Open Samba In Firewall

You already did this earlier with:

```bash
sudo ufw allow Samba
```


9. Step 9 — Access From Windows


In Windows Explorer:

```
\\192.168.0.246
```

or 

```
\\piroman-server
```

With IP works, with name not (for now).

<img width="1017" height="263" alt="image" src="https://github.com/user-attachments/assets/3743e748-1d9b-4c7e-84ec-635e455b3a70" />

<img width="510" height="156" alt="image" src="https://github.com/user-attachments/assets/57e0a5ae-584e-42d7-9369-63dc5c754c4c" />

Drives that were successfully mapped in Windows

<img width="268" height="238" alt="image" src="https://github.com/user-attachments/assets/9fd8004b-bd95-4996-8142-cf4c3a6192dc" />


### DNS and Reverse Proxy

Nginx Proxy Manager becomes the centralized ingress layer for all services.

```mermaid
flowchart TD
    A[AdGuard Home DNS] --> B[Nginx Proxy Manager]
    B --> C[Portainer]
    B --> D[Immich]
    B --> E[Plex]
    B --> F[Kavita]
```

| Component     | Solves                                       |
| ------------- | -------------------------------------------- |
| DNS           | “Where is this service?”                     |
| Reverse Proxy | “Which backend should receive this request?” |


#### DNS Foundation (AdGuard Home)

1. Create AdGuard Home Directory Structure

Create isolated application directories following the established Docker architecture principle:

```bash
mkdir -p /srv/docker/adguard-home/{work,conf}
```

This creates:
  - work/ → runtime/work data
  - conf/ → persistent configuration

2. Create AdGuard Home Compose File

Navigate to the application directory:

```bash
cd /srv/docker/adguard-home
```

Create the compose file:

```bash
nano compose.yml
```

<img width="957" height="628" alt="image" src="https://github.com/user-attachments/assets/e576574e-6307-4a89-91be-aaad9b28e477" />

3. Add AdGuard Home Docker Compose Configuration

```yaml
services:
  adguard-home:
    image: adguard/adguardhome:latest
    container_name: adguard-home
    restart: unless-stopped

    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "67:67/udp"
      - "68:68/udp"
      - "3000:3000/tcp"
      - "80:80/tcp"

    volumes:
      - ./work:/opt/adguardhome/work
      - ./conf:/opt/adguardhome/conf
```


<img width="957" height="628" alt="image" src="https://github.com/user-attachments/assets/aefb8111-a869-4404-8310-befe0250798f" />

| Port Mapping    | Protocol | Purpose                                                                                          |
| --------------- | -------- | ------------------------------------------------------------------------------------------------ |
| `53:53/tcp`     | TCP      | Standard DNS queries over TCP. Used for larger DNS responses and some advanced DNS operations.   |
| `53:53/udp`     | UDP      | Main DNS traffic port. Most DNS queries from client devices use UDP.                             |
| `67:67/udp`     | UDP      | DHCP server port used when AdGuard Home provides automatic IP address assignment on the network. |
| `68:68/udp`     | UDP      | DHCP client communication port paired with DHCP server operations.                               |
| `3000:3000/tcp` | TCP      | Initial AdGuard Home setup web interface. Used only during first-time configuration.             |
| `80:80/tcp`     | TCP      | Main AdGuard Home web management interface after setup completion.                               |


###### DNS Ports (53)

Ports `53/tcp` and `53/udp` are the core DNS communication ports used by `AdGuard Home` to process DNS requests from client devices. Most standard DNS queries use UDP, while TCP is used for larger responses and specific DNS operations. Without these ports exposed, `AdGuard Home` cannot function as a network DNS server.


###### DHCP Ports (67, 68)

Ports `67/udp` and `68/udp` are used for DHCP communication when `AdGuard Home` acts as a DHCP server, assigning IP addresses to devices on the network. In the current setup, the ISP router still manages DHCP. Hence, these ports are not immediately required but are kept available for possible future migration of DHCP functionality to `AdGuard Home`.

###### Web Interface Ports (3000, 80)

Ports `3000/tcp` and `80/tcp` provide access to the `AdGuard Home` web management interface. Port `3000` is primarily used during the initial setup wizard, while port 80 serves as the main web administration interface after setup completion. These ports allow browser-based configuration and management of DNS settings, filters, and local domain rewrites.


4. Validate Compose File

```bash
docker compose config
```

This validates:

  - YAML syntax
  - Docker Compose structure
  - Configuration correctness

<img width="1920" height="1032" alt="image" src="https://github.com/user-attachments/assets/53046249-8735-4c76-9e81-cab74d915864" />

5. Deploy AdGuard Home

```bash
docker compose up -d
```
This command does the following tasks:

  - Downloads the image
  - Creates the container
  - Attaches persistent storage
  - Starts AdGuard Home

##### Get an error that port 53 is already in use

<img width="1115" height="628" alt="image" src="https://github.com/user-attachments/assets/f04b7492-858e-4e78-81e7-96cc649997df" />


###### Verify What Uses Port 53

```bash
# Show which process/service is currently using DNS port 53
sudo lsof -i :53
```

Command Not Found 

<img width="1115" height="628" alt="image" src="https://github.com/user-attachments/assets/e2a27fa2-6f22-4ca6-886c-8ff8acd55554" />

or 

``` bash
# Display active network listeners and filter for services using port 53
sudo ss -tulpn | grep :53
```

<img width="1115" height="628" alt="image" src="https://github.com/user-attachments/assets/50ecff9b-0a00-4921-aa62-2e4d1e3a16e2" />

The `systemd-resolved` process is currently binding port `53`, preventing `AdGuard Home` from starting its DNS server.

`systemd-resolved` is Ubuntu’s built-in local DNS resolver service responsible for handling DNS queries, caching DNS responses, and forwarding DNS requests to external DNS servers configured on the system. By default, it listens on the local DNS port 53, allowing Ubuntu itself to perform domain name resolution (for example, resolving google.com to an IP address).

###### So if Ubuntu already has a DNS resolver, why do I need AdGuard Home?

`systemd-resolved` and `AdGuard Home` serve very different purposes, even though both are related to DNS.

`systemd-resolved` is a lightweight local DNS resolver designed primarily for Ubuntu. Its job is to help the operating system resolve Internet domain names, cache DNS requests, and forward queries to external DNS servers. It is intended for local system functionality, not for managing an entire network.

`AdGuard Home`, on the other hand, is a full-featured network DNS platform designed to serve all devices in the homelab. It provides centralized DNS management, ad blocking, tracking protection, local domain rewrites, DNS filtering, statistics, custom records, parental controls, and future integration with reverse proxies and internal infrastructure services.

| Component          | Purpose                                                     |
| ------------------ | ----------------------------------------------------------- |
| `systemd-resolved` | Basic Ubuntu local DNS resolver                             |
| AdGuard Home       | Centralized network-wide DNS server and management platform |


After migration:

- Ubuntu itself will still use DNS normally
- But AdGuard Home becomes the primary DNS authority for:
  - Windows devices
  - Phones
  - Docker services
  - Local homelab domains
  - Ad blocking
  - Internal infrastructure routing

This is why AdGuard Home is a foundational infrastructure component, while systemd-resolved is only an operating system utility.


6. Disable Ubuntu DNS Stub Listener

Edit resolved configuration in `/etc/systemd/resolved.conf` find this line `#DNSStubListener=yes` and change it to `DNSStubListener=no`. If the line does not exist, add it manually anywhere under [Resolve].

<img width="1115" height="628" alt="image" src="https://github.com/user-attachments/assets/9b4bda74-c757-4cdb-909f-cbad2466c95b" />

7. Restart systemd-resolved

```bash
sudo systemctl restart systemd-resolved
```

<img width="1115" height="628" alt="image" src="https://github.com/user-attachments/assets/9407a3f4-3280-4b8f-9d59-9a46dc3a469b" />

8. Recreate resolv.conf Symlink

```bash
# This ensures Ubuntu continues to resolve DNS properly after disabling the stub listener.
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
```

<img width="1115" height="628" alt="image" src="https://github.com/user-attachments/assets/f3cb1a40-2fd2-429b-9712-b6641c3c5d25" />


9. Check that Ubuntu itself can still resolve internet domain names correctly through DNS

```bash
ping google.com
```

or 

```bash
resolvectl query google.com
```

<img width="1115" height="628" alt="image" src="https://github.com/user-attachments/assets/948ed54c-1470-44a0-915f-7eceda979738" />

This confirms everything is working correctly. Ubuntu successfully:

- resolved `google.com`
- returned both IPv4 and IPv6 addresses
- used DNS normally after disabling the stub listener

Most importantly:

- `systemd-resolved` still functions as Ubuntu’s DNS resolver

10. Verify Port 53 Is Free

```bash
sudo ss -tulpn | grep :53
```

<img width="1115" height="628" alt="image" src="https://github.com/user-attachments/assets/6f4de071-0c69-407b-a47f-4bc0b59a054b" />


11. Then Start AdGuard Home Again

At that point, `AdGuard Home` should successfully claim port `53` and start normally.

```bash
cd /srv/docker/adguard-home
docker compose up -d
```

<img width="1115" height="628" alt="image" src="https://github.com/user-attachments/assets/efd9a100-a004-43f7-b628-cb4fb973cc89" />

<img width="1918" height="474" alt="image" src="https://github.com/user-attachments/assets/5ae79a59-dbab-4ae5-8e2e-024f45dd06f3" />





