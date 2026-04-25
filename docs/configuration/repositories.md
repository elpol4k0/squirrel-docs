# Repositories

The `repositories` section defines named repository targets that profiles reference by name.

```yaml
repositories:
  <name>:
    url: <backend-url>
    password: <secret>          # or
    password-file: <path>
    env:
      KEY: value
```

---

## Fields

| Field | Type | Description |
|---|---|---|
| `url` | string | **Required.** Backend URL or path. See [Storage Backends](../backends/index.md). |
| `password` | string | Repository password. Supports secret provider syntax `${...}`. |
| `password-file` | string | Path to a file containing the repository password. |
| `env` | map[string]string | Extra environment variables injected when accessing this repository. Values support `${...}` syntax. |

Exactly one of `password` or `password-file` must be set (or an interactive prompt is used).

---

## Backend URL Schemes

| Scheme | Example |
|---|---|
| Local filesystem | `/backup/myrepo` or `file:///backup/myrepo` |
| Amazon S3 / MinIO | `s3:bucket/prefix` |
| Google Cloud Storage | `gs:bucket/prefix` |
| Azure Blob Storage | `az:container/prefix` |
| SFTP | `sftp://user@host/path` |

---

## Examples

### Local filesystem

```yaml
repositories:
  local:
    url: /backup/squirrel-repo
    password: ${env:SQUIRREL_PASSWORD}
```

### Amazon S3

```yaml
repositories:
  s3-prod:
    url: s3:my-bucket/squirrel/prod
    password: ${sops-secrets:repos/s3-prod/password}
    env:
      AWS_ACCESS_KEY_ID: ${env:AWS_ACCESS_KEY_ID}
      AWS_SECRET_ACCESS_KEY: ${env:AWS_SECRET_ACCESS_KEY}
      AWS_DEFAULT_REGION: eu-central-1
```

### MinIO (S3-compatible)

```yaml
repositories:
  minio:
    url: s3:squirrel-backups/prod
    password: ${env:SQUIRREL_PASSWORD}
    env:
      AWS_ACCESS_KEY_ID: minioadmin
      AWS_SECRET_ACCESS_KEY: ${env:MINIO_SECRET}
      AWS_ENDPOINT_URL: http://minio.example.com:9000
```

### Azure Blob Storage

```yaml
repositories:
  azure:
    url: az:mycontainer/squirrel
    password: ${vault-prod:secret/squirrel#azure_repo_password}
    env:
      AZURE_STORAGE_ACCOUNT: mystorageaccount
      AZURE_STORAGE_KEY: ${env:AZURE_STORAGE_KEY}
```

### Google Cloud Storage

```yaml
repositories:
  gcs:
    url: gs:my-gcs-bucket/squirrel
    password: ${keyring-prod:repo-password}
    env:
      GOOGLE_APPLICATION_CREDENTIALS: /etc/squirrel/gcs-sa.json
```

### SFTP with key file

```yaml
repositories:
  sftp-offsite:
    url: sftp://backup@offsite.example.com/backups/squirrel
    password-file: /etc/squirrel/sftp-repo.key
```

### Multiple repositories

```yaml
repositories:
  primary:
    url: s3:primary-bucket/squirrel
    password: ${env:SQUIRREL_PASSWORD}
    env:
      AWS_ACCESS_KEY_ID: ${env:AWS_PRIMARY_KEY}
      AWS_SECRET_ACCESS_KEY: ${env:AWS_PRIMARY_SECRET}

  offsite:
    url: sftp://backup@offsite.example.com/backups/squirrel
    password: ${env:SQUIRREL_PASSWORD}
```

---

## Notes

- Repository names must be unique within the config file.
- `env` values are injected into the process environment before the backend driver initializes.
- Secret references in `env` values are resolved before injection.
- The same physical repository can be listed under multiple names with different credentials.
