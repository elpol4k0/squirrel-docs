# squirrel stats

Display repository statistics including snapshot count, blob count, storage size, and optional deduplication ratio.

```
squirrel stats --repo <url> [flags]
```

---

## Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--repo` | string | — | **Required.** Repository URL or path. |
| `--json` | bool | false | Output as JSON. |
| `--dedup` | bool | false | Compute logical size and dedup ratio. Requires reading all snapshot trees (slower). |

---

## Examples

### Basic stats

```bash
squirrel stats --repo /backup/myrepo
```

Sample output:

```
Snapshots:     12
Blobs:       1842
Pack files:    38
Storage size:  4.2 GiB
```

### With dedup ratio

```bash
squirrel stats --repo /backup/myrepo --dedup
```

Sample output:

```
Snapshots:     12
Blobs:       1842
Pack files:    38
Storage size:  4.2 GiB
Logical size:  18.7 GiB
Dedup ratio:   4.45x
```

A `dedup ratio` greater than 1 means data is stored more efficiently than raw; higher values indicate more deduplication.

### JSON output

```bash
squirrel stats --repo /backup/myrepo --json
```

```json
{
  "snapshots": 12,
  "blobs": 1842,
  "pack_files": 38,
  "storage_bytes": 4509715456,
  "logical_bytes": 20079869952,
  "dedup_ratio": 4.45
}
```

---

## Notes

- `--dedup` reads every snapshot tree to compute logical size, which may be slow on repositories with many large snapshots.
- Storage size reflects the actual bytes used on the backend (after compression and encryption).
- Logical size reflects the total uncompressed, undeduplicated size of all backed-up data across all snapshots.
