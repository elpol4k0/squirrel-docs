# Profiles

Profiles define backup jobs. Each profile specifies what to back up, where to store it, when to run, and how to handle retention and notifications.

```yaml
profiles:
  <name>:
    repository: <repo-name>
    type: files | postgres | mysql
    # ... type-specific fields
    schedule: "<cron>"
    tags: [tag1, tag2]
    retention: {}
    hooks: {}
    extends: <parent-profile>
    abstract: false
```

---

## Common Fields

| Field | Type | Description |
|---|---|---|
| `repository` | string | **Required.** Name of the repository from the `repositories:` section. |
| `type` | string | Backup type: `files`, `postgres`, or `mysql`. Defaults to `files`. |
| `schedule` | string | Cron expression or shortcut for `squirrel daemon` / `squirrel schedule`. |
| `tags` | []string | Tags attached to every snapshot created by this profile. |
| `retention` | object | Retention policy. See [Retention](retention.md). |
| `hooks` | object | Pre/post-backup hooks. See [Hooks](hooks.md). |
| `extends` | string | Inherit settings from another profile. |
| `abstract` | bool | If `true`, this profile cannot be run directly (only used as a base for inheritance). |

---

## File Backup Fields

Used when `type: files` (or type is omitted).

| Field | Type | Description |
|---|---|---|
| `paths` | []string | **Required.** Paths to back up. |
| `excludes` | []string | Glob patterns to exclude. Evaluated relative to each path. |

```yaml
profiles:
  files-home:
    repository: local
    type: files
    paths:
      - /home/user/documents
      - /home/user/projects
    excludes:
      - "**/.git"
      - "**/node_modules"
      - "**/__pycache__"
      - "**/*.tmp"
    tags: [files, home]
    schedule: "0 2 * * *"
```

---

## PostgreSQL Profile Fields

Used when `type: postgres`.

| Field | Type | Description |
|---|---|---|
| `dsn` | string | **Required.** PostgreSQL DSN. Supports `${...}` syntax. |
| `slot` | string | Replication slot name. Default: `squirrel`. |

```yaml
profiles:
  pg-prod:
    repository: s3-prod
    type: postgres
    dsn: ${vault-prod:secret/data/postgres#replication_url}
    slot: squirrel
    tags: [postgres, production]
    schedule: "0 3 * * *"
```

---

## MySQL Profile Fields

Used when `type: mysql`.

| Field | Type | Description |
|---|---|---|
| `dsn` | string | **Required.** MySQL DSN. Supports `${...}` syntax. |
| `databases` | []string | Databases to include. If omitted, all databases are backed up. |

```yaml
profiles:
  mysql-prod:
    repository: s3-prod
    type: mysql
    dsn: ${sops-secrets:mysql/prod/dsn}
    databases:
      - appdb
      - auditdb
    tags: [mysql, production]
    schedule: "0 1 * * *"
```

---

## Profile Inheritance

Profiles can inherit from other profiles using `extends:`. Child fields override parent fields; maps are merged, not replaced.

### Abstract Base Profile

```yaml
profiles:
  _base-retention:
    abstract: true
    retention:
      keep-last: 5
      keep-daily: 14
      keep-weekly: 8
      keep-monthly: 24
      keep-yearly: 5
      prune: true
```

Abstract profiles are prefixed with `_` by convention and cannot be run directly.

### Child Profile

```yaml
profiles:
  pg-prod:
    extends: _base-retention
    repository: s3-prod
    type: postgres
    dsn: ${vault-prod:secret/data/postgres#replication_url}
    slot: squirrel
    tags: [postgres, production]
    schedule: "0 3 * * *"
    hooks:
      post-success:
        - webhook: https://monitoring.example.com/healthcheck/pg-prod
```

The child inherits `retention` from `_base-retention` and adds its own `repository`, `type`, `dsn`, `slot`, `tags`, `schedule`, and `hooks`.

### Multi-Repo Offsite

A profile can extend another to back up to a second location:

```yaml
profiles:
  pg-prod-offsite:
    extends: pg-prod
    repository: sftp-offsite
    tags: [postgres, production, offsite]
    schedule: "0 5 * * 0"   # weekly on Sunday at 05:00
```

---

## Complete Profile Example

```yaml
profiles:
  files-home:
    repository: local
    paths:
      - /home/user/documents
      - /home/user/projects
    excludes:
      - "**/.git"
      - "**/node_modules"
      - "**/__pycache__"
    tags: [files, home]
    schedule: "0 2 * * *"
    retention:
      keep-last: 7
      keep-daily: 30
      keep-weekly: 12
      prune: true
    hooks:
      post-success:
        - webhook: https://monitoring.example.com/healthcheck/files-home
      post-failure:
        - command: [notify-send, "squirrel", "files-home backup FAILED"]
```

---

## Profile Naming

- Use lowercase, hyphen-separated names: `pg-prod`, `files-home`, `mysql-dev`.
- Abstract profiles are conventionally prefixed with `_`: `_base-retention`, `_base-hooks`.
- Profile names are passed to `squirrel run <name>` and exposed as `SQUIRREL_PROFILE` in hooks.
