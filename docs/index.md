# squirrel

**Database-aware backup for PostgreSQL and MySQL/MariaDB.**  
Content-addressed, AES-256-GCM encrypted, and deduplicated — with first-class support for WAL streaming, binlog capture, and physical base backups.

---

## Features

<div class="grid cards" markdown>

-   :material-database-lock: **PostgreSQL**

    ---

    Physical base backup via the replication protocol + continuous WAL streaming. PITR to any point in time. No `pg_dump`, no superuser.

    [:octicons-arrow-right-24: PostgreSQL docs](databases/postgres.md)

-   :material-database: **MySQL / MariaDB**

    ---

    Logical SQL dump + binlog streaming (position and GTID). Physical data-dir backup. Restore to any binlog position.

    [:octicons-arrow-right-24: MySQL docs](databases/mysql.md)

-   :material-shield-lock: **Encryption**

    ---

    AES-256-GCM per blob with a random nonce. Argon2id key derivation. Multi-password key management. Zero plaintext on the backend.

    [:octicons-arrow-right-24: Encryption](architecture/encryption.md)

-   :material-content-duplicate: **Deduplication**

    ---

    Rabin CDC variable-size chunking (512 KiB – 8 MiB). SHA-256 content addressing. Dedup across hosts, time, and database types.

    [:octicons-arrow-right-24: Deduplication](architecture/deduplication.md)

-   :material-cloud-upload: **Storage Backends**

    ---

    Local filesystem, S3/MinIO, Azure Blob Storage, Google Cloud Storage, SFTP. Pluggable — one config, any backend.

    [:octicons-arrow-right-24: Backends](backends/index.md)

-   :material-key-variant: **Secret Providers**

    ---

    env, file, OS keyring, HashiCorp Vault, Mozilla SOPS, age, 1Password, arbitrary shell commands. Secrets never in plaintext config.

    [:octicons-arrow-right-24: Secret Providers](configuration/secret-providers.md)

-   :material-clock-outline: **Scheduling & Daemon**

    ---

    Built-in cron daemon with Prometheus metrics. Native OS integration via systemd, launchd, and Windows Task Scheduler.

    [:octicons-arrow-right-24: Scheduling](operations/scheduling.md)

-   :material-folder-network: **FUSE Mount**

    ---

    Mount any snapshot as a read-only filesystem. Browse and copy individual files without a full restore.

    [:octicons-arrow-right-24: mount command](commands/mount.md)

</div>

---

## Commands

<div class="grid cards" markdown>

-   **Repository**

    ---

    [`init`](commands/init.md) · [`check`](commands/check.md) · [`stats`](commands/stats.md) · [`prune`](commands/prune.md) · [`key`](commands/key.md)

-   **Backup**

    ---

    [`backup`](commands/backup.md) · [`backup postgres`](commands/backup-postgres.md) · [`backup mysql`](commands/backup-mysql.md)

-   **Restore**

    ---

    [`restore`](commands/restore.md) · [`restore postgres`](commands/restore-postgres.md) · [`restore mysql`](commands/restore-mysql.md)

-   **Snapshots**

    ---

    [`snapshots`](commands/snapshots.md) · [`forget`](commands/forget.md) · [`diff`](commands/diff.md) · [`mount`](commands/mount.md)

-   **Configuration**

    ---

    [`config`](commands/config.md) · [`run`](commands/run.md) · [`secrets`](commands/secrets.md)

-   **Daemon**

    ---

    [`daemon`](commands/daemon.md) · [`schedule`](commands/schedule.md) · [`pg`](commands/pg.md)

</div>

---

## Quick Start

```bash
# Initialize a repository
squirrel init --repo /backup/myrepo

# Back up a directory
squirrel backup /home/user --repo /backup/myrepo

# Back up PostgreSQL (base + WAL)
squirrel backup postgres \
  --dsn "postgres://squirrel:secret@localhost/postgres" \
  --repo /backup/pg-repo --slot squirrel

# List snapshots
squirrel snapshots --repo /backup/myrepo

# Restore a snapshot
squirrel restore abc12345 --repo /backup/myrepo --target /restore/dir
```

[:octicons-arrow-right-24: Full Quickstart](quickstart.md)

---

## Architecture

``` mermaid
flowchart LR
    SRC([files / PostgreSQL / MySQL]) --> CDC[Rabin CDC\nchunking]
    CDC --> SHA[SHA-256\nblob ID]
    SHA --> DEDUP{in repo?}
    DEDUP -- yes --> SKIP([skip])
    DEDUP -- no --> ZSTD[zstd\ncompression]
    ZSTD --> AES[AES-256-GCM\nencryption]
    AES --> PACK[pack file]
    PACK --> BE([local / S3 / GCS / Azure / SFTP])

    style SKIP fill:#1a1a1a,stroke:#f5a623,color:#f5a623
```

[:octicons-arrow-right-24: Architecture overview](architecture/index.md)
