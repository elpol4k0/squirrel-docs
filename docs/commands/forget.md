# squirrel forget

Apply a retention policy to remove old snapshots. Optionally prune unreferenced blobs afterwards.

```
squirrel forget --repo <url> [retention flags] [flags]
```

At least one `--keep-*` flag must be specified.

---

## Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--repo` | string | — | **Required.** Repository URL or path. |
| `--keep-last` | int | — | Keep the N most recent snapshots regardless of age. |
| `--keep-hourly` | int | — | Keep the last N hours that have at least one snapshot. |
| `--keep-daily` | int | — | Keep the last N days that have at least one snapshot. |
| `--keep-weekly` | int | — | Keep the last N weeks that have at least one snapshot. |
| `--keep-monthly` | int | — | Keep the last N months that have at least one snapshot. |
| `--keep-yearly` | int | — | Keep the last N years that have at least one snapshot. |
| `--prune` | bool | false | Run `prune` automatically after removing snapshots. |
| `--dry-run` | bool | false | Show which snapshots would be removed without deleting anything. |

---

## Retention Policy Logic

Snapshots are evaluated from newest to oldest. A snapshot is kept if it is the **first** snapshot in its retention bucket that has not yet been filled:

1. **keep-last** — The N newest snapshots are always kept, regardless of other rules.
2. **keep-hourly** — For each of the last N calendar hours, keep the newest snapshot in that hour.
3. **keep-daily** — For each of the last N calendar days, keep the newest snapshot on that day.
4. **keep-weekly** — For each of the last N calendar weeks (Monday-based), keep the newest.
5. **keep-monthly** — For each of the last N calendar months, keep the newest.
6. **keep-yearly** — For each of the last N calendar years, keep the newest.

Snapshots not selected by any rule are removed.

---

## Examples

### Keep last 7 snapshots

```bash
squirrel forget --repo /backup/myrepo --keep-last 7
```

### Daily / weekly / monthly policy

```bash
squirrel forget --repo /backup/myrepo \
  --keep-daily 30 \
  --keep-weekly 12 \
  --keep-monthly 24
```

### Dry run to preview

```bash
squirrel forget --repo /backup/myrepo --keep-last 3 --dry-run
```

Sample output:

```
Would remove: b2c3d4e5  2026-04-23 02:00  files
Would remove: c3d4e5f6  2026-04-22 02:00  files
Keep:         abc12345  2026-04-25 02:00  files
Keep:         d4e5f6a7  2026-04-24 02:00  files
Keep:         e5f6a7b8  2026-04-23 03:00  files
```

### Forget and prune in one step

```bash
squirrel forget --repo /backup/myrepo \
  --keep-last 5 \
  --keep-daily 30 \
  --prune
```

### In config profile

```yaml
profiles:
  files-home:
    retention:
      keep-last: 7
      keep-daily: 30
      keep-weekly: 12
      prune: true
```

When `prune: true` is set in the profile, `squirrel run` will prune after each backup automatically.

---

## Notes

- `forget` only removes snapshot manifests. Blob data is not freed until [`squirrel prune`](prune.md) runs.
- Use `--prune` to combine both operations in a single command.
- Retention policies apply per snapshot type independently; WAL snapshots and base snapshots each have their own retention history.
