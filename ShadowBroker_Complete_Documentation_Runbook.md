# ShadowBroker Complete Deployment Documentation & Operations Runbook

## 1. Purpose

This document describes the ShadowBroker server deployment:

- what was built
- why each design decision was made
- how it was implemented
- how to operate, update, troubleshoot, back up, and recover the system

ShadowBroker runs as a self-hosted Docker Compose application providing:

- Frontend web interface
- Backend API services
- Scheduler and ingestion services
- AI agent integration
- InfOnet trust functionality
- Wormhole identity functionality
- I2P private transport support

---

## 2. Deployment Architecture

### Server

Host:

- `pirman-server`
- LAN IP: `192.168.0.10`

Installation directory:

```text
/srv/docker/shadowbroker
```

Directory structure:

```text
/srv/docker/shadowbroker/
├── compose.yml
├── .env
├── data/
│   └── backend/
└── i2pd/
```

Reason:

Application configuration, persistent state, and container definitions are kept together so the entire stack is easy to back up, restore, inspect, and operate.

### Architecture Diagram

```mermaid
flowchart TD
    Operator[Operator / Admin] -->|Browser| Proxy[Nginx Proxy Manager / LAN Reverse Proxy]
    Proxy -->|HTTP :3010| Frontend[shadowbroker-frontend<br/>UI container]
    Frontend -->|API calls| Backend[shadowbroker-backend<br/>FastAPI / scheduler / ingestion]
    Backend --> Data[(./data/backend<br/>persistent app data)]
    Backend --> Wormhole[(Wormhole identity files)]
    Backend --> Transparency[(Transparency ledger)]
    Backend --> I2P[i2p container<br/>SOCKS5 :4447]
    I2P --> PrivateNet[Private I2P transport]
    Backend --> ExternalAPIs[External data/API providers]
    Backend --> AIProvider[AI model/API provider]
```

---

## 3. Container Stack

### Frontend

Container:

```text
shadowbroker-frontend
```

Purpose:

- Provides the ShadowBroker web interface
- Talks to the backend APIs

Ports:

```text
External: 3010
Internal: 3000
```

Configured through:

```env
FRONTEND_PORT=3010
```

### Backend

Container:

```text
shadowbroker-backend
```

Purpose:

- REST API
- Data processing
- Scheduler
- Feed ingestion
- AI agent services
- InfOnet verification
- Wormhole identity management

Ports:

```text
External: 8010
Internal: 8000
```

Configured through:

```env
BACKEND_PORT=8010
```

### I2P

Purpose:

Provides private network transport.

Ports:

```text
7070
4447
```

Internal Docker communication:

```text
i2p:4447
```

LAN access:

```text
192.168.0.10:4447
```

### Container Relationship Diagram

```mermaid
flowchart LR
    subgraph DockerHost["pirman-server / 192.168.0.10"]
        subgraph ShadowBrokerDir["/srv/docker/shadowbroker"]
            Compose[compose.yml]
            Env[.env]
            BackendData[(data/backend)]
            I2PData[(i2pd)]
        end

        Frontend[shadowbroker-frontend<br/>3000 internal / 3010 external]
        Backend[shadowbroker-backend<br/>8000 internal / 8010 external]
        I2P[i2p<br/>4447 SOCKS / 7070 console]

        Frontend --> Backend
        Backend --> BackendData
        Backend --> I2P
        I2P --> I2PData
        Compose --> Frontend
        Compose --> Backend
        Compose --> I2P
        Env --> Frontend
        Env --> Backend
    end
```

---

## 4. Configuration Strategy

### Why `.env` Exists

The deployment separates static infrastructure from deployment-specific values.

Static infrastructure:

```text
compose.yml
```

Environment-specific settings:

```text
.env
```

This allows updates without rewriting Docker configuration.

Important values:

```env
SHADOWBROKER_VERSION=x.x.x
FRONTEND_PORT=3010
BACKEND_PORT=8010
ADMIN_KEY=<secret>
```

### Configuration Flow

```mermaid
flowchart TD
    EnvFile[.env] --> Version[SHADOWBROKER_VERSION]
    EnvFile --> Ports[FRONTEND_PORT / BACKEND_PORT]
    EnvFile --> Admin[ADMIN_KEY]
    Compose[compose.yml] --> Containers[Docker containers]
    Version --> Images[Pinned GHCR images]
    Ports --> PortMap[Host-to-container port mapping]
    Admin --> BackendAuth[Backend/operator authentication]
    Images --> Containers
    PortMap --> Containers
    BackendAuth --> Containers
```

---

## 5. Version Management

Images are pinned.

Reason:

Pinned versions avoid accidental upgrades and reduce the risk of unexpected breaking changes.

Current version is controlled with:

```env
SHADOWBROKER_VERSION=
```

Upgrade example:

```diff
-SHADOWBROKER_VERSION=0.9.83
+SHADOWBROKER_VERSION=0.9.84
```

---

## 6. Daily Operations Runbook

### Enter Application Directory

```bash
cd /srv/docker/shadowbroker
```

### Start ShadowBroker

```bash
docker compose up -d
```

### Stop ShadowBroker

```bash
docker compose down
```

### Restart Everything

```bash
docker compose restart
```

### Restart Backend Only

```bash
docker compose restart backend
```

### Force Recreate Backend

```bash
docker compose up -d --force-recreate backend
```

### Basic Operations Flow

```mermaid
flowchart TD
    Start([Need to operate ShadowBroker]) --> CD[cd /srv/docker/shadowbroker]
    CD --> Choice{What do you need?}
    Choice -->|Start| Up[docker compose up -d]
    Choice -->|Stop| Down[docker compose down]
    Choice -->|Restart all| Restart[docker compose restart]
    Choice -->|Restart backend| RestartBackend[docker compose restart backend]
    Choice -->|Recreate backend| Recreate[docker compose up -d --force-recreate backend]
    Up --> Verify[docker ps + logs]
    Restart --> Verify
    RestartBackend --> Verify
    Recreate --> Verify
```

---

## 7. Health Checks

### Check Containers

```bash
docker ps | grep -E "shadowbroker|i2p"
```

Expected:

```text
shadowbroker-backend    Up
shadowbroker-frontend   Up
i2p                     Up
```

### Backend API Test

```bash
curl http://127.0.0.1:8000
```

### Health Check Decision Tree

```mermaid
flowchart TD
    A[Start health check] --> B[Run docker ps]
    B --> C{All containers Up?}
    C -->|Yes| D[Test backend API with curl]
    C -->|No| E[Check container logs]
    D --> F{API responds?}
    F -->|Yes| G[System appears healthy]
    F -->|No| H[Check backend logs and ports]
    E --> I{Permission or startup error?}
    I -->|Yes| J[Apply permissions fix]
    I -->|No| K[Inspect compose/env and recreate service]
    H --> L[Verify port mapping and backend startup]
```

---

## 8. Logs

Everything:

```bash
docker compose logs -f
```

Backend:

```bash
docker logs -f shadowbroker-backend
```

Frontend:

```bash
docker logs -f shadowbroker-frontend
```

I2P:

```bash
docker logs -f i2p
```

Search errors:

```bash
docker logs shadowbroker-backend 2>&1 | grep ERROR
```

Healthy backend example:

```text
Application startup complete
Uvicorn running on http://0.0.0.0:8000
```

---

## 9. Updating ShadowBroker

### Step 1 - Backup

```bash
cd /srv/docker
tar czvf shadowbroker-backup.tar.gz shadowbroker/
```

### Step 2 - Edit Version

```bash
cd /srv/docker/shadowbroker
nano .env
```

Change:

```env
SHADOWBROKER_VERSION=new_version
```

### Step 3 - Pull Images

```bash
docker compose pull
```

### Step 4 - Deploy

```bash
docker compose up -d
```

### Step 5 - Verify

```bash
docker ps
docker logs shadowbroker-backend -f
```

### Update Workflow

```mermaid
flowchart TD
    A[Start update] --> B[Backup /srv/docker/shadowbroker]
    B --> C[Edit .env]
    C --> D[Set SHADOWBROKER_VERSION]
    D --> E[docker compose pull]
    E --> F[docker compose up -d]
    F --> G[docker ps]
    G --> H[Check backend logs]
    H --> I{Healthy?}
    I -->|Yes| J[Update complete]
    I -->|No| K[Rollback to previous version]
```

---

## 10. Rollback Procedure

Edit:

```text
.env
```

Restore previous version:

```env
SHADOWBROKER_VERSION=old_version
```

Run:

```bash
docker compose pull
docker compose up -d
```

### Rollback Workflow

```mermaid
flowchart TD
    A[Problem after update] --> B[Edit .env]
    B --> C[Set previous SHADOWBROKER_VERSION]
    C --> D[docker compose pull]
    D --> E[docker compose up -d]
    E --> F[Check containers]
    F --> G[Check backend logs]
    G --> H{Recovered?}
    H -->|Yes| I[Rollback complete]
    H -->|No| J[Restore from backup]
```

---

## 11. Backup & Restore

### Backup

```bash
cd /srv/docker
tar czvf shadowbroker-backup.tar.gz shadowbroker/
```

Protect this backup because it contains:

- configuration
- `ADMIN_KEY`
- persistent identity files
- application state

### Restore

```bash
tar xzvf shadowbroker-backup.tar.gz -C /srv/docker/
cd /srv/docker/shadowbroker
docker compose up -d
```

### Backup and Restore Diagram

```mermaid
flowchart LR
    AppDir[/srv/docker/shadowbroker/] --> Archive[shadowbroker-backup.tar.gz]
    Archive --> RestorePath[/srv/docker/shadowbroker restored/]
    RestorePath --> ComposeUp[docker compose up -d]
    ComposeUp --> Running[Running ShadowBroker stack]

    AppDir --> Compose[compose.yml]
    AppDir --> Env[.env]
    AppDir --> Data[data/]
    AppDir --> I2PData[i2pd/]
```

---

## 12. Wormhole Operations

Stored under:

```text
/app/data/
```

Important files:

```text
wormhole.json
wormhole_status.json
operator_api_keys.env
```

Check:

```bash
docker exec shadowbroker-backend ls /app/data
```

### Wormhole Data Flow

```mermaid
flowchart TD
    Backend[shadowbroker-backend] --> AppData[/app/data/]
    AppData --> Identity[wormhole.json]
    AppData --> Status[wormhole_status.json]
    AppData --> Keys[operator_api_keys.env]
    Backend --> OperatorSession[Operator session / admin unlock]
    OperatorSession --> Identity
    OperatorSession --> Keys
```

---

## 13. InfOnet Trust

Transparency ledger:

```text
/app/data/transparency/
```

Expected state:

```text
TRANSPARENCY CURRENT
```

External witness:

- Optional for LAN deployments
- Required only for stronger independent trust verification

### InfOnet Trust Model

```mermaid
flowchart TD
    DMRoot[DM Root trust state] --> Transparency[Local transparency ledger]
    DMRoot --> Witness[External witness]
    Transparency --> LocalCheck[Local readback verification]
    Witness --> IndependentCheck[Independent verification source]
    LocalCheck --> LANTrust[LAN/private deployment trust]
    IndependentCheck --> StrongTrust[Stronger trust model]
```

---

## 14. Permissions Troubleshooting

### Backend Database Error

Example:

```text
sqlite3.OperationalError
unable to open database file
```

Fix:

```bash
cd /srv/docker/shadowbroker

sudo mkdir -p data/backend
sudo chown -R 1000:1000 data/backend
sudo chmod -R 775 data/backend

docker compose restart backend
```

### I2P Permissions

```bash
sudo chown -R 100:65534 i2pd
docker compose restart i2p
```

### Permissions Troubleshooting Flow

```mermaid
flowchart TD
    A[Container restart loop or startup failure] --> B[Check logs]
    B --> C{Backend database permission error?}
    C -->|Yes| D[chown/chmod data/backend]
    C -->|No| E{I2P data permission error?}
    E -->|Yes| F[chown i2pd]
    E -->|No| G[Inspect .env, compose.yml, ports, image version]
    D --> H[Restart backend]
    F --> I[Restart i2p]
    G --> J[Recreate affected service]
```

---

## 15. Network Troubleshooting

Windows tests:

```powershell
Test-NetConnection 192.168.0.10 -Port 3010
Test-NetConnection 192.168.0.10 -Port 8010
Test-NetConnection 192.168.0.10 -Port 4447
```

### Network Path

```mermaid
flowchart LR
    Browser[Browser / Windows client] -->|HTTP/HTTPS| Proxy[Nginx Proxy Manager]
    Proxy -->|HTTP :3010| Frontend[ShadowBroker Frontend]
    Frontend -->|API| Backend[ShadowBroker Backend :8010]
    Backend -->|SOCKS5| I2P[I2P :4447]
```

---

## 16. Reverse Proxy

Recommended:

Scheme:

```text
http
```

Forward host/IP:

```text
192.168.0.10
```

Forward port:

```text
3010
```

Enable:

- SSL
- HTTP/2
- WebSocket support
- Block common exploits

### Reverse Proxy Flow

```mermaid
sequenceDiagram
    participant User as User browser
    participant NPM as Nginx Proxy Manager
    participant FE as shadowbroker-frontend
    participant BE as shadowbroker-backend

    User->>NPM: Open ShadowBroker URL
    NPM->>FE: Forward HTTP request to 192.168.0.10:3010
    FE->>BE: Backend API requests
    BE-->>FE: API responses
    FE-->>NPM: Rendered UI response
    NPM-->>User: HTTPS response
```

---

## 17. Maintenance Checklist

### Daily

- Verify containers are running
- Check backend logs
- Confirm feeds operate
- Review InfOnet status

### Weekly

- Backup `/srv/docker/shadowbroker`
- Review available updates
- Verify API keys
- Check Docker storage

### Monthly

- Test restore process
- Review exposed ports
- Rotate secrets if required

### Maintenance Cycle

```mermaid
flowchart TD
    Daily[Daily checks] --> Weekly[Weekly backup and update review]
    Weekly --> Monthly[Monthly restore/security review]
    Monthly --> Daily

    Daily --> Containers[Containers running]
    Daily --> Logs[Logs clean]
    Daily --> Feeds[Feeds active]
    Weekly --> Backup[Backup completed]
    Weekly --> Keys[API keys verified]
    Monthly --> Restore[Test restore]
    Monthly --> Secrets[Review/rotate secrets]
```

---

## 18. Clean Rebuild Procedure

Warning:

This removes ShadowBroker state.

```bash
cd /srv/docker/shadowbroker

docker compose down

sudo rm -rf data
sudo rm -rf i2pd

mkdir -p data/backend
mkdir -p i2pd

sudo chown -R 1000:1000 data
sudo chown -R 100:65534 i2pd

docker compose pull
docker compose up -d
```

### Clean Rebuild Flow

```mermaid
flowchart TD
    A[Need clean rebuild] --> B[Stop stack]
    B --> C[Remove data and i2pd]
    C --> D[Recreate directories]
    D --> E[Apply ownership]
    E --> F[Pull images]
    F --> G[Start stack]
    G --> H[Reconfigure ShadowBroker state]
```

---

## 19. Recovery Principle

The important assets are:

1. `compose.yml`
2. `.env`
3. `data/`
4. `i2pd/`

With those restored, the complete ShadowBroker environment can be rebuilt.

### Recovery Dependency Diagram

```mermaid
flowchart TD
    Compose[compose.yml] --> Rebuild[Rebuild stack]
    Env[.env] --> Rebuild
    Data[data/] --> Rebuild
    I2PData[i2pd/] --> Rebuild
    Rebuild --> DockerUp[docker compose up -d]
    DockerUp --> ShadowBroker[Operational ShadowBroker]
```

---

## 20. Useful Commands Reference

Enter backend:

```bash
docker exec -it shadowbroker-backend sh
```

Inspect ports:

```bash
docker port shadowbroker-backend
docker port i2p
```

Recreate one service:

```bash
docker compose up -d --force-recreate SERVICE
```

Example:

```bash
docker compose up -d --force-recreate i2p
```

View resource usage:

```bash
docker stats
```

Clean unused Docker resources:

```bash
docker system prune
```

---

## 21. Final Notes

For private LAN operation, required items are:

- Backend running
- Feeds active
- Transparency current

Optional items are:

- External witness
- Strong Trust

Strong Trust requires another trusted witness source.
