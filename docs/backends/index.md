# Storage Backends

squirrel supports multiple storage backends. The backend is selected by the URL scheme in the `url` field of a repository definition.

---

## Supported Backends

<div class="grid cards" markdown>

-   :material-harddisk: **Local Filesystem**

    ---

    Local or NFS-mounted directory. Zero dependencies, fastest for single-machine setups.

    **URL:** `/path/to/repo` or `file:///path/to/repo`

    [:octicons-arrow-right-24: Local docs](local.md)

-   :material-aws: **S3 / MinIO**

    ---

    Amazon S3 or any S3-compatible backend (MinIO, Wasabi, Backblaze B2). Multi-region, lifecycle policies.

    **URL:** `s3:bucket/prefix`

    [:octicons-arrow-right-24: S3 docs](s3.md)

-   :material-google-cloud: **Google Cloud Storage**

    ---

    Google Cloud object storage. Authenticated via ADC or service account JSON.

    **URL:** `gs:bucket/prefix`

    [:octicons-arrow-right-24: GCS docs](gcs.md)

-   :material-microsoft-azure: **Azure Blob Storage**

    ---

    Microsoft Azure Blob storage. Authenticated via connection string or managed identity.

    **URL:** `az:container/prefix`

    [:octicons-arrow-right-24: Azure docs](azure.md)

-   :material-ssh: **SFTP**

    ---

    SSH File Transfer Protocol. Works with any SSH server. Ideal for off-site backup over SSH.

    **URL:** `sftp://user@host/path`

    [:octicons-arrow-right-24: SFTP docs](sftp.md)

</div>

---

## Choosing a Backend

| Scenario | Recommended |
|---|---|
| Single machine, local disk | Local filesystem |
| Cloud-native, scalable | S3, GCS, or Azure |
| Self-hosted object storage | MinIO (S3-compatible) |
| Off-site via SSH | SFTP |
| Air-gapped / no internet | Local filesystem or SFTP |

---

## Repository Layout

All backends store the same directory structure:

```
<repo>/
├── config           # Encrypted repository configuration
├── keys/            # Password-wrapped master keys
├── data/            # Pack files (2-char hex prefix sharding)
│   ├── ab/
│   │   ├── ab1234...pack
│   │   └── ab5678...pack
│   └── cd/
├── index/           # Blob ID → pack location mappings
├── snapshots/       # Snapshot manifests (encrypted JSON)
└── locks/           # Concurrency advisory locks
```

---

## Pack File Format

Every backend stores the same internal format — **pack files** containing groups of AES-256-GCM encrypted blobs:

```
[blob_0][blob_1]…[blob_N][encrypted_header][header_len: 4 bytes LE]
```

Each encrypted region: `nonce(12 bytes) || ciphertext || GCM-tag(16 bytes)`

The header maps blob IDs → byte offsets within the pack file.

---

## Multi-Repo Strategy

Back up to multiple repositories per profile using `extends:`:

```yaml
profiles:
  pg-prod:
    repository: s3-prod
    type: postgres
    dsn: ${vault-prod:secret/postgres#dsn}
    slot: squirrel

  pg-prod-offsite:
    extends: pg-prod
    repository: sftp-offsite
    schedule: "0 5 * * 0"   # weekly offsite copy
```
