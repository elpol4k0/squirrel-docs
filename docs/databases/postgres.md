# PostgreSQL

squirrel provides physical backup and point-in-time recovery for PostgreSQL using the native replication protocol — no `pg_dump`, no external tools required.

---

## How It Works

``` mermaid
sequenceDiagram
    autonumber
    participant SQ as squirrel
    participant PG as PostgreSQL

    SQ->>PG: replication connection (BASE_BACKUP protocol)
    PG-->>SQ: base.tar (PGDATA)
    PG-->>SQ: pg_tblspc_OID.tar (tablespaces)
    SQ->>SQ: store as blobs → create postgres-base snapshot
    SQ->>PG: WAL streaming from start_lsn
    loop Every 16 MiB WAL segment
        PG-->>SQ: WAL segment
        SQ->>SQ: store as blobs → create postgres-wal snapshot
    end
```

squirrel uses the same protocol as `pg_basebackup` and `streaming replication slaves`. No superuser is required — only `REPLICATION` privilege.

---

## Requirements

### PostgreSQL Version

PostgreSQL **12 or later**. Both the V1 (`<= PG 14`) and V2 (`>= PG 15`) base backup streaming protocols are supported automatically.

### PostgreSQL Configuration

```ini
# postgresql.conf
wal_level = replica          # or logical
max_wal_senders = 5          # at least 2
max_replication_slots = 5    # at least 1
```

### Replication User

```sql
CREATE USER squirrel REPLICATION LOGIN PASSWORD 'secret';

-- Allow reading all databases for the dump connection (PG 14+)
GRANT pg_read_all_data TO squirrel;
```

### pg_hba.conf

Allow the squirrel host to connect for replication:

```
# TYPE  DATABASE     USER       ADDRESS         METHOD
host    replication  squirrel   10.0.0.0/8     scram-sha-256
```

For local Docker testing (any host, no TLS):

```
host    replication  all        0.0.0.0/0       trust
```

---

## Backup

### Full Backup (base + WAL)

```bash
squirrel backup postgres \
  --dsn "postgres://squirrel:secret@localhost/postgres?sslmode=disable" \
  --repo /backup/pg-repo \
  --slot squirrel \
  --tag production
```

This command:
1. Performs a base backup (physical copy of `PGDATA`).
2. Starts WAL streaming from the base backup's start LSN.
3. Runs until **Ctrl-C** is pressed or the context is cancelled.

### WAL-Only Streaming

```bash
squirrel backup postgres \
  --dsn "postgres://squirrel:secret@localhost/postgres?sslmode=disable" \
  --repo /backup/pg-repo \
  --slot squirrel \
  --wal-only
```

Start a dedicated WAL archiver after an initial base backup. Use a scheduled job or `squirrel daemon` to run this continuously.

### Config Profile

```yaml
profiles:
  pg-prod:
    repository: s3-prod
    type: postgres
    dsn: ${vault-prod:secret/data/postgres#replication_url}
    slot: squirrel
    tags: [postgres, production]
    schedule: "0 3 * * *"
    retention:
      keep-last: 5
      keep-daily: 14
      keep-weekly: 8
      prune: true
    hooks:
      pre-backup:
        - command: [psql, -c, "CHECKPOINT"]
      post-success:
        - webhook: https://monitoring.example.com/healthcheck/pg-prod
      post-failure:
        - webhook: https://alerts.example.com/pg-prod-backup-failed
```

---

## WAL and PITR

### What is WAL?

PostgreSQL's Write-Ahead Log (WAL) records every change to the database in a sequential log. WAL segments are fixed-size files (16 MiB by default) written to `pg_wal/`.

squirrel streams WAL segments via the physical replication protocol and stores them as `postgres-wal` snapshots. This enables:

- **Point-in-Time Recovery (PITR)** — restore the database to any moment between the base backup and the last WAL segment.
- **Near-zero RPO** — WAL is archived continuously, not just at backup time.

### WAL Segment Names

WAL segments are named `<timeline><lsn_high><lsn_low>`, e.g.:

```
000000010000000000000005
```

squirrel stores segments with their original names for compatibility with `restore_command`.

---

## Restore

### Basic Restore (recover to latest)

```bash
squirrel restore postgres abc12345 \
  --repo /backup/pg-repo \
  --target /var/lib/postgresql/17/main \
  --verify
```

### PITR — Restore to Specific Time

```bash
squirrel restore postgres abc12345 \
  --repo /backup/pg-repo \
  --target /var/lib/postgresql/17/main \
  --pitr "2026-04-20 14:30:00"
```

### PITR — Restore to Specific LSN

```bash
squirrel restore postgres abc12345 \
  --repo /backup/pg-repo \
  --target /var/lib/postgresql/17/main \
  --pitr-lsn "0/5000028"
```

### What Restore Does

1. Extracts `base.tar` to `--target` (the PostgreSQL data directory).
2. Rewrites tablespace symlinks to local paths.
3. Extracts all WAL segments from matching `postgres-wal` snapshots to `--wal-dir`.
4. Writes `recovery.signal` to the data directory.
5. Appends recovery configuration to `postgresql.auto.conf`:
   ```ini
   restore_command = 'cp /tmp/wal-dir/%f %p'
   recovery_target_time = '2026-04-20 14:30:00'   # if --pitr
   recovery_target_action = 'promote'
   ```

### Starting PostgreSQL After Restore

```bash
chown -R postgres:postgres /var/lib/postgresql/17/main
systemctl start postgresql@17-main
tail -f /var/log/postgresql/postgresql-17-main.log
```

PostgreSQL detects `recovery.signal`, applies WAL, stops at the target, promotes to primary.

---

## Replication Slot Management

### Why Slots Matter

A physical replication slot prevents PostgreSQL from discarding WAL segments that squirrel has not yet archived. Without a slot, the primary may recycle WAL before squirrel reads it.

### Slot Retention Risk

If squirrel stops streaming WAL for an extended period, the slot causes the primary to accumulate WAL indefinitely. Monitor slot lag:

```sql
SELECT slot_name, restart_lsn, pg_size_pretty(
  pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
) AS lag
FROM pg_replication_slots;
```

### Drop a Stuck Slot

```bash
squirrel pg drop-slot \
  --dsn "postgres://squirrel:secret@localhost/postgres?sslmode=disable" \
  --slot squirrel
```

---

## Snapshot Types

| Type | Contents | Key Metadata |
|---|---|---|
| `postgres-base` | Physical PGDATA (base.tar + tablespace TARs) | `start_lsn`, `system_id`, `timeline` |
| `postgres-wal` | WAL segment files | `slot`, `timeline`, `segments[]`, `base_snapshot`, `system_id` |

List base backups:

```bash
squirrel snapshots --repo /backup/pg-repo --type postgres-base
```

List WAL archives:

```bash
squirrel snapshots --repo /backup/pg-repo --type postgres-wal
```

---

## Best Practices

1. **Run WAL streaming continuously** alongside scheduled base backups to minimize RPO.
2. **Monitor slot lag** — alert if WAL accumulation exceeds a safe threshold (e.g. 10 GiB).
3. **Verify restores regularly** by testing recovery in a staging environment.
4. **Use `--verify`** on every restore to confirm WAL coverage.
5. **Set a retention policy** that keeps enough WAL for your recovery window:
   ```yaml
   retention:
     keep-last: 5        # base backups
     keep-daily: 14
     prune: true
   ```
6. **Run `CHECKPOINT`** in a pre-backup hook to minimize recovery time after restore.
