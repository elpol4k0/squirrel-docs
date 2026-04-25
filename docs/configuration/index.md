# Configuration

squirrel is configured via a YAML file. This allows you to define repositories, profiles, retention policies, hooks, and secret providers in one place and run backups with `squirrel run <profile>`.

---

## Config File Location

| Platform | Default Path |
|---|---|
| Linux | `~/.config/squirrel/config.yml` |
| macOS | `~/.config/squirrel/config.yml` |
| Windows | `%APPDATA%\squirrel\config.yml` |

Override with `--config <path>` on any command, or `SQUIRREL_CONFIG` environment variable.

---

## Top-Level Structure

```yaml
version: 1              # (required) Schema version — always 1

secrets: {}             # Named secret providers (optional)
repositories: {}        # Repository definitions
defaults: {}            # Global defaults applied to all profiles
profiles: {}            # Backup profiles
```

---

## Sections

| Section | Description | Reference |
|---|---|---|
| `secrets` | Named secret providers (Vault, SOPS, keyring, …) | [Secret Providers](secret-providers.md) |
| `repositories` | Repository URLs, passwords, backend env vars | [Repositories](repositories.md) |
| `defaults` | Retention defaults applied to all profiles | [Defaults](defaults.md) |
| `profiles` | Backup jobs with type, paths, schedule, retention, hooks | [Profiles](profiles.md) |

---

## Minimal Example

```yaml
version: 1

repositories:
  local:
    url: /backup/myrepo
    password: ${env:SQUIRREL_PASSWORD}

profiles:
  files-home:
    repository: local
    paths:
      - /home/user/documents
    schedule: "0 2 * * *"
    retention:
      keep-daily: 7
      prune: true
```

---

## Full Example

See [`config.yml.example`](https://github.com/elpol4k0/squirrel/blob/develop/config.yml.example) in the repository for a comprehensive example covering all backends, secret providers, and profile types.

---

## Managing the Config

```bash
# Create from template
squirrel config init

# Validate syntax and references
squirrel config validate

# Show resolved profile (secrets masked)
squirrel config show pg-daily

# Show with secrets revealed
squirrel config show pg-daily --reveal
```
