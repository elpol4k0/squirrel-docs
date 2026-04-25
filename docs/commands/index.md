# Commands

squirrel uses a hierarchical command structure: `squirrel <command> [subcommand] [flags]`

All commands that open a repository require either an interactive password prompt or a secret resolved from config.

---

## Global Flags

| Flag | Type | Description |
|---|---|---|
| `--help` | bool | Show help for the command |
| `--version` | bool | Print version information |

---

## Command Groups

<div class="grid cards" markdown>

-   **Repository**

    ---

    | Command | Description |
    |---|---|
    | [`init`](init.md) | Initialize a new encrypted repository |
    | [`check`](check.md) | Verify repository integrity |
    | [`stats`](stats.md) | Repository statistics and dedup ratio |
    | [`prune`](prune.md) | Remove unreferenced blobs |

-   **Backup**

    ---

    | Command | Description |
    |---|---|
    | [`backup`](backup.md) | Back up files or directories |
    | [`backup postgres`](backup-postgres.md) | PostgreSQL base backup + WAL |
    | [`backup mysql`](backup-mysql.md) | MySQL/MariaDB logical or physical backup |

-   **Restore**

    ---

    | Command | Description |
    |---|---|
    | [`restore`](restore.md) | Restore files from a snapshot |
    | [`restore postgres`](restore-postgres.md) | Restore PostgreSQL base + WAL |
    | [`restore mysql`](restore-mysql.md) | Restore MySQL dump or physical backup |

-   **Snapshots**

    ---

    | Command | Description |
    |---|---|
    | [`snapshots`](snapshots.md) | List snapshots |
    | [`forget`](forget.md) | Apply retention policy |
    | [`diff`](diff.md) | Diff between two snapshots |
    | [`mount`](mount.md) | Mount snapshot as FUSE filesystem |

-   **Keys & Secrets**

    ---

    | Command | Description |
    |---|---|
    | [`key`](key.md) | Add / remove / list keys |
    | [`secrets`](secrets.md) | Manage OS keyring secrets |

-   **Automation**

    ---

    | Command | Description |
    |---|---|
    | [`config`](config.md) | Init / validate / show config |
    | [`run`](run.md) | Execute config profiles |
    | [`daemon`](daemon.md) | Run as background scheduler |
    | [`schedule`](schedule.md) | Manage OS-level schedules |

-   **Utilities**

    ---

    | Command | Description |
    |---|---|
    | [`pg`](pg.md) | PostgreSQL utilities |
    | [`self-update`](self-update.md) | Update squirrel to latest release |
    | [`completion`](completion.md) | Generate shell completions |

</div>

---

## Snapshot ID Prefixes

All commands that accept a snapshot ID accept either the full 64-character SHA-256 ID or a unique prefix (e.g. `abc12345`).
