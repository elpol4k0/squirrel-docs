# squirrel backup mysql

Back up a MySQL or MariaDB server via logical dump, physical data-directory copy, or binlog streaming.

```
squirrel backup mysql --repo <url> --dsn <dsn> [flags]
```

---

## Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--repo` | string | — | **Required.** Repository URL or path. |
| `--dsn` | string | — | **Required.** MySQL DSN. Can also be set via `SQUIRREL_MYSQL_DSN`. |
| `--database` | []string | all | Databases to include in the dump. Can be repeated. If omitted, all databases are backed up. |
| `--tag` | []string | — | Tags to attach to the snapshot. Can be repeated. |
| `--binlog-only` | bool | false | Stream binlog events only — skip the full dump. Requires an existing dump snapshot. |
| `--physical` | bool | false | Physical backup: copy the MySQL data directory instead of dumping SQL. |
| `--data-dir` | string | — | MySQL data directory path (required when `--physical` is set). |

---

## DSN Format

MySQL DSN in Go driver format:

```
user:password@tcp(host:port)/
user:password@tcp(host:port)/dbname
root:secret@tcp(localhost:3306)/
```

---

## Backup Modes

### Logical Dump (default)

squirrel connects to MySQL, issues `FLUSH TABLES WITH READ LOCK`, captures the current binlog position, starts a consistent transaction, and dumps all specified databases as SQL. The lock is released immediately after the binlog position is recorded.

A `mysql-dump` snapshot is created containing:
- SQL dump blobs
- `binlog_file` and `binlog_pos` (or `gtid_set` for GTID-enabled servers)

### Binlog-Only (`--binlog-only`)

Streams binlog events from the server using MySQL replication protocol, starting from the position recorded in the latest dump snapshot. Each 16 MiB of binlog data is stored as a `mysql-binlog` snapshot.

Press **Ctrl-C** to stop streaming.

### Physical Backup (`--physical`)

Acquires `FLUSH TABLES WITH READ LOCK`, copies the MySQL data directory, then releases the lock. Creates a `mysql-physical` snapshot.

!!! warning
    Physical backup requires direct filesystem access to the MySQL data directory. Typically only possible on the same host as MySQL.

---

## Examples

### Logical dump of specific databases

```bash
squirrel backup mysql \
  --dsn "root:secret@tcp(localhost:3306)/" \
  --repo /backup/mysql-repo \
  --database myapp \
  --database auditdb \
  --tag daily
```

### Backup all databases

```bash
squirrel backup mysql \
  --dsn "root:secret@tcp(localhost:3306)/" \
  --repo /backup/mysql-repo
```

### Binlog-only streaming

```bash
squirrel backup mysql \
  --dsn "root:secret@tcp(localhost:3306)/" \
  --repo /backup/mysql-repo \
  --binlog-only
```

### Physical backup

```bash
squirrel backup mysql \
  --dsn "root:secret@tcp(localhost:3306)/" \
  --repo /backup/mysql-repo \
  --physical \
  --data-dir /var/lib/mysql
```

### Config profile with GTID

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
```

---

## Snapshot Types Created

| Snapshot Type | Created When |
|---|---|
| `mysql-dump` | After each successful logical dump |
| `mysql-binlog` | After each 16 MiB binlog segment |
| `mysql-physical` | After each physical data-dir backup |

### mysql-dump metadata

```json
{
  "binlog_file": "mysql-bin.000003",
  "binlog_pos": 1234,
  "gtid_set": "abc123:1-100"
}
```

### mysql-binlog metadata

```json
{
  "segments": ["mysql-bin.000003", "mysql-bin.000004"],
  "dump_snapshot": "abc12345..."
}
```

### mysql-physical metadata

```json
{
  "binlog_file": "mysql-bin.000003",
  "binlog_pos": 1234,
  "gtid_set": "abc123:1-100",
  "data_dir": "/var/lib/mysql"
}
```

---

## MySQL / MariaDB Requirements

- MySQL 5.7 / 8.x or MariaDB 10.5+
- Binary logging enabled: `log_bin = ON`
- User privileges:
  ```sql
  GRANT RELOAD, REPLICATION SLAVE, REPLICATION CLIENT, SELECT ON *.* TO 'squirrel'@'%';
  ```
- For GTID streaming:
  ```
  gtid_mode = ON
  enforce_gtid_consistency = ON
  ```
