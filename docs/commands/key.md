# squirrel key

Manage repository keys (passwords). squirrel supports multiple passwords per repository — all wrap the same master key, so no data re-encryption is needed when adding or removing passwords.

---

## Subcommands

### squirrel key list

List all keys stored in the repository.

```
squirrel key list --repo <url>
```

| Flag | Type | Required | Description |
|---|---|---|---|
| `--repo` | string | **Yes** | Repository URL or path. |

Sample output:

```
ID        Created               Hostname
────────  ────────────────────  ──────────
a1b2c3d4  2026-04-01 10:00:00   myhost     (current)
b2c3d4e5  2026-04-10 12:30:00   myhost
```

---

### squirrel key add

Add a new password to the repository. The new password independently wraps the same master key.

```
squirrel key add --repo <url>
```

| Flag | Type | Required | Description |
|---|---|---|---|
| `--repo` | string | **Yes** | Repository URL or path. |

squirrel prompts for the **existing** password to unlock the master key, then prompts for the **new** password twice.

---

### squirrel key remove

Remove a key from the repository by its ID.

```
squirrel key remove <key-id> --repo <url>
```

| Flag | Type | Required | Description |
|---|---|---|---|
| `--repo` | string | **Yes** | Repository URL or path. |

| Argument | Description |
|---|---|
| `<key-id>` | Key ID from `squirrel key list`. |

!!! warning
    You cannot remove the key you are currently authenticated with. Keep at least one key at all times or the repository becomes inaccessible.

---

## Examples

### Add a second password

```bash
squirrel key add --repo /backup/myrepo
# Enter current password:
# Enter new password:
# Confirm new password:
```

### List keys

```bash
squirrel key list --repo /backup/myrepo
```

### Remove an old key

```bash
squirrel key remove b2c3d4e5 --repo /backup/myrepo
```

### Password rotation workflow

```bash
# 1. Add new password
squirrel key add --repo /backup/myrepo

# 2. Verify the new password works
squirrel snapshots --repo /backup/myrepo
# (enter new password when prompted)

# 3. Remove the old key
squirrel key remove <old-key-id> --repo /backup/myrepo
```

---

## Notes

- All keys wrap the same 256-bit master key — removing a key does not change the master key or require re-encryption.
- Key IDs are content-addressed (hash of the wrapped key material).
- The `(current)` marker in `key list` indicates which key was used for the current session.
