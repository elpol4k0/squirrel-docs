# squirrel backup

Back up one or more files or directories into a squirrel repository.

```
squirrel backup --repo <url> --path <path> [flags]
```

Files are chunked with Rabin CDC, compressed with zstd, encrypted with AES-256-GCM, and stored as content-addressed blobs. Identical content is deduplicated automatically.

---

## Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--repo` | string | — | **Required.** Repository URL or path. |
| `--path` | string | — | **Required.** File or directory to back up. Can be specified multiple times. |
| `--tag` | []string | — | Tags to attach to the snapshot. Can be repeated: `--tag prod --tag nightly`. |
| `--dry-run` | bool | false | Show what would be uploaded without writing any data. |
| `--parallel` | int | 0 (= CPU count) | Number of parallel file upload workers. |

---

## Excludes

Exclude patterns are specified per-profile in the config file using the `excludes` field. They are evaluated as glob patterns relative to the backup path.

Common excludes:

```yaml
profiles:
  my-backup:
    excludes:
      - "**/.git"
      - "**/node_modules"
      - "**/__pycache__"
      - "**/*.tmp"
      - "**/*.log"
```

---

## Examples

### Basic backup

```bash
squirrel backup --repo /backup/myrepo --path /home/user/documents
```

### Multiple paths

```bash
squirrel backup --repo /backup/myrepo \
  --path /home/user/documents \
  --path /etc \
  --path /var/www
```

### With tags

```bash
squirrel backup --repo /backup/myrepo \
  --path /home/user \
  --tag daily \
  --tag prod
```

### Dry run

```bash
squirrel backup --repo /backup/myrepo --path /home/user --dry-run
```

Expected output:
```
[dry-run] No data was written.
```

### Parallel uploads

```bash
squirrel backup --repo s3:mybucket/repo --path /data --parallel 8
```

---

## Snapshot Output

A successful backup prints a snapshot ID and statistics:

```
Snapshot a1b2c3d4 saved  (128 blobs, 512 MiB, 14 MiB new)
```

- **blobs** — total blobs in this snapshot
- **MiB** — total logical size
- **new** — bytes actually uploaded (rest was deduplicated)

---

## Deduplication

On subsequent backups of the same or similar data, squirrel checks the in-session `pending` map and the persistent index before uploading a blob. Only new or changed content is transmitted.

```
Snapshot b2c3d4e5 saved  (128 blobs, 512 MiB, 0 MiB new)
#                                                ^^^^^^^^^^
#                                         fully deduplicated
```

---

## Notes

- The snapshot contains the hostname, timestamp, paths, tags, and a tree of blob IDs.
- File permissions and modification times are preserved.
- Symlinks and hardlinks are stored correctly.
- The snapshot ID is deterministic based on content — identical backups produce the same tree.
