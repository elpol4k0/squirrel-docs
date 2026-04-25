# Azure Blob Storage

Store repository data in a Microsoft Azure Blob Storage container.

---

## URL Format

```
az:container/prefix
az:account/container/prefix
```

---

## Configuration

```yaml
repositories:
  azure:
    url: az:mycontainer/squirrel
    password: ${env:SQUIRREL_PASSWORD}
    env:
      AZURE_STORAGE_ACCOUNT: mystorageaccount
      AZURE_STORAGE_KEY: ${env:AZURE_STORAGE_KEY}
```

---

## Authentication

### Storage Account Key (recommended for automation)

```yaml
repositories:
  azure:
    url: az:mycontainer/squirrel
    password: ${vault-prod:secret/squirrel#azure_repo_password}
    env:
      AZURE_STORAGE_ACCOUNT: mystorageaccount
      AZURE_STORAGE_KEY: ${env:AZURE_STORAGE_KEY}
```

Retrieve the key from the Azure Portal under **Storage Account → Access Keys**.

### Connection String

```yaml
repositories:
  azure:
    url: az:mycontainer/squirrel
    password: ${env:SQUIRREL_PASSWORD}
    env:
      AZURE_STORAGE_CONNECTION_STRING: ${env:AZURE_STORAGE_CONNECTION_STRING}
```

### Managed Identity (Azure VMs / AKS)

When running on an Azure VM or AKS pod with a Managed Identity, credentials are resolved automatically from the instance metadata service. No key or connection string is needed.

---

## Environment Variables

| Variable | Description |
|---|---|
| `AZURE_STORAGE_ACCOUNT` | Storage account name |
| `AZURE_STORAGE_KEY` | Storage account access key |
| `AZURE_STORAGE_CONNECTION_STRING` | Full connection string (alternative to account + key) |

---

## Notes

- The container must exist before running `squirrel init`.
- Enable **Blob soft delete** and **Container soft delete** in the storage account for extra protection.
- Azure Blob Storage supports **immutable storage** (WORM policies) for compliance requirements.
- squirrel uses Block Blobs with append semantics for pack files.
