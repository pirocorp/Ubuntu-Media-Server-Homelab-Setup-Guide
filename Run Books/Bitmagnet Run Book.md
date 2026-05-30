## PostgreSQL Administration

### Open PostgreSQL Shell

Connect to the PostgreSQL container:

```bash
docker exec -it bitmagnet-postgres psql -U bitmagnet -d bitmagnet
```

Expected prompt:

```text
bitmagnet=#
```

### List Tables

```sql
\dt
```

### Show Database Size

```sql
SELECT pg_size_pretty(pg_database_size('bitmagnet'));
```

Example:

```text
  pg_size_pretty
-----------------
 42 GB
```

### Show Largest Tables

```sql
SELECT
    relname AS table_name,
    pg_size_pretty(pg_total_relation_size(relid)) AS size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;
```

Useful when identifying which tables are responsible for database growth.

### Exit PostgreSQL

```sql
\q
```

---

## PostgreSQL Monitoring

### Database Directory Size

Filesystem perspective:

```bash
du -sh /srv/docker/bitmagnet/postgres
```

### Database Size (PostgreSQL)

Database perspective:

```bash
docker exec -it bitmagnet-postgres \
psql -U bitmagnet -d bitmagnet \
-c "SELECT pg_size_pretty(pg_database_size('bitmagnet'));"
```

### Container Resource Usage

```bash
docker stats bitmagnet-postgres
```

---

## Netdata Monitoring

Bitmagnet is primarily:

* CPU intensive during crawling
* Disk intensive during indexing
* PostgreSQL intensive as the database grows

### Recommended Netdata Checks

#### CPU Usage

Navigate to:

```text
Netdata
→ System Overview
→ CPU
```

Watch for:

* Sustained high CPU usage
* CPU spikes during crawling
* CPU saturation

Expected on Xeon E3-1231 v3:

```text
Occasional spikes are normal.
Sustained 100% usage is not.
```

---

#### Memory Usage

Navigate to:

```text
Netdata
→ System Overview
→ Memory
```

Watch for:

* RAM consumption
* Swap usage

Current baseline:

```text
30 GB RAM
8 GB Swap
```

Bitmagnet should not require large amounts of RAM for a personal homelab deployment.

---

#### Disk Usage

Navigate to:

```text
Netdata
→ Disks
→ Space Usage
```

Monitor:

```text
/
```

Expected:

```text
1.8 TB NVMe SSD
```

Recommended alert threshold:

```text
80% utilization
```

---

#### Disk I/O

Navigate to:

```text
Netdata
→ Disks
→ I/O
```

Watch for:

* High write activity
* PostgreSQL indexing activity
* SSD bottlenecks

---

#### Docker Containers

Navigate to:

```text
Netdata
→ Containers
```

Monitor:

```text
bitmagnet
bitmagnet-postgres
```

Key metrics:

* CPU
* Memory
* Network
* Disk I/O

---

## Operational Health Checklist

Weekly:

```bash
docker compose ps
docker compose logs --tail=50 bitmagnet
du -sh /srv/docker/bitmagnet/postgres
df -h /
```

Monthly:

```bash
sudo ncdu -x /srv/docker/bitmagnet
docker stats bitmagnet
docker stats bitmagnet-postgres
```

Quarterly:

```bash
docker compose pull
docker compose config
```

Review:

* Database growth
* SSD capacity
* Container resource consumption
* Available Bitmagnet updates

```
