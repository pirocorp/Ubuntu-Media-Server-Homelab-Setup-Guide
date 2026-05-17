- Server IP: 192.168.0.10
- Router: 192.168.0.1
- AdGuard: 3000
- NPM: 81 / 443
- Plex: 32400
- Portainer: 9443
- Cockpit: 9090


### Standard Docker Update Workflow

Recommended maintenance process:

- Download images `docker compose pull`
- Recreate containers using new images `docker compose up -d`
- Avoid using: `docker compose down` unless rebuilding or troubleshooting.
