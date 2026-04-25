# squirrel restore mysql

Restore a MySQL or MariaDB backup from a dump snapshot and optionally extract binlog segments for Point-in-Time Recovery.

```
squirrel restore mysql <snapshot-id> --repo <url> [flags]
```

---

## Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--repo` | string | — | **Required.** Repository URL or path. |
| `--dsn` | string | — | MySQL DSN for executing the SQL restore. Required unless `--target` is used. |
| `--binlog-dir` | string | `/tmp/squirrel-binlogs` | Directory to extract binlog files into. |
| `--sql-only` | bool | false | Restore the SQL dump only — do not extract binlogs. |
| `--target` | string | — | Target directory for physical restore (restores data directory files). |
| `--innodb-recovery` | int | 0 | InnoDB force recovery level (1–6). Used for physical restores with corruption. |

---

## Arguments

| Argument | Description |
|---|---|
| `<snapshot-id>` | ID or prefix of the `mysql-dump` or `mysql-physical` snapshot to restore. |

---

## Restore Modes

### SQL Restore (`--sql-only`)

Executes the stored SQL dump against the target MySQL server:

```bash
squirrel restore mysql abc12345 \
  --repo /backup/mysql-repo \
  --dsn "root:secret@tcp(localhost:3306)/" \
  --sql-only
```

Use this when you only need to restore to the dump point and do not need binlog replay.

### SQL Restore + Binlog Extraction (default)

Restores the SQL dump and extracts all binlog segments captured after the dump:

```bash
squirrel restore mysql abc12345 \
  --repo /backup/mysql-repo \
  --dsn "root:secret@tcp(localhost:3306)/" \
  --binlog-dir /tmp/binlogs
```

After restore, replay the binlogs manually:

```bash
mysqlbinlog /tmp/binlogs/mysql-bin.000003 \
            /tmp/binlogs/mysql-bin.000004 \
  | mysql -uroot -p
```

For PITR to a specific point in time:

```bash
mysqlbinlog --stop-datetime="2026-04-20 14:30:00" \
  /tmp/binlogs/mysql-bin.* | mysql -uroot -p
```

### Physical Restore (`--target`)

Extracts data directory files from a `mysql-physical` snapshot:

```bash
squirrel restore mysql abc12345 \
  --repo /backup/mysql-repo \
  --target /var/lib/mysql \
  --innodb-recovery 1
```

After extraction, start MySQL normally. If there is corruption, increase `--innodb-recovery`.

---

## InnoDB Recovery Levels

| Level | Description |
|---|---|
| 0 | Normal (default) |
| 1 | Skip undo log rollback for incomplete transactions |
| 2 | Skip background threads, prevent redo log rollback |
| 3 | Skip corrupt data pages |
| 4 | Skip undo log (no rollback) |
| 5 | Skip undo log with no consistency guarantees |
| 6 | Skip redo log application |

!!! warning
    Higher recovery levels bypass data integrity checks. Use only when lower levels fail.

---

## Examples

### Restore dump and extract binlogs

```bash
squirrel restore mysql abc12345 \
  --repo s3:mybucket/mysql-backup \
  --dsn "root:secret@tcp(localhost:3306)/" \
  --binlog-dir /tmp/binlogs
```

### SQL dump only

```bash
squirrel restore mysql abc12345 \
  --repo /backup/mysql-repo \
  --dsn "root:secret@tcp(localhost:3306)/" \
  --sql-only
```

### Physical restore with InnoDB recovery

```bash
squirrel restore mysql abc12345 \
  --repo /backup/mysql-repo \
  --target /var/lib/mysql \
  --innodb-recovery 1
```

---

## After Physical Restore

```bash
# Fix ownership
chown -R mysql:mysql /var/lib/mysql

# Start MySQL
systemctl start mysql

# Check error log
journalctl -u mysql -n 50
```
