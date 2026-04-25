# Architecture

squirrel is a content-addressed, encrypted backup system. This page explains the key architectural decisions and how data flows from source to storage.

---

## Data Flow

``` mermaid
flowchart TD
    A([Source data\nfiles / PostgreSQL / MySQL]) --> B[Rabin CDC\n512 KiB - 8 MiB chunks]
    B --> C[SHA-256\ncontent-addressed blob ID]
    C --> D{Already in repo?}
    D -- Yes --> SKIP([skip - deduplication])
    D -- No --> E[zstd lvl 3\ncompression]
    E --> F[AES-256-GCM\nrandom nonce per blob]
    F --> G[Pack file\ngroup blobs for upload]
    G --> H([Backend storage\nlocal / S3 / GCS / Azure / SFTP])

    style SKIP fill:#1a1a1a,stroke:#f5a623,color:#f5a623
    style A fill:#1a1a1a,stroke:#555
    style H fill:#1a1a1a,stroke:#555
```

---

## Component Overview

``` mermaid
flowchart LR
    subgraph CLI["squirrel CLI"]
        B[backup] & R[restore] & S[snapshots] & F[forget] & P[prune] & C[check] & RN[run]
    end

    CLI --> REPO

    subgraph REPO["Repo Layer"]
        IDX[blob index] & SNAP[snapshots / trees] & GC[prune / GC]
    end

    REPO --> CRYPTO

    subgraph CRYPTO["Packer / Crypto"]
        CDC[Rabin CDC] --> ZSTD[zstd] --> AES[AES-256-GCM]
    end

    CRYPTO --> BACKENDS

    subgraph BACKENDS["Storage Backends"]
        L[local] & S3[S3 / MinIO] & GCS[GCS] & AZ[Azure] & SFTP[SFTP]
    end
```

---

## Repository Layout

```
repo/
├── config           # Encrypted repo metadata (compression level, version)
├── keys/            # Password-wrapped master key copies
│   ├── <key-id-1>
│   └── <key-id-2>
├── data/            # Pack files (2-character hex prefix sharding)
│   ├── ab/
│   │   ├── ab1234...pack
│   │   └── ab5678...pack
│   └── cd/
│       └── cd9012...pack
├── index/           # Blob ID → (pack file, offset, length) mappings
│   ├── index-1
│   └── index-2
├── snapshots/       # Snapshot manifests (JSON, encrypted)
│   ├── snapshot-abc12345
│   └── snapshot-b2c3d4e5
└── locks/           # Concurrency advisory locks
```

---

## Pack File Format

``` mermaid
flowchart LR
    B0["blob_0\nencrypted"] --- B1["blob_1\nencrypted"] --- DOTS[" ... "] --- HDR["encrypted header\nblob ID to offset"] --- LEN["header_len\n4 bytes LE"]

    style B0 fill:#1a1a1a,stroke:#f5a623
    style B1 fill:#1a1a1a,stroke:#f5a623
    style HDR fill:#1a1a1a,stroke:#555
    style LEN fill:#1a1a1a,stroke:#555
    style DOTS fill:#0d0d0d,stroke:#333
```

Each encrypted region uses the format:

```
nonce(12 bytes) || AES-256-GCM ciphertext || GCM-tag(16 bytes)
```

**Read sequence to locate a blob:**

``` mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant P as Pack file
    C->>P: Read last 4 bytes to get header_len
    C->>P: Seek to file_size - 4 - header_len
    C->>P: Decrypt header, get blob ID map
    C->>P: Seek to blob offset and decrypt blob
```

---

## Snapshot Structure

``` mermaid
erDiagram
    SNAPSHOT {
        string id
        string time
        string hostname
        string type
        string tree
    }
    TREE {
        string nodes
    }
    FILE_NODE {
        string name
        string type
        int mode
        string mtime
        string content
    }
    DIR_NODE {
        string name
        string subtree
    }
    BLOB {
        string sha256_id
        bytes data
    }

    SNAPSHOT ||--|| TREE : "tree"
    TREE ||--o{ FILE_NODE : contains
    TREE ||--o{ DIR_NODE : contains
    DIR_NODE ||--|| TREE : "subtree"
    FILE_NODE ||--o{ BLOB : "content"
```

---

## Concurrency Model

``` mermaid
stateDiagram-v2
    [*] --> Idle
    Idle --> Locked : backup / prune / check
    Idle --> Reading : snapshots / restore
    Locked --> Idle : operation complete
    Reading --> Idle : done
    Locked --> Locked : concurrent lock attempt
```

---

## Further Reading

- [Encryption](encryption.md) — Key management, AES-GCM, Argon2id
- [Deduplication](deduplication.md) — Rabin CDC, SHA-256, content addressing
