# Nextcloud Docker Upgrade Runbook

This runbook is for the Nextcloud Docker deployment located at:

```bash
/srv/docker/nextcloud
```

It covers safe upgrades for the current stack:

- `nextcloud-app`
- `nextcloud-db`
- `nextcloud-redis`
- `nextcloud-cron`
- `nextcloud-mail-worker`
- `nextcloud-apps-init`

The deployment uses pinned major versions in `.env`, for example:

```env
NEXTCLOUD_VERSION=32-apache
POSTGRES_VERSION=16-alpine
REDIS_VERSION=8-alpine
```

---

# 1. Golden Rules

Do not upgrade Nextcloud from the web UI updater.

Use Docker Compose and `.env` only.

Do not skip major versions.

Correct path:

```text
32.x -> 33.x -> 34.x
```

Wrong path:

```text
31.x -> 33.x
```

Always backup before changing `NEXTCLOUD_VERSION`.

---

# 2. Pre-Upgrade Checks

Go to the app folder:

```bash
cd /srv/docker/nextcloud
```

Check current containers:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}"
```

Check current Nextcloud version:

```bash
docker exec -u www-data nextcloud-app php occ status
```

Check maintenance mode:

```bash
docker exec -u www-data nextcloud-app php occ maintenance:mode
```

Expected:

```text
Maintenance mode is currently disabled
```

Check background job mode:

```bash
docker exec -u www-data nextcloud-app php occ config:system:get backgroundjobs_mode
```

Expected:

```text
cron
```

Check Calendar invitation handling:

```bash
docker exec -u www-data nextcloud-app php occ config:app:get dav handleCalendarInvitations
```

Expected:

```text
yes
```

---

# 3. Backup Before Upgrade

Create a backup folder:

```bash
mkdir -p /srv/docker/nextcloud/backups
```

Backup compose and environment files:

```bash
cp compose.yml backups/compose.yml.$(date +%F-%H%M).bak
cp .env backups/.env.$(date +%F-%H%M).bak
```

Backup PostgreSQL database:

```bash
docker exec -t nextcloud-db pg_dump -U nextcloud nextcloud > backups/nextcloud-db.$(date +%F-%H%M).sql
```

Backup Nextcloud app data/config folder:

```bash
tar czf backups/nextcloud-files.$(date +%F-%H%M).tar.gz nextcloud
```

Optional full app folder backup:

```bash
tar czf /srv/docker/nextcloud-backup.$(date +%F-%H%M).tar.gz /srv/docker/nextcloud
```

---

# 4. Stop Non-Essential Workers

Stop the mail worker and cron before upgrading:

```bash
docker stop nextcloud-mail-worker nextcloud-cron
```

Keep database and Redis running.

---

# 5. Enable Maintenance Mode

```bash
docker exec -u www-data nextcloud-app php occ maintenance:mode --on
```

Verify:

```bash
docker exec -u www-data nextcloud-app php occ maintenance:mode
```

---

# 6. Change Nextcloud Major Version

Edit `.env`:

```bash
nano .env
```

Example upgrade from Nextcloud 32 to 33:

```env
NEXTCLOUD_VERSION=33-apache
```

Save the file.

---

# 7. Pull New Docker Images

```bash
docker compose pull
```

---

# 8. Recreate Containers

```bash
docker compose up -d
```

Check containers:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}"
```

---

# 9. Run Nextcloud Upgrade

```bash
docker exec -u www-data nextcloud-app php occ upgrade
```

If upgrade succeeds, disable maintenance mode:

```bash
docker exec -u www-data nextcloud-app php occ maintenance:mode --off
```

---

# 10. Post-Upgrade Repair Commands

Run database/index repairs:

```bash
docker exec -u www-data nextcloud-app php occ db:add-missing-indices
```

```bash
docker exec -u www-data nextcloud-app php occ db:add-missing-columns
```

```bash
php occ db:add-missing-primary-keys
```

```bash
docker exec -u www-data nextcloud-app php occ maintenance:repair
```

Optional file scan:

```bash
docker exec -u www-data nextcloud-app php occ files:scan --all
```

---

# 11. Re-Apply Desired Configuration

Your `apps` init container should apply most of this automatically, but verify manually.

Check apps:

```bash
docker exec -u www-data nextcloud-app php occ app:list | grep -E "calendar|contacts|mail"
```

Ensure cron mode:

```bash
docker exec -u www-data nextcloud-app php occ background:cron
```

Store cron mode explicitly:

```bash
docker exec -u www-data nextcloud-app php occ config:system:set backgroundjobs_mode --value=cron
```

Ensure invitation handling:

```bash
docker exec -u www-data nextcloud-app php occ config:app:set dav handleCalendarInvitations --value=yes
```

Ensure Yahoo IMAP thread limit:

```bash
docker exec -u www-data nextcloud-app php occ config:app:set mail max_imap_thread_count --value=1
```

If using dedicated mail worker, disable built-in Mail background sync to reduce locking:

```bash
docker exec -u www-data nextcloud-app php occ config:app:set mail background-sync --value=false
```

---

# 12. Restart Workers

```bash
docker start nextcloud-cron nextcloud-mail-worker
```

Or recreate all services:

```bash
docker compose up -d
```

---

# 13. Validation Checklist

Check Nextcloud status:

```bash
docker exec -u www-data nextcloud-app php occ status
```

Expected:

```text
installed: true
maintenance: false
version: 33.x.x
```

Check cron:

```bash
docker logs --tail=50 nextcloud-cron
```

Check mail worker:

```bash
docker logs --tail=50 nextcloud-mail-worker
```

Check Yahoo mail account:

```bash
docker exec -u www-data nextcloud-app php occ mail:account:export admin
```

Diagnose mail account:

```bash
docker exec -u www-data nextcloud-app php occ mail:account:diagnose 1
```

Expected:

```text
IMAP connection: OK
SMTP connection: OK
```

Test mail sync manually only if the mail worker is stopped:

```bash
docker stop nextcloud-mail-worker
```

```bash
docker exec -u www-data nextcloud-app php occ mail:account:sync 1 --force -vvv
```

```bash
docker start nextcloud-mail-worker
```

---

# 14. Calendar Invitation Test

Create a new Outlook/Gmail meeting invite to:

```text
zdr0@yahoo.com
```

Expected flow:

```text
Corporate Outlook/Gmail
        ↓
Yahoo Mail
        ↓
nextcloud-mail-worker pulls message
        ↓
Nextcloud Mail detects invitation
        ↓
Accept / Decline / Tentative buttons appear
        ↓
After Accept/Tentative, event appears in Nextcloud Calendar
```

Note: Nextcloud usually does not auto-create tentative events exactly like Outlook/Exchange. It detects the invite and creates the calendar event after your response.

---

# 15. Rollback Procedure

Use rollback only if the upgrade fails and cannot be repaired.

Stop containers:

```bash
docker compose down
```

Restore `.env` and `compose.yml` from backup:

```bash
cp backups/.env.YYYY-MM-DD-HHMM.bak .env
cp backups/compose.yml.YYYY-MM-DD-HHMM.bak compose.yml
```

Restore Nextcloud folder if needed:

```bash
rm -rf nextcloud
```

```bash
tar xzf backups/nextcloud-files.YYYY-MM-DD-HHMM.tar.gz
```

Restore PostgreSQL database if needed:

```bash
docker compose up -d db
```

```bash
cat backups/nextcloud-db.YYYY-MM-DD-HHMM.sql | docker exec -i nextcloud-db psql -U nextcloud nextcloud
```

Start everything:

```bash
docker compose up -d
```

Check status:

```bash
docker exec -u www-data nextcloud-app php occ status
```

---

# 16. Common Issues

## Maintenance mode stuck

```bash
docker exec -u www-data nextcloud-app php occ maintenance:mode --off
```

## MailboxLockedException

This usually means two sync processes are running at the same time.

Stop mail worker before manual sync:

```bash
docker stop nextcloud-mail-worker
```

Then run sync:

```bash
docker exec -u www-data nextcloud-app php occ mail:account:sync 1 --force -vvv
```

Restart worker:

```bash
docker start nextcloud-mail-worker
```

## Yahoo IMAP authentication errors

Check account:

```bash
docker exec -u www-data nextcloud-app php occ mail:account:diagnose 1
```

If authentication fails, regenerate Yahoo App Password and update Nextcloud Mail account.

## Apps missing after upgrade

Run the apps init container:

```bash
docker compose run --rm apps
```

Or enable manually:

```bash
docker exec -u www-data nextcloud-app php occ app:enable calendar
```

```bash
docker exec -u www-data nextcloud-app php occ app:enable contacts
```

```bash
docker exec -u www-data nextcloud-app php occ app:enable mail
```

---

# 17. Final Post-Upgrade Checklist

- [ ] `nextcloud-app` is running
- [ ] `nextcloud-db` is running
- [ ] `nextcloud-redis` is running
- [ ] `nextcloud-cron` is running
- [ ] `nextcloud-mail-worker` is running
- [ ] `occ status` shows maintenance false
- [ ] Calendar app works
- [ ] Mail app works
- [ ] Yahoo IMAP diagnose is OK
- [ ] Invitation actions appear in Mail
- [ ] Calendar events can be accepted
- [ ] iPhone CalDAV still syncs
- [ ] Nginx Proxy Manager route still works
- [ ] `https://nextcloud.home` opens correctly
