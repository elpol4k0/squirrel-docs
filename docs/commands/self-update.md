# squirrel self-update

Update the squirrel binary to the latest release from GitHub.

```
squirrel self-update [flags]
```

---

## Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--check` | bool | false | Check for a newer version without downloading or installing. |

---

## Examples

### Update to latest

```bash
squirrel self-update
```

Sample output:

```
Current version: v1.2.0
Latest version:  v1.3.0
Downloading v1.3.0...
Updated successfully.
```

If already on the latest version:

```
squirrel is up to date (v1.3.0)
```

### Check for updates without installing

```bash
squirrel self-update --check
```

---

## Notes

- `self-update` replaces the current binary in-place. The running process is not affected; restart squirrel after updating.
- On Linux/macOS, the binary must be writable by the current user. Run with `sudo` if it is installed system-wide.
- When installed via Homebrew, update with `brew upgrade elpol4k0/tap/squirrel` instead.
- When running inside Docker, update the image instead: `docker pull ghcr.io/elpol4k0/squirrel:latest`.
