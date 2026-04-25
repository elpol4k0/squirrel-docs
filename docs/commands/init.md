# squirrel init

Initialize a new squirrel repository at the given location.

```
squirrel init --repo <url>
```

`init` creates the directory structure, generates a 256-bit master key, wraps it with Argon2id using the password you supply, and stores the result in `keys/`. The repository is ready to use immediately.

---

## Flags

| Flag | Type | Required | Description |
|---|---|---|---|
| `--repo` | string | **Yes** | Repository URL or path. See [Storage Backends](../backends/index.md) for supported URL schemes. |

---

## Repository Structure Created

```
<repo>/
├── config        # Encrypted repository metadata
├── keys/         # One or more password-wrapped master keys
├── data/         # Pack files containing encrypted blobs
├── index/        # Blob ID → pack location mappings
├── snapshots/    # Snapshot manifests
└── locks/        # Concurrency locks
```

---

## Examples

=== "Local"
    ```bash
    squirrel init --repo /backup/myrepo
    ```

=== "S3"
    ```bash
    export AWS_ACCESS_KEY_ID=...
    export AWS_SECRET_ACCESS_KEY=...
    squirrel init --repo s3:mybucket/squirrel
    ```

=== "SFTP"
    ```bash
    squirrel init --repo sftp://backup@offsite.example.com/backups/myrepo
    ```

=== "Azure"
    ```bash
    export AZURE_STORAGE_ACCOUNT=myaccount
    export AZURE_STORAGE_KEY=...
    squirrel init --repo az:mycontainer/squirrel
    ```

---

## Password Entry

squirrel prompts for a password interactively. The password is used to derive a key-encryption key (KEK) via Argon2id (time=3, mem=64 MiB, threads=4), which wraps the master key.

To automate this (e.g. in scripts), source the password from a secret provider in `config.yml`:

```yaml
repositories:
  prod:
    url: s3:mybucket/prod
    password: ${env:SQUIRREL_PASSWORD}
```

See [Secret Providers](../configuration/secret-providers.md) for all options.

---

## Notes

- The target path must be empty or non-existent; `init` will not overwrite an existing repository.
- Multiple passwords can be added later with [`squirrel key add`](key.md).
- The master key never leaves the repository in plaintext — only password-wrapped copies are stored.
