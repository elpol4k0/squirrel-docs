# squirrel backup postgres

Perform a PostgreSQL physical base backup and/or continuous WAL streaming.

```
squirrel backup postgres --repo <url> --dsn <dsn> [flags]
```

squirrel connects via the PostgreSQL replication protocol, streams the base backup as a TAR archive, and stores the chunks as deduplicated blobs. WAL segments are captured continuously until interrupted.

---

## Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--repo` | string | — | **Required.** Repository URL or path. |
| `--dsn` | string | — | **Required.** PostgreSQL connection string (DSN). Can also be set via `SQUIRREL_PG_DSN`. |
| `--slot` | string | `squirrel` | Replication slot name. Created automatically if it does not exist. |
| `--tag` | []string | — | Tags to attach to each snapshot. Can be repeated. |
| `--wal-only` | bool | false | Stream WAL only — skip the base backup. Requires an existing base snapshot. |

---

## How It Works

### Full Backup Mode (default)

1. Opens a replication connection.
2. Calls `START_REPLICATION` / `BASE_BACKUP` to stream `base.tar` (pg_data) and any tablespace TARs.
3. Stores each TAR in deduplicated blobs; saves a `postgres-base` snapshot with `start_lsn`, `system_id`, and `timeline`.
4. Switches to WAL streaming from `start_lsn` and captures segments until **Ctrl-C**.
5. Each completed WAL segment (16 MiB by default) is stored as a `postgres-wal` snapshot.

### WAL-Only Mode (`--wal-only`)

Skips step 2–3 and starts streaming WAL from the LSN of the latest base snapshot. Use this to run a dedicated WAL archiver alongside periodic base backups.

---

## DSN Format

Standard PostgreSQL connection string or URI:

```
# URI format
postgres://user:password@host:5432/dbname?sslmode=disable

# Key=value format
host=localhost port=5432 user=squirrel password=secret dbname=postgres sslmode=disable
```

!!! warning "sslmode"
    If your PostgreSQL server does not have TLS configured (e.g. a local Docker container), add `?sslmode=disable` to the URI, otherwise the connection will fail.

---

## Replication Slot

squirrel uses a physical replication slot to track WAL progress. The slot is created automatically on the first backup if it does not exist.

Slots retain WAL on the primary — if squirrel stops streaming for a long time, WAL accumulates on disk. Drop unused slots with:

```bash
squirrel pg drop-slot --dsn <dsn> --slot squirrel
```

---

## Examples

### Full backup (base + WAL until Ctrl-C)

```bash
squirrel backup postgres \
  --dsn "postgres://squirrel:secret@localhost/postgres?sslmode=disable" \
  --repo /backup/pg-repo \
  --slot squirrel \
  --tag production
```

### WAL-only streaming

```bash
squirrel backup postgres \
  --dsn "postgres://squirrel:secret@localhost/postgres?sslmode=disable" \
  --repo /backup/pg-repo \
  --slot squirrel \
  --wal-only
```

### With S3 backend

```bash
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...

squirrel backup postgres \
  --dsn "${PG_DSN}" \
  --repo s3:mybucket/pg-backup \
  --slot prod-slot \
  --tag prod
```

### In config profile

```yaml
profiles:
  pg-prod:
    type: postgres
    repository: s3-prod
    dsn: ${vault-prod:secret/postgres#replication_url}
    slot: squirrel
    tags: [postgres, production]
    schedule: "0 3 * * *"
    hooks:
      post-success:
        - webhook: https://monitoring.example.com/healthcheck/pg-prod
```

---

## Snapshot Types Created

| Snapshot Type | Created When |
|---|---|
| `postgres-base` | After each successful base backup |
| `postgres-wal` | After each 16 MiB WAL segment is flushed |

### postgres-base metadata

```json
{
  "start_lsn": "0/5000000",
  "system_id": "7123456789012345678",
  "timeline": 1
}
```

### postgres-wal metadata

```json
{
  "slot": "squirrel",
  "timeline": 1,
  "segments": ["000000010000000000000005", "000000010000000000000006"],
  "base_snapshot": "abc12345...",
  "system_id": "7123456789012345678"
}
```

---

## PostgreSQL Requirements

- PostgreSQL **12 or later**
- A replication user:
  ```sql
  CREATE USER squirrel REPLICATION LOGIN PASSWORD 'secret';
  GRANT pg_read_all_data TO squirrel;  -- PostgreSQL 14+
  ```
- `wal_level = replica` (or `logical`) in `postgresql.conf`
- Sufficient `max_wal_senders` (at least 2)
- Sufficient `max_replication_slots` (at least 1)

### pg_hba.conf

Allow the squirrel host to connect for replication:

```
host  replication  squirrel  10.0.0.0/8  scram-sha-256
```

---

## PostgreSQL 15+ Protocol (V2)

squirrel automatically detects PostgreSQL 15+ and handles the V2 `BASE_BACKUP` streaming protocol, which adds a 1-byte type prefix to each `CopyData` message:

| Prefix byte | Meaning |
|---|---|
| `n` (0x6E) | Tablespace header — skipped |
| `d` (0x64) | TAR data payload |
| `e` (0x65) | End-of-tablespace marker — skipped |

Older PostgreSQL (≤ 14) uses the raw TAR byte stream without type prefixes. Both protocols are handled transparently.
