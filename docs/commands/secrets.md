# squirrel secrets

Manage secrets stored in the OS keyring (macOS Keychain, Windows Credential Manager, GNOME Keyring / libsecret on Linux).

These secrets are used by the `${keyring:service/key}` secret provider in `config.yml`.

---

## Subcommands

### squirrel secrets set

Store a secret in the OS keyring.

```
squirrel secrets set <key> [--service <name>]
```

| Flag | Type | Default | Description |
|---|---|---|---|
| `--service` | string | `squirrel` | Keyring service namespace. |

| Argument | Description |
|---|---|
| `<key>` | Key name under which the secret is stored. |

squirrel prompts for the secret value interactively.

---

### squirrel secrets list

List all keys stored under a service in the OS keyring.

```
squirrel secrets list [--service <name>]
```

| Flag | Type | Default | Description |
|---|---|---|---|
| `--service` | string | `squirrel` | Keyring service namespace. |

---

### squirrel secrets delete

Delete a secret from the OS keyring.

```
squirrel secrets delete <key> [--service <name>]
```

| Flag | Type | Default | Description |
|---|---|---|---|
| `--service` | string | `squirrel` | Keyring service namespace. |

| Argument | Description |
|---|---|
| `<key>` | Key name to delete. |

---

## Examples

### Store a repository password

```bash
squirrel secrets set repo-password
# Enter value:
```

Then reference it in `config.yml`:

```yaml
repositories:
  local:
    url: /backup/myrepo
    password: ${keyring:squirrel/repo-password}
```

### Store with a custom service namespace

```bash
squirrel secrets set prod-password --service squirrel-prod
```

```yaml
secrets:
  keyring-prod:
    type: keyring
    service: squirrel-prod

repositories:
  prod:
    url: s3:mybucket/prod
    password: ${keyring-prod:prod-password}
```

### List all stored keys

```bash
squirrel secrets list
```

### Delete a secret

```bash
squirrel secrets delete repo-password
```

---

## Platform Notes

| Platform | Backend |
|---|---|
| macOS | Keychain |
| Windows | Credential Manager (`squirrel/<service>/<key>`) |
| Linux | GNOME Keyring (via libsecret) or KWallet |

On headless Linux servers without a keyring daemon, use the `${env:VAR}` or `${file:/path}` secret providers instead.
