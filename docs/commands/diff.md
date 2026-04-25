# squirrel diff

Show the differences between two snapshots.

```
squirrel diff <snapshot-a> <snapshot-b> --repo <url> [flags]
```

---

## Flags

| Flag | Type | Required | Description |
|---|---|---|---|
| `--repo` | string | **Yes** | Repository URL or path. |
| `--stat` | bool | No | Show only a summary (counts of added/removed/modified files). |

---

## Arguments

| Argument | Description |
|---|---|
| `<snapshot-a>` | First snapshot ID or prefix (older). |
| `<snapshot-b>` | Second snapshot ID or prefix (newer). |

---

## Output

Each line shows the change status and path:

```
+ /home/user/new-file.txt
- /home/user/deleted-file.txt
M /home/user/changed-file.txt
```

| Prefix | Meaning |
|---|---|
| `+` | File added in snapshot B |
| `-` | File removed in snapshot B |
| `M` | File modified between the two snapshots |

---

## Examples

### Full diff

```bash
squirrel diff abc12345 b2c3d4e5 --repo /backup/myrepo
```

### Summary only

```bash
squirrel diff abc12345 b2c3d4e5 --repo /backup/myrepo --stat
```

Sample stat output:

```
Added:    3 files
Removed:  1 file
Modified: 12 files
```

---

## Notes

- Both snapshots must be of type `files`. Diffing database snapshots is not supported.
- Snapshot A is typically older (the baseline); snapshot B is newer (the comparison).
- Order matters: swapping A and B reverses the `+`/`-` symbols.
