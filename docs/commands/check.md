# squirrel check

Verify the integrity of a repository by checking all blob checksums.

```
squirrel check --repo <url>
```

---

## Flags

| Flag | Type | Required | Description |
|---|---|---|---|
| `--repo` | string | **Yes** | Repository URL or path. |

---

## What Is Checked

1. **Index consistency** — All blob IDs in the index can be located in pack files.
2. **Blob checksums** — Each blob is decrypted and its SHA-256 is verified against the stored ID.
3. **Snapshot references** — All blobs referenced by snapshots exist in the index.

---

## Examples

```bash
squirrel check --repo /backup/myrepo
```

Expected output on success:

```
repository OK
```

Output on failure:

```
ERROR: blob a1b2c3... failed checksum verification
```

---

## Notes

- `check` reads and decrypts every blob in the repository — it can take a significant amount of time on large repositories.
- Use it periodically to detect silent data corruption or storage-level bit rot.
- `check` is read-only and does not modify the repository.
