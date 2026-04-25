# Quickstart

This page walks through the most common workflows from scratch.

---

## File Backup

### 1. Initialize a Repository

```bash
squirrel init --repo /backup/myrepo
# Enter a password when prompted
```

Expected output:
```
Repository initialised at /backup/myrepo
```

### 2. Back Up a Directory

```bash
squirrel backup --repo /backup/myrepo --path /home/user/data --tag daily
```

Expected output:
```
Snapshot abc12345 saved  (42 blobs, 128 MiB)
```

### 3. List Snapshots

```bash
squirrel snapshots --repo /backup/myrepo
```

### 4. Restore a Snapshot

```bash
squirrel restore abc12345 --repo /backup/myrepo --target /restore/data
```

### 5. Apply Retention Policy

```bash
squirrel forget --repo /backup/myrepo --keep-last 7 --keep-daily 30 --prune
```

---

## PostgreSQL Backup

### Requirements

- PostgreSQL 12+
- A replication user with `REPLICATION` privilege
- `wal_level = replica` (or `logical`)

### Base Backup + WAL Streaming

```bash
squirrel backup postgres \
  --dsn "postgres://replicator:pw@localhost/postgres?sslmode=disable" \
  --repo s3:mybucket/pg-backup \
  --slot squirrel \
  --tag production
```

Press **Ctrl-C** to stop WAL streaming once enough WAL has been captured.

### WAL-Only (after a base backup exists)

```bash
squirrel backup postgres \
  --dsn "postgres://replicator:pw@localhost/postgres?sslmode=disable" \
  --repo s3:mybucket/pg-backup \
  --slot squirrel \
  --wal-only
```

### Restore with PITR

```bash
squirrel restore postgres abc12345 \
  --repo s3:mybucket/pg-backup \
  --target /var/lib/postgresql/17/main \
  --pitr "2026-04-20 14:30:00" \
  --verify
```

---

## MySQL / MariaDB Backup

### Requirements

- MySQL 5.7 / 8.x or MariaDB 10.5+
- Binary logging enabled: `log_bin = ON`
- User privileges: `RELOAD`, `REPLICATION SLAVE`, `REPLICATION CLIENT`, `SELECT`

### Logical Dump + Binlog Streaming

```bash
squirrel backup mysql \
  --dsn "root:pw@tcp(localhost:3306)/" \
  --repo /backup/mysql \
  --database myapp \
  --tag daily
```

### Restore SQL Dump

```bash
squirrel restore mysql abc12345 \
  --repo /backup/mysql \
  --dsn "root:pw@tcp(localhost:3306)/" \
  --sql-only
```

### Extract Binlogs for PITR

```bash
squirrel restore mysql abc12345 \
  --repo /backup/mysql \
  --dsn "root:pw@tcp(localhost:3306)/" \
  --binlog-dir /tmp/binlogs
```

Replay with `mysqlbinlog`:

```bash
mysqlbinlog /tmp/binlogs/*.bin | mysql -uroot -p
```

---

## S3 Backend

```bash
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...

squirrel init --repo s3:mybucket/prefix
squirrel backup --path /data --repo s3:mybucket/prefix
```

---

## Using a Config File

Create `~/.config/squirrel/config.yml` (Linux/macOS) or `%APPDATA%\squirrel\config.yml` (Windows):

```yaml
version: 1

repositories:
  production:
    url: s3:mybucket/prod
    password: ${env:SQUIRREL_PASSWORD}

profiles:
  pg-daily:
    repository: production
    type: postgres
    dsn: ${env:PG_DSN}
    slot: squirrel
    schedule: "0 2 * * *"
    retention:
      keep-daily: 7
      keep-weekly: 4
      keep-monthly: 12
      prune: true
    hooks:
      post-success:
        - webhook: https://monitoring.example.com/healthcheck/pg-daily
```

Run the profile:

```bash
squirrel run pg-daily
```

See [Configuration](configuration/index.md) for the complete reference.
