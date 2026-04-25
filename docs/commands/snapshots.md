# squirrel snapshots

List snapshots stored in a repository with optional filtering.

```
squirrel snapshots --repo <url> [flags]
```

---

## Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--repo` | string | — | **Required.** Repository URL or path. |
| `--host` | string | — | Filter snapshots by hostname. |
| `--tag` | string | — | Filter snapshots by tag. |
| `--type` | string | — | Filter by snapshot type. See [Snapshot Types](#snapshot-types) below. |
| `--latest` | bool | false | Show only the most recent snapshot (respects other filters). |
| `--json` | bool | false | Output as JSON array instead of a table. |

---

## Snapshot Types

| Type | Description |
|---|---|
| `files` | File or directory backup |
| `postgres-base` | PostgreSQL physical base backup |
| `postgres-wal` | PostgreSQL WAL segment archive |
| `mysql-dump` | MySQL/MariaDB logical SQL dump |
| `mysql-binlog` | MySQL/MariaDB binlog segment archive |
| `mysql-physical` | MySQL/MariaDB physical data-directory backup |

---

## Examples

### List all snapshots

```bash
squirrel snapshots --repo /backup/myrepo
```

Sample output:

```
ID        Time                  Host        Tags          Type     Size
────────  ────────────────────  ──────────  ────────────  ───────  ──────
abc12345  2026-04-25 02:00:01   myhost      daily,prod    files    512 MiB
b2c3d4e5  2026-04-24 02:00:02   myhost      daily,prod    files    510 MiB
c3d4e5f6  2026-04-25 03:00:00   myhost      postgres      postgres-base  2.1 GiB
```

### Filter by type

```bash
squirrel snapshots --repo /backup/pg-repo --type postgres-base
```

### Filter by tag

```bash
squirrel snapshots --repo /backup/myrepo --tag prod
```

### Most recent snapshot

```bash
squirrel snapshots --repo /backup/myrepo --latest
```

### Most recent of a specific type

```bash
squirrel snapshots --repo /backup/pg-repo --type postgres-base --latest
```

### JSON output

```bash
squirrel snapshots --repo /backup/myrepo --json
```

```json
[
  {
    "id": "abc12345...",
    "time": "2026-04-25T02:00:01Z",
    "hostname": "myhost",
    "paths": ["/home/user/data"],
    "tags": ["daily", "prod"],
    "tree": "blob-id-of-tree",
    "meta": {
      "type": "files"
    }
  }
]
```

### Use in scripts (PowerShell)

```powershell
$latest = (squirrel snapshots --repo C:\backup\repo --type postgres-base --json |
            ConvertFrom-Json)[0].id
squirrel restore postgres $latest --repo C:\backup\repo --target C:\pgdata
```

### Use in scripts (Bash)

```bash
LATEST=$(squirrel snapshots --repo /backup/pg-repo --type postgres-base --json \
         | jq -r '.[0].id')
squirrel restore postgres "$LATEST" --repo /backup/pg-repo --target /pgdata
```

---

## Snapshot Metadata Fields

Every snapshot contains:

| Field | Description |
|---|---|
| `id` | Unique content-addressed identifier |
| `time` | Creation timestamp (UTC) |
| `hostname` | Hostname of the machine that created the snapshot |
| `paths` | Paths that were backed up |
| `tags` | User-supplied tags |
| `tree` | Blob ID of the root tree |
| `meta.type` | Snapshot type |
| `meta.*` | Type-specific fields (LSN, binlog position, etc.) |
