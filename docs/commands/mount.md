# squirrel mount

Mount a snapshot as a read-only FUSE filesystem.

```
squirrel mount <snapshot-id> <mountpoint> --repo <url>
```

---

## Flags

| Flag | Type | Required | Description |
|---|---|---|---|
| `--repo` | string | **Yes** | Repository URL or path. |

---

## Arguments

| Argument | Description |
|---|---|
| `<snapshot-id>` | Snapshot ID or prefix to mount. |
| `<mountpoint>` | Directory to mount the snapshot into. Must exist and be empty. |

---

## Platform Support

| Platform | Support | Requirement |
|---|---|---|
| Linux | ✓ Native | FUSE kernel module (usually pre-installed) |
| macOS | ✓ Native | macFUSE (install from [osxfuse.github.io](https://osxfuse.github.io/)) |
| Windows | ✓ with WinFsp | [WinFsp](https://winfsp.dev/) must be installed; squirrel must be built with CGO |

!!! note "Windows CGO"
    On Windows, `mount` requires squirrel to be built **without** `CGO_ENABLED=0`. Install WinFsp first, then rebuild:
    ```powershell
    go build -o squirrel.exe .\cmd\squirrel\
    ```

---

## Examples

=== "Linux / macOS"
    ```bash
    mkdir /mnt/snapshot
    squirrel mount abc12345 /mnt/snapshot --repo /backup/myrepo

    # Browse the snapshot
    ls /mnt/snapshot

    # Unmount when done
    fusermount -u /mnt/snapshot    # Linux
    umount /mnt/snapshot           # macOS
    ```

=== "Windows"
    ```powershell
    # Mount to drive letter
    squirrel mount abc12345 S: --repo C:\backup\myrepo

    # Or to a directory
    squirrel mount abc12345 C:\mnt\snapshot --repo C:\backup\myrepo
    ```

---

## Use Cases

- **Browse a snapshot** without a full restore to verify its contents.
- **Restore individual files** by copying from the mounted filesystem.
- **Compare** a snapshot against the live filesystem.

---

## Notes

- The mounted filesystem is **read-only** — no writes are possible.
- Press **Ctrl-C** in the terminal running `mount` to unmount.
- FUSE mount decrypts blobs on-demand as files are accessed — it does not download the entire snapshot.
