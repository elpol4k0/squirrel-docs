# squirrel prune

Remove blobs that are no longer referenced by any snapshot and rebuild the repository index.

```
squirrel prune --repo <url>
```

---

## Flags

| Flag | Type | Required | Description |
|---|---|---|---|
| `--repo` | string | **Yes** | Repository URL or path. |

---

## How It Works

1. Loads all snapshot manifests.
2. Collects the set of all blob IDs referenced by any live snapshot.
3. Compares against all blobs in the repository index.
4. Deletes pack files (or portions thereof) that contain only unreferenced blobs.
5. Rebuilds and rewrites the index.

---

## When to Run

- After [`squirrel forget`](forget.md) to reclaim disk/storage space.
- Periodically as part of repository maintenance.

Alternatively, set `prune: true` in a retention policy or use `squirrel forget --prune` to combine both steps.

---

## Examples

### Manual prune

```bash
squirrel prune --repo /backup/myrepo
```

Sample output:

```
Pruning unreferenced blobs...
Removed 42 pack files (1.2 GiB freed)
Index rebuilt.
```

### Combined forget + prune

```bash
squirrel forget --repo /backup/myrepo --keep-last 7 --prune
```

---

## Notes

- `prune` acquires a repository lock to prevent concurrent modifications.
- Running `prune` concurrently with an active backup is safe — squirrel uses locks to coordinate.
- `prune` does not remove snapshots; use [`squirrel forget`](forget.md) for that.
