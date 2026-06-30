# Immich Update And Backup Runbook

Status: Implemented
Purpose: Focused runbook for updating, backing up, restoring, and validating the deployed Immich stack.
Depends on: [Immich service overview](./README.md)
Related docs: [Services index](../README.md), [Common commands](../../operations/common-commands.md)

This runbook is for the Immich Docker deployment located at:

```bash
/srv/docker/immich
```

It covers the currently documented stack:

- `immich_server`
- `immich_machine_learning`
- `immich_redis`
- `immich_postgres`

Current version control:

```env
IMMICH_VERSION=v2.7.5
```

## 1. Key Paths

Stack files:

```text
/srv/docker/immich/compose.yml
/srv/docker/immich/.env
```

Persistent application data:

```text
/mnt/data/Immich/Library
/srv/docker/immich/postgres
/srv/docker/immich/model-cache
```

Planned future paths, not core restore inputs yet:

```text
/mnt/data/Immich/Backups
/mnt/data/Immich/PhotoAlbums
```

## 2. Standard Update Workflow

Go to the stack directory:

```bash
cd /srv/docker/immich
```

Validate the compose file:

```bash
docker compose config
```

Check current containers:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Image}}" | grep immich
```

Backup before changing anything.

If staying on the same pinned version and only refreshing the deployed images:

```bash
docker compose pull
docker compose up -d
```

If upgrading Immich to a newer pinned app version:

1. Edit `.env`
2. Change `IMMICH_VERSION`
3. Run:

```bash
docker compose pull
docker compose up -d
```

## 3. Post-Update Checks

Verify containers are running:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}" | grep immich
```

Check application logs:

```bash
docker logs -f immich_server
```

Check machine learning logs if needed:

```bash
docker logs -f immich_machine_learning
```

Validate the app opens:

```text
https://immich.pirocorp.com
```

Confirm large uploads still work through the reverse proxy if Nginx configuration or image versions changed.

## 4. Backup Procedure

Recommended approach: stop the stack before filesystem-level backup so the PostgreSQL data directory and managed library state are captured consistently.

Go to the stack directory:

```bash
cd /srv/docker/immich
```

Stop the stack:

```bash
docker compose down
```

Create a backup root if needed:

```bash
mkdir -p /mnt/data/Immich/Backups
```

Archive stack configuration:

```bash
tar czvf /mnt/data/Immich/Backups/immich-stack-files.tar.gz compose.yml .env
```

Archive PostgreSQL data:

```bash
tar czvf /mnt/data/Immich/Backups/immich-postgres.tar.gz postgres
```

Archive the managed library:

```bash
tar czvf /mnt/data/Immich/Backups/immich-library.tar.gz /mnt/data/Immich/Library
```

Optionally archive model cache:

```bash
tar czvf /mnt/data/Immich/Backups/immich-model-cache.tar.gz model-cache
```

Start the stack again:

```bash
docker compose up -d
```

## 5. Restore Procedure

Restore only onto a stopped stack.

Go to the stack directory:

```bash
cd /srv/docker/immich
docker compose down
```

Restore stack files:

```bash
tar xzvf /mnt/data/Immich/Backups/immich-stack-files.tar.gz -C /srv/docker/immich
```

Restore PostgreSQL data:

```bash
tar xzvf /mnt/data/Immich/Backups/immich-postgres.tar.gz -C /srv/docker/immich
```

Restore the managed library:

```bash
tar xzvf /mnt/data/Immich/Backups/immich-library.tar.gz -C /
```

If needed, restore model cache:

```bash
tar xzvf /mnt/data/Immich/Backups/immich-model-cache.tar.gz -C /srv/docker/immich
```

Start the stack:

```bash
docker compose up -d
```

## 6. Restore Validation

Check container status:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}" | grep immich
```

Check for startup errors:

```bash
docker logs immich_server --tail 200
```

If the stack crashes, verify the managed library still contains the required hidden Immich marker files:

```bash
find /mnt/data/Immich/Library -maxdepth 2 -name .immich
```

If that returns nothing, the managed library state is incomplete and must be corrected before the application will start reliably.

## 7. Practical Notes

- `PhotoAlbums` and `Backups` are part of the intended long-term workflow, but they are still in the planning phase.
- The essential restore inputs today are the stack files, PostgreSQL data, and `/mnt/data/Immich/Library`.
- The preferred TV client in this setup is the non-official `Immich TV` app.
- Because reverse proxy timeouts were previously a real issue, treat large-upload validation as part of major updates.
