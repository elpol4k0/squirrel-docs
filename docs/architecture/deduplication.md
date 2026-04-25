# Deduplication

squirrel uses content-addressed storage to automatically deduplicate identical data across files, snapshots, hosts, and time. Only unique chunks are ever stored.

---

## Content Addressing

Every blob is identified by the **SHA-256 hash of its plaintext** (before compression and encryption). This means:

- The same file backed up twice produces the same blob ID — the second backup writes nothing.
- The same content on two different machines deduplicates across hosts.
- Modified files produce different chunk IDs only for the changed chunks.

---

## Rabin CDC Chunking

squirrel uses **Rabin fingerprint Content-Defined Chunking (CDC)** to split data into variable-size chunks. Unlike fixed-size chunking, CDC is shift-resistant: inserting bytes at the beginning of a file does not invalidate all subsequent chunks.

### Parameters

| Parameter | Value |
|---|---|
| Algorithm | Rabin fingerprint (polynomial `0x3DA3358B4DC173`) |
| Minimum chunk size | 512 KiB |
| Maximum chunk size | 8 MiB |
| Average chunk size | ~4 MiB |

### How CDC Works

``` mermaid
flowchart LR
    DATA([byte stream]) --> WIN[sliding\nhash window]
    WIN --> HASH{Rabin hash\nmatches target?}
    HASH -- No --> WIN
    HASH -- Yes --> CUT[cut chunk\nboundary]
    CUT --> ID[SHA-256\nblob ID]
    ID --> DUP{exists in\nindex?}
    DUP -- Yes --> SKIP([skip upload])
    DUP -- No --> UPLOAD([compress then encrypt then upload])

    style SKIP fill:#1a1a1a,stroke:#f5a623,color:#f5a623
    style UPLOAD fill:#1a1a1a,stroke:#555
```

### Shift-Resistance

| Scenario | Fixed-size chunks | Rabin CDC |
|---|---|---|
| Insert 1 byte at start of file | All chunks shift — 0% dedup | Only ~2 boundary chunks change |
| Insert text in middle of large file | All subsequent chunks change | ~2 boundary chunks change |
| Two identical files | 100% dedup | 100% dedup |
| Two files differing by 1 byte | ~1 chunk deduplicated | Most chunks deduplicated |

---

## Deduplication Scope

``` mermaid
flowchart LR
    subgraph Sources
        H1[host A\ndaily backup]
        H2[host B\ndaily backup]
        PG[PostgreSQL\nbase backup]
        MY[MySQL\nphysical backup]
    end

    subgraph Repo["Repository (shared blob pool)"]
        BLOB1[blob aaaa...]
        BLOB2[blob bbbb...]
        BLOB3[blob cccc...]
    end

    H1 & H2 & PG & MY --> Repo
```

Deduplication works across **time**, **hosts**, **database types**, and **snapshot types** — all sharing the same blob pool.

---

## In-Session Deduplication

During a backup session, squirrel maintains a `pending` map of blob IDs being uploaded in the current session. A blob is not uploaded twice even if the persistent index has not been flushed yet.

---

## Persistent Index

The persistent index in `index/` maps blob IDs to pack file locations:

```
blob-sha256 → (pack-file-id, byte-offset, length)
```

Before uploading any chunk, squirrel checks both the in-memory `pending` map and the persistent index. If the blob ID is found in either, the upload is skipped entirely.

---

## Pack File Grouping

``` mermaid
flowchart LR
    B1[blob] & B2[blob] & B3[blob] --> PACK[pack file\n~128 MiB target]
    PACK --> UPLOAD([single upload\nto backend])
```

Grouping blobs into pack files reduces:

- S3 PUT request count (and cost)
- SFTP file count overhead
- Index size

Blobs within a pack file are stored contiguously; the pack header maps blob ID → offset for random access during restore.

---

## Measuring Deduplication

```bash
squirrel stats --repo /backup/myrepo --dedup
```

```
Storage size:  4.2 GiB
Logical size:  18.7 GiB
Dedup ratio:   4.45x
```

**Dedup ratio = Logical size / Storage size**

A ratio of 4.45× means the repository stores 18.7 GiB of data in only 4.2 GiB — a 76% reduction.
