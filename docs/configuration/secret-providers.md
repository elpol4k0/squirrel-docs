# Secret Providers

squirrel resolves secrets at runtime using a `${provider:path}` syntax. This keeps sensitive values (passwords, tokens, DSNs) out of the config file.

---

## Inline Secret Syntax

Any config field that accepts a string can use:

```
${provider:path}
${provider:path#field}
${named-provider:key}
```

The provider is either a **built-in** (resolved directly) or a **named provider** defined in the `secrets:` section.

---

## Built-In Providers

These work without a `secrets:` entry.

### `${env:VAR}`

Read from an environment variable.

```yaml
password: ${env:SQUIRREL_PASSWORD}
dsn: ${env:PG_DSN}
```

### `${file:/path}`

Read the contents of a file (whitespace is trimmed).

```yaml
password: ${file:/etc/squirrel/repo.key}
```

### `${keyring:service/key}`

Read from the OS keyring. Service defaults to `squirrel`.

```yaml
password: ${keyring:squirrel/repo-password}
```

Store with: `squirrel secrets set repo-password`

### `${cmd:command arg1 arg2}`

Execute a command and use its stdout as the secret value.

```yaml
password: ${cmd:pass show squirrel/repo-password}
dsn: ${cmd:op read op://vault/postgres/dsn}
```

### `${op://vault/item/field}`

1Password native syntax (executed via the `op` CLI).

```yaml
password: ${op://Personal/squirrel-repo/password}
```

---

## Named Providers

Define reusable provider configurations in the `secrets:` section, then reference them as `${name:key}`.

### HashiCorp Vault

```yaml
secrets:
  vault-prod:
    type: vault
    address: https://vault.example.com
    token-from: env:VAULT_TOKEN    # or file:/path/to/token
```

Reference:

```yaml
password: ${vault-prod:secret/data/squirrel#repo_password}
dsn: ${vault-prod:secret/data/postgres#replication_dsn}
```

Path format: `<mount>/<path>#<field>` (KVv2)

#### Vault Fields

| Field | Description |
|---|---|
| `type` | `vault` |
| `address` | Vault server URL |
| `token-from` | Token source: `env:VAR` or `file:/path` |

---

### Mozilla SOPS

```yaml
secrets:
  sops-secrets:
    type: sops
    file: /etc/squirrel/secrets.enc.yaml
```

Reference:

```yaml
password: ${sops-secrets:repos/s3-prod/password}
dsn: ${sops-secrets:mysql/prod/dsn}
```

Path format: dot-separated key path into the decrypted YAML structure.

#### SOPS Fields

| Field | Description |
|---|---|
| `type` | `sops` |
| `file` | Path to the SOPS-encrypted YAML file |

Requires `sops` binary on PATH and appropriate key access (GPG, AWS KMS, age, etc.).

---

### OS Keyring

```yaml
secrets:
  keyring-prod:
    type: keyring
    service: squirrel-prod
```

Reference:

```yaml
password: ${keyring-prod:repo-password}
```

#### Keyring Fields

| Field | Description |
|---|---|
| `type` | `keyring` |
| `service` | Keyring service namespace |

---

### age

```yaml
secrets:
  age-key:
    type: age
    file: /etc/squirrel/master.age
```

Reference:

```yaml
password: ${age-key:}
```

The entire decrypted file contents are used as the secret value. The path after `:` is ignored for age.

#### age Fields

| Field | Description |
|---|---|
| `type` | `age` |
| `file` | Path to the age-encrypted file |

Requires `age` binary on PATH and `AGE_SECRET_KEY` environment variable (or SSH key).

---

### Plain File (named)

```yaml
secrets:
  file-key:
    type: file
    file: /etc/squirrel/repo.key
```

Reference:

```yaml
password: ${file-key:}
```

Equivalent to the inline `${file:/etc/squirrel/repo.key}` syntax but reusable by name.

#### File Fields

| Field | Description |
|---|---|
| `type` | `file` |
| `file` | Path to the plaintext key file |

---

## Secret Provider Comparison

| Provider | Type | Use Case |
|---|---|---|
| `env:` | Built-in | CI/CD, Docker secrets via env vars |
| `file:` | Built-in | Key files, Kubernetes secrets mounted as files |
| `keyring:` | Built-in / Named | Interactive workstations, macOS/Windows desktops |
| `cmd:` | Built-in | 1Password, pass, custom scripts |
| `op://` | Built-in | 1Password native syntax |
| `vault` | Named | Production servers with HashiCorp Vault |
| `sops` | Named | GitOps with encrypted YAML committed to source |
| `age` | Named | Simple age-encrypted secret files |
| `file` | Named | Reusable file references across multiple fields |

---

## Combining Providers

Multiple providers can be used within the same config:

```yaml
secrets:
  vault-prod:
    type: vault
    address: https://vault.example.com
    token-from: env:VAULT_TOKEN
  sops-secrets:
    type: sops
    file: /etc/squirrel/secrets.enc.yaml

repositories:
  s3-prod:
    url: s3:my-bucket/prod
    password: ${vault-prod:secret/data/squirrel#password}
    env:
      AWS_ACCESS_KEY_ID: ${sops-secrets:aws/access_key_id}
      AWS_SECRET_ACCESS_KEY: ${sops-secrets:aws/secret_access_key}
```
