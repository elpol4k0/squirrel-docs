# SFTP

Store repository data on a remote server over SSH File Transfer Protocol (SFTP).

---

## URL Format

```
sftp://user@host/absolute/path
sftp://user:password@host/absolute/path
sftp://user@host:port/path
```

---

## Configuration

```yaml
repositories:
  sftp-offsite:
    url: sftp://backup@offsite.example.com/backups/squirrel
    password-file: /etc/squirrel/sftp-repo.key
```

---

## Authentication

### SSH Key (recommended)

squirrel uses the SSH key at `~/.ssh/id_rsa` (or `~/.ssh/id_ed25519`) by default.

```yaml
repositories:
  sftp-offsite:
    url: sftp://backup@offsite.example.com/backups/squirrel
    password: ${env:SQUIRREL_PASSWORD}
```

No SSH password is needed if the key is accepted by the server.

### ssh-agent

If `SSH_AUTH_SOCK` is set, squirrel uses the running ssh-agent automatically.

### Password in URL

For simple setups (not recommended for production):

```yaml
repositories:
  sftp-dev:
    url: sftp://backup:sshpassword@dev.example.com/backups/squirrel
    password: ${env:SQUIRREL_PASSWORD}
```

### Custom Port

```yaml
repositories:
  sftp-offsite:
    url: sftp://backup@offsite.example.com:2222/backups/squirrel
    password-file: /etc/squirrel/sftp-repo.key
```

---

## Server Setup

On the remote server, create a dedicated backup user:

```bash
useradd -m -s /bin/bash backup
mkdir -p /backups/squirrel
chown backup:backup /backups/squirrel

# Add squirrel host's public key
mkdir -p /home/backup/.ssh
cat squirrel-host.pub >> /home/backup/.ssh/authorized_keys
chmod 600 /home/backup/.ssh/authorized_keys
```

The backup user only needs read/write access to the backup directory — no shell access is required (can use `rssh` or `scponly` as the shell).

---

## Notes

- The remote directory must exist before running `squirrel init`, or the backup user must have permission to create it.
- squirrel uses write-then-rename for atomic operations over SFTP.
- SFTP is typically slower than object storage due to per-file round-trip overhead; consider parallelism settings accordingly.
- SSH host key verification is performed using `~/.ssh/known_hosts`.
