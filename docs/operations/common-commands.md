# Common Commands

Status: Implemented
Purpose: Shared command reference for health checks, troubleshooting, and routine maintenance.
Related docs: [Operations index](./README.md), [Current state](../overview/current-state.md)

## Homelab Health Check Commands

These commands provide a quick overview of the server's hardware, resource usage, storage capacity, Docker environment, networking, and service health. They are useful for routine maintenance, troubleshooting, capacity planning, and validating infrastructure changes.

### Hardware & System Information

| Command                      | Purpose                              | Example Output / Notes                           |
| ---------------------------- | ------------------------------------ | ------------------------------------------------ |
| `lscpu`                      | Displays detailed CPU information    | Model, cores, threads, architecture, cache sizes |
| `lscpu \| grep "Model name"` | Shows only the CPU model             | `Intel(R) Xeon(R) CPU E3-1231 v3 @ 3.40GHz`      |
| `lscpu \| grep "^CPU(s)"`    | Shows total logical CPUs (threads)   | `CPU(s): 8`                                      |
| `nproc`                      | Displays available processing units  | Quick CPU thread count                           |
| `free -h`                    | Displays RAM and swap usage          | Human-readable memory statistics                 |
| `hostnamectl`                | Displays hostname and OS information | Useful for inventory and documentation           |
| `uname -a`                   | Displays Linux kernel information    | Useful for troubleshooting                       |
| `sudo ncdu -x /`             | Show who consumes free space         | Useful for Space Managment                       |


---

### Storage & Filesystems

| Command                                | Purpose                                                    | Example Output / Notes                        |
| -------------------------------------- | ---------------------------------------------------------- | --------------------------------------------- |
| `df -h`                                | Displays mounted filesystems and disk usage                | Human-readable storage usage                  |
| `df -h /`                              | Displays Linux system drive usage only                     | Useful for monitoring SSD capacity            |
| `lsblk`                                | Lists block devices and partitions                         | Physical storage overview                     |
| `lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT` | Lists disks with filesystems and mount points              | Useful for validating storage architecture    |
| `mount \| column -t`                   | Displays mounted filesystems in a readable format          | Verify active mounts                          |
| `du -sh /srv/docker/*`                 | Shows size of Docker application directories               | Identify large Docker applications            |
| `tree /srv/docker`                     | Displays Docker folder structure                           | Infrastructure overview                       |
| `sudo tree /srv/docker`                | Displays all Docker directories including restricted paths | Useful when container files are owned by root |

---

### Docker Environment

| Command                   | Purpose                                     | Example Output / Notes             |
| ------------------------- | ------------------------------------------- | ---------------------------------- |
| `docker ps`               | Lists running containers                    | Shows status, ports, and uptime    |
| `docker ps -a`            | Lists all containers                        | Includes stopped containers        |
| `docker images`           | Lists downloaded Docker images              | Useful for cleanup and audits      |
| `docker stats`            | Displays real-time container resource usage | CPU, memory, network, and disk I/O |
| `docker volume ls`        | Lists Docker volumes                        | Useful for backup planning         |
| `docker network ls`       | Lists Docker networks                       | Network troubleshooting            |
| `systemctl status docker` | Displays Docker service status              | Verify Docker is running           |

---

### Networking & DNS

| Command                       | Purpose                                          | Example Output / Notes                   |
| ----------------------------- | ------------------------------------------------ | ---------------------------------------- |
| `ip addr show`                | Displays network interfaces and IP addresses     | Useful for identifying server IPs        |
| `ss -tulpn`                   | Displays listening ports and associated services | Linux equivalent of `netstat -ano`       |
| `sudo ss -tulpn \| grep :53`  | Shows which process is using DNS port 53         | Useful when troubleshooting AdGuard Home |
| `sudo lsof -i :PORT`          | Shows which process is using a specific port     | Replace `PORT` with the desired port     |
| `resolvectl status`           | Displays current DNS configuration               | DNS troubleshooting                      |
| `resolvectl query google.com` | Tests DNS resolution                             | Verifies DNS functionality               |
| `ping google.com`             | Tests internet connectivity and DNS resolution   | Basic connectivity check                 |

---

### Quick Homelab Health Check

Run the following commands periodically to verify the overall health of the homelab:

```bash
# CPU
lscpu | grep "Model name"

# RAM
free -h

# System SSD usage
df -h /

# Mounted storage drives
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT

# Running containers
docker ps

# Docker resource usage
docker stats

# Docker storage consumption
du -sh /srv/docker/*

# Open ports and services
ss -tulpn
```

### Current Hardware Baseline

| Component          | Value                           |
| ------------------ | ------------------------------- |
| CPU                | Intel Xeon E3-1231 v3 @ 3.40GHz |
| Logical CPUs       | 8                               |
| RAM                | 32 GB                           |
| System Drive       | 2 TB NVMe SSD                   |
| Data Drives        | Multiple NTFS HDDs              |
| Container Platform | Docker + Docker Compose         |
| Management         | Portainer + Netdata             |
| DNS                | AdGuard Home                    |
| Reverse Proxy      | Nginx Proxy Manager             |

This baseline can be updated in the future after hardware upgrades or infrastructure changes.
