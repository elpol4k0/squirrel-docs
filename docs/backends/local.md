# Local Filesystem

Store repository data in a local directory or any mounted filesystem (NFS, SMB, external drive).

---

## URL Format

```
/absolute/path/to/repo
file:///absolute/path/to/repo
```

---

## Configuration

```yaml
repositories:
  local:
    url: /backup/squirrel-repo
    password: ${env:SQUIRREL_PASSWORD}
```

No additional environment variables are required.

---

## Examples

### Standard local backup

```yaml
repositories:
  local:
    url: /backup/myrepo
    password: ${env:SQUIRREL_PASSWORD}
```

### External drive

```yaml
repositories:
  external:
    url: /mnt/backup-drive/squirrel
    password: ${file:/etc/squirrel/repo.key}
```

### Windows

```yaml
repositories:
  local:
    url: D:\Backup\squirrel-repo
    password: ${env:SQUIRREL_PASSWORD}
```

---

## Atomicity

The local backend writes blobs to a temporary file first, then renames it atomically into the final location. This prevents partial writes from corrupting the repository in case of a crash.

---

## Notes

- The directory is created automatically on `squirrel init` if it does not exist.
- On NFS mounts, ensure the mount supports atomic rename operations.
- Concurrent access from multiple hosts is supported via advisory locks in `locks/`.
- For network-attached storage with poor NFS semantics, prefer SFTP or an object storage backend.
