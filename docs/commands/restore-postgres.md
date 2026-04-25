# squirrel restore postgres

Restore a PostgreSQL base backup and WAL segments to a target data directory, optionally configuring Point-in-Time Recovery (PITR).

```
squirrel restore postgres <snapshot-id> --repo <url> --target <path> [flags]
```

---

## Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--repo` | string | — | **Required.** Repository URL or path. |
| `--target` | string | — | **Required.** Target PostgreSQL data directory (`PGDATA`). |
| `--pitr` | string | — | Point-in-time recovery timestamp. Format: `"2026-04-20 14:30:00"`. |
| `--pitr-lsn` | string | — | Point-in-time recovery target LSN. Format: `"0/5000028"`. |
| `--wal-dir` | string | auto | Directory to extract WAL segments into. Defaults to a temp directory. |
| `--verify` | bool | false | Verify WAL segment coverage after restore. Prints segment range. |

---

## Arguments

| Argument | Description |
|---|---|
| `<snapshot-id>` | ID or prefix of the `postgres-base` snapshot to restore. |

---

## How It Works

1. Loads the `postgres-base` snapshot and extracts `base.tar` into `--target`.
2. Rewrites tablespace symlinks to point to locally extracted copies.
3. Collects all `postgres-wal` snapshots created after the base backup (matching `system_id` and `timeline`).
4. Extracts WAL segments into `--wal-dir`.
5. Writes `recovery.signal` and appends to `postgresql.auto.conf`:
   - `restore_command = 'cp <wal-dir>/%f %p'`
   - `recovery_target_time` (if `--pitr` is set)
   - `recovery_target_lsn` (if `--pitr-lsn` is set)
   - `recovery_target_action = 'promote'`
6. Optionally verifies WAL segment count and range.

Start PostgreSQL normally — it will enter recovery mode automatically.

---

## Examples

### Restore to latest (no PITR)

```bash
squirrel restore postgres abc12345 \
  --repo /backup/pg-repo \
  --target /var/lib/postgresql/17/main
```

### Restore with PITR timestamp

```bash
squirrel restore postgres abc12345 \
  --repo s3:mybucket/pg-backup \
  --target /var/lib/postgresql/17/main \
  --pitr "2026-04-20 14:30:00" \
  --verify
```

### Restore with PITR LSN

```bash
squirrel restore postgres abc12345 \
  --repo s3:mybucket/pg-backup \
  --target /var/lib/postgresql/17/main \
  --pitr-lsn "0/5000028"
```

### Custom WAL directory

```bash
squirrel restore postgres abc12345 \
  --repo /backup/pg-repo \
  --target /var/lib/postgresql/17/main \
  --wal-dir /mnt/wal-archive
```

---

## After Restore — Starting PostgreSQL

```bash
# Fix ownership
chown -R postgres:postgres /var/lib/postgresql/17/main

# Start PostgreSQL
systemctl start postgresql@17-main

# Watch recovery progress
tail -f /var/log/postgresql/postgresql-17-main.log
```

PostgreSQL will:
1. Enter recovery mode (detects `recovery.signal`)
2. Replay WAL segments via `restore_command`
3. Stop at `recovery_target_time` or `recovery_target_lsn` (if set)
4. Promote to primary (removes `recovery.signal`)

---

## `--verify` Output

```
WAL verified: 42 segment(s), range 000000010000000000000005 .. 00000001000000000000002E
```

If no WAL segments are found, `--verify` returns an error.

---

## Tablespace Handling

If the original PostgreSQL server had tablespaces, squirrel:

1. Extracts each tablespace TAR into `<target>/pg_tblspc_data/<oid>/`
2. Replaces the symlink in `<target>/pg_tblspc/<oid>` to point to the local copy

This allows restore to a host with a different filesystem layout than the original server.

---

## Generated Configuration

squirrel appends the following to `postgresql.auto.conf`:

```ini
# Written by squirrel
restore_command = 'cp /path/to/wal/%f %p'
recovery_target_time = '2026-04-20 14:30:00'   # only when --pitr is set
recovery_target_action = 'promote'              # only when --pitr or --pitr-lsn is set
```

And creates an empty `recovery.signal` file in the data directory.
