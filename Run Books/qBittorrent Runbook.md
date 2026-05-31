# qBittorrent ↔ Plex Integration

## Overview

This setup downloads media directly into Plex library folders and automatically refreshes Plex when a torrent finishes downloading.

### Workflow

```text
Torrent added
      ↓
Downloaded directly to Plex library folder
      ↓
Torrent reaches 100%
      ↓
qBittorrent executes completion hook
      ↓
Plex library refresh is triggered
      ↓
New media appears automatically in Plex
      ↓
Torrent continues seeding
```

---

## Container Information

### Container Name

```bash
qbittorrent
```

### Docker Directory

```bash
/srv/docker/qbittorrent
```

### qBittorrent Version

```text
lscr.io/linuxserver/qbittorrent:5.2.1
```

### User Mapping

```text
PUID=1000
PGID=1000
```

Container files are owned by the `piroman` user.

---

## Plex Library Mount

### Host Path

```bash
/mnt/data/Plex
```

### Container Path

```bash
/plex
```

Current structure:

```text
/plex
├── Movies
├── Series
└── Transcode
```

---

## qBittorrent Categories

### Movies

```text
Category: plex-movies
Save Path: /plex/Movies
```

### Series

```text
Category: plex-series
Save Path: /plex/Series
```

---

## Plex Refresh Script

### Host Location

```bash
/srv/docker/qbittorrent/scripts/plex-refresh.sh
```

### Container Location

```bash
/scripts/plex-refresh.sh
```

### Script Contents

```bash
#!/bin/sh

echo "$(date) - Torrent completed, refreshing Plex" >> /config/plex-refresh.log

curl -s -X POST \
"http://192.168.0.10:32400/library/sections/all/refresh?X-Plex-Token=YOUR_PLEX_TOKEN" \
>/dev/null 2>&1
```

### Make Script Executable

```bash
chmod +x /srv/docker/qbittorrent/scripts/plex-refresh.sh
```

---

## Docker Volume Mapping

```yaml
volumes:
  - /srv/docker/qbittorrent/config:/config
  - /srv/docker/qbittorrent/scripts:/scripts:ro
  - /mnt/data/Plex:/plex
```

---

## qBittorrent Completion Hook

Navigate to:

```text
Options
→ Downloads
→ Run external program
```

Enable:

```text
Run on torrent finished
```

Command:

```bash
/scripts/plex-refresh.sh
```

---

## Logging

### Log File

```bash
/config/plex-refresh.log
```

### View Log

```bash
docker exec -it qbittorrent cat /config/plex-refresh.log
```

Example:

```text
Sun May 31 14:08:02 CEST 2026 - Torrent completed, refreshing Plex
Sun May 31 14:35:47 CEST 2026 - Torrent completed, refreshing Plex
```

---

## Testing

### Execute Script Manually

```bash
docker exec -it qbittorrent /scripts/plex-refresh.sh
```

### Verify Log Entry

```bash
docker exec -it qbittorrent cat /config/plex-refresh.log
```

### Verify Plex Refresh

Navigate to:

```text
Plex
→ Settings
→ Status
→ Alerts
```

Expected messages:

```text
Scanning the "Филми" section
Scanning the "ТВ сериали" section
Library scan complete
```

---

## Torrent Port Configuration

### qBittorrent Listening Port

```text
6881
```

### Docker Ports

```yaml
ports:
  - "8080:8080"
  - "6881:6881/tcp"
  - "6881:6881/udp"
```

### Router Port Forwarding

```text
External Port: 6881
Internal IP: 192.168.0.10
Internal Port: 6881
Protocol: TCP + UDP
```

### External Verification

```bash
curl -4 ifconfig.me
```

Port validation:

```text
https://canyouseeme.org/
```

Expected result:

```text
Success: I can see your service on <public-ip> on port 6881
```

---

## Useful Commands

### Start qBittorrent

```bash
cd /srv/docker/qbittorrent
docker compose up -d
```

### Stop qBittorrent

```bash
cd /srv/docker/qbittorrent
docker compose down
```

### View Logs

```bash
docker compose logs -f qbittorrent
```

### Enter Container

```bash
docker exec -it qbittorrent sh
```

### Verify Script Exists

```bash
docker exec -it qbittorrent ls -l /scripts
```

### Verify Plex Refresh Log

```bash
docker exec -it qbittorrent cat /config/plex-refresh.log
```

---

## Notes

* Downloads are stored directly in Plex library folders.
* No Radarr or Sonarr are required.
* Completed torrents remain available for seeding.
* Plex refreshes automatically when a torrent completes.
* Library refresh is triggered using the Plex HTTP API.
* Port 6881 is externally reachable and forwarded through the router.
