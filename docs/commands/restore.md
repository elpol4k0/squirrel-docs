# squirrel restore

Restore files from a snapshot into a target directory.

```
squirrel restore <snapshot-id> --repo <url> --target <path>
```

---

## Flags

| Flag | Type | Required | Description |
|---|---|---|---|
| `--repo` | string | **Yes** | Repository URL or path. |
| `--target` | string | **Yes** | Directory to restore files into. Created if it does not exist. |

---

## Arguments

| Argument | Description |
|---|---|
| `<snapshot-id>` | Full or prefix of the snapshot ID to restore. Use [`squirrel snapshots`](snapshots.md) to find IDs. |

---

## Behavior

- Files are restored with original permissions and modification times.
- Symlinks and hardlinks are recreated.
- The target directory is created if it does not exist.
- Existing files in the target are overwritten.

---

## Examples

### Restore to a new directory

```bash
squirrel restore abc12345 --repo /backup/myrepo --target /restore/data
```

### Using a prefix

```bash
squirrel restore abc1 --repo /backup/myrepo --target /restore/data
```

### Restore from S3

```bash
squirrel restore abc12345 \
  --repo s3:mybucket/squirrel \
  --target /restore/data
```

---

## Verifying a Restore

After restoring, compare the source and target with standard tools:

=== "Linux / macOS"
    ```bash
    diff -rq /original/data /restore/data
    ```

=== "PowerShell"
    ```powershell
    Compare-Object `
      (Get-ChildItem -Recurse /original/data) `
      (Get-ChildItem -Recurse /restore/data)
    ```

---

## Notes

- Only `files`-type snapshots can be restored with this command.
- For PostgreSQL restores, see [`squirrel restore postgres`](restore-postgres.md).
- For MySQL restores, see [`squirrel restore mysql`](restore-mysql.md).
