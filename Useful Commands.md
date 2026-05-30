# Useful Commands

## Useful Hardware & System Information Commands

| Command | Purpose | Example Output / Notes |
|----------|----------|----------|
| `lscpu` | Displays detailed CPU information | Model, cores, threads, architecture, cache sizes |
| `lscpu \| grep "Model name"` | Shows only the CPU model | `Intel(R) Xeon(R) CPU E3-1231 v3 @ 3.40GHz` |
| `lscpu \| grep "^CPU(s)"` | Shows total logical CPUs (threads) | `CPU(s): 8` |
| `nproc` | Displays available processing units | Useful quick CPU thread count |
| `free -h` | Displays RAM and swap usage in human-readable format | Shows total, used, free, and available memory |
| `df -h` | Displays mounted filesystems and disk usage | Shows total size, used space, free space, and mount points |
| `df -h /` | Shows usage of the Linux system drive only | Useful for monitoring root filesystem capacity |
| `lsblk` | Lists block devices and partitions | Shows disks and partition hierarchy |
| `lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT` | Lists disks with filesystem types and mount points | Useful for storage inventory and troubleshooting |
| `mount \| column -t` | Displays currently mounted filesystems in a readable format | Helpful for verifying active mounts |
| `du -sh /srv/docker/*` | Shows size of each Docker application directory | Useful for identifying storage-heavy containers |
| `docker ps` | Lists running Docker containers | Shows status, ports, uptime, and container names |
| `docker stats` | Displays real-time container resource usage | CPU, memory, network, and disk I/O |
| `systemctl status docker` | Displays Docker service status | Confirms Docker is running correctly |
| `hostnamectl` | Displays system hostname and OS information | Shows hostname, OS version, kernel, architecture |
| `uname -a` | Displays Linux kernel information | Useful for troubleshooting and documentation |
| `ip addr show` | Displays network interfaces and IP addresses | Useful for identifying server IPs |
| `ss -tulpn` | Lists listening ports and associated services | Linux equivalent of Windows `netstat -ano` |
| `sudo ss -tulpn \| grep :53` | Shows which process is using DNS port 53 | Useful when troubleshooting AdGuard Home |
| `sudo lsof -i :PORT` | Shows which process is using a specific port | Replace `PORT` with the desired port number |
| `resolvectl status` | Displays current DNS configuration | Useful for troubleshooting DNS issues |
| `resolvectl query google.com` | Tests DNS resolution through the current resolver | Verifies DNS functionality |
| `ping google.com` | Tests network connectivity and DNS resolution | Basic connectivity check |
| `tree /srv/docker` | Displays Docker folder structure | Useful for documenting infrastructure layout |
| `sudo tree /srv/docker` | Displays all Docker directories including restricted paths | Useful when container files are owned by root |
