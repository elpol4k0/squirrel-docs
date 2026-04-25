# MySQL / MariaDB

squirrel supports logical (SQL dump), physical (data directory), and binlog-based backups for MySQL and MariaDB.

---

## Supported Versions

| Server | Minimum Version |
|---|---|
| MySQL | 5.7, 8.0, 8.4 |
| MariaDB | 10.5, 10.6, 11.x |

squirrel automatically detects the server flavor (MySQL vs MariaDB) and adapts accordingly.

---

## Requirements

### Binary Logging

Binary logging must be enabled in `my.cnf`:

```ini
[mysqld]
log_bin = ON           # or log_bin = /var/log/mysql/mysql-bin
binlog_format = ROW    # recommended for consistent PITR
```

### User Privileges

```sql
CREATE USER 'squirrel'@'%' IDENTIFIED BY 'secret';
GRANT RELOAD, REPLICATION SLAVE, REPLICATION CLIENT, SELECT ON *.* TO 'squirrel'@'%';
FLUSH PRIVILEGES;
```

### For GTID Support

```ini
[mysqld]
gtid_mode = ON
enforce_gtid_consistency = ON
```

---

## Backup Modes

### Logical Dump (default)

A consistent SQL dump using `FLUSH TABLES WITH READ LOCK` + `START TRANSACTION WITH CONSISTENT SNAPSHOT`.

```bash
squirrel backup mysql \
  --dsn "root:secret@tcp(localhost:3306)/" \
  --repo /backup/mysql-repo \
  --database myapp \
  --database auditdb \
  --tag daily
```

The lock is held only briefly to record the binlog position, then released. The dump proceeds from the consistent snapshot without holding the lock.

**Creates:** `mysql-dump` snapshot with `binlog_file`, `binlog_pos`, and `gtid_set` (if GTID is enabled).

### Binlog Streaming

Stream binlog events from the server using MySQL replication protocol, starting from the position recorded in the latest dump snapshot.

```bash
squirrel backup mysql \
  --dsn "root:secret@tcp(localhost:3306)/" \
  --repo /backup/mysql-repo \
  --binlog-only
```

Press **Ctrl-C** to stop. Run continuously for near-zero RPO.

**Creates:** `mysql-binlog` snapshots (one per 16 MiB of binlog data).

### Physical Backup

Copy the MySQL data directory while holding `FLUSH TABLES WITH READ LOCK`.

```bash
squirrel backup mysql \
  --dsn "root:secret@tcp(localhost:3306)/" \
  --repo /backup/mysql-repo \
  --physical \
  --data-dir /var/lib/mysql
```

!!! warning
    Physical backup holds the global read lock for the duration of the copy. Plan for downtime or use a replica.

**Creates:** `mysql-physical` snapshot.

---

## Config Profile

```yaml
profiles:
  mysql-prod:
    extends: _base-retention
    repository: s3-prod
    type: mysql
    dsn: ${sops-secrets:mysql/prod/dsn}
    databases:
      - appdb
      - auditdb
    tags: [mysql, production]
    schedule: "0 1 * * *"
    hooks:
      post-success:
        - webhook: https://monitoring.example.com/healthcheck/mysql-prod
      post-failure:
        - webhook: https://alerts.example.com/mysql-prod-backup-failed

  mysql-dev:
    repository: local
    type: mysql
    dsn: root:rootpw@tcp(localhost:3306)/
    databases:
      - devdb
    tags: [mysql, dev]
    retention:
      keep-last: 5
      prune: true
```

---

## DSN Format

```
user:password@tcp(host:port)/
user:password@tcp(host:port)/dbname
root:secret@tcp(localhost:3306)/
squirrel:pw@tcp(db.example.com:3306)/
```

---

## Restore

### SQL Restore Only

Restore the SQL dump without extracting binlogs:

```bash
squirrel restore mysql abc12345 \
  --repo /backup/mysql-repo \
  --dsn "root:secret@tcp(localhost:3306)/" \
  --sql-only
```

### SQL Restore + Binlog Extraction for PITR

```bash
squirrel restore mysql abc12345 \
  --repo /backup/mysql-repo \
  --dsn "root:secret@tcp(localhost:3306)/" \
  --binlog-dir /tmp/binlogs
```

Then replay binlogs:

```bash
# Replay all binlogs
mysqlbinlog /tmp/binlogs/*.bin | mysql -uroot -p

# PITR to specific time
mysqlbinlog --stop-datetime="2026-04-20 14:30:00" \
  /tmp/binlogs/*.bin | mysql -uroot -p

# PITR with GTID
mysqlbinlog --include-gtids="abc123:1-1000" \
  /tmp/binlogs/*.bin | mysql -uroot -p
```

### Physical Restore

```bash
squirrel restore mysql abc12345 \
  --repo /backup/mysql-repo \
  --target /var/lib/mysql \
  --innodb-recovery 1
```

After restore:

```bash
chown -R mysql:mysql /var/lib/mysql
systemctl start mysql
```

---

## Snapshot Types

| Type | Contents | Key Metadata |
|---|---|---|
| `mysql-dump` | SQL dump blobs | `binlog_file`, `binlog_pos`, `gtid_set` |
| `mysql-binlog` | Binlog event files | `segments[]`, `dump_snapshot` |
| `mysql-physical` | Data directory files | `binlog_file`, `binlog_pos`, `gtid_set`, `data_dir` |

List dump snapshots:

```bash
squirrel snapshots --repo /backup/mysql-repo --type mysql-dump
```

List binlog archives:

```bash
squirrel snapshots --repo /backup/mysql-repo --type mysql-binlog
```

---

## GTID vs Position-Based Replication

| Feature | Position-based | GTID |
|---|---|---|
| Config | Default | Requires `gtid_mode = ON` |
| Stored in snapshot | `binlog_file` + `binlog_pos` | `gtid_set` |
| Replay flexibility | Tied to specific file+position | Position-independent |
| MariaDB GTID | Uses domain_id:server_id:seq | Different format from MySQL |

squirrel stores both `binlog_pos` and `gtid_set` (when available) in the snapshot metadata.

---

## Best Practices

1. **Use GTID mode** for simpler and more reliable PITR.
2. **Run binlog streaming continuously** to minimize RPO.
3. **Back up all databases** by omitting `--database` unless you need selective backup.
4. **Test restores regularly** in a staging environment.
5. **Use `binlog_format = ROW`** for the most consistent and replayable binlog events.
6. **Use a read replica** for physical backups to avoid locking the primary.
