# Encryption

squirrel encrypts all data at rest using AES-256-GCM with per-blob random nonces. The master key is derived from a password using Argon2id.

---

## Master Key

### Generation

At `squirrel init`, a **256-bit (32-byte) random master key** is generated using a cryptographically secure random source. This master key is used to encrypt all blobs and metadata.

The master key itself is never stored in plaintext — only password-wrapped copies are stored in `keys/`.

### Key Wrapping

``` mermaid
flowchart TD
    PW([password + salt]) --> KDF["Argon2id\ntime=3, memory=64 MiB, threads=4"]
    KDF --> KEK[key-encryption key]
    KEK --> WRAP["AES-256-GCM\nwrap master key with KEK"]
    WRAP --> STORE([stored in keys/key-id])
```

### Adding Passwords

Each `squirrel key add` operation:

1. Decrypts the master key using the existing password.
2. Generates a new random salt.
3. Derives a new KEK from the new password + salt via Argon2id.
4. Wraps the master key with the new KEK.
5. Stores the new wrapped key in `keys/`.

**No blob re-encryption is needed** — the master key does not change.

### Password Rotation

``` mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant SQ as squirrel
    participant KD as keys dir
    U->>SQ: squirrel key add
    SQ->>KD: decrypt existing wrapped key
    SQ->>SQ: generate new salt, derive new KEK
    SQ->>KD: write new wrapped key
    U->>SQ: squirrel key remove old-id
    SQ->>KD: delete old wrapped key
```

---

## Per-Blob Encryption

Every blob (file chunk, tree, snapshot, index) is individually encrypted before storage:

``` mermaid
flowchart TD
    PT([plaintext blob]) --> ZSTD["zstd level 3\ncompression"]
    ZSTD --> NONCE["generate random\n96-bit nonce"]
    NONCE --> GCM["AES-256-GCM\nencrypt with master key"]
    GCM --> OUT(["nonce || ciphertext || GCM-tag\n12 B + N B + 16 B"])
```

### AES-256-GCM Properties

| Property | Value |
|---|---|
| Algorithm | AES-256-GCM |
| Key size | 256 bits (32 bytes) |
| Nonce size | 96 bits (12 bytes) |
| Authentication tag | 128 bits (16 bytes) |
| Nonce per blob | Randomly generated |

The GCM authentication tag provides **integrity protection** — any tampering with the ciphertext is detected during decryption.

---

## Key ID

The key ID stored in `keys/` is the SHA-256 hash of the wrapped key material. It is used to select the correct key when opening the repository with a given password.

---

## Password Verification

``` mermaid
flowchart TD
    START([open repository]) --> LIST[list all keys]
    LIST --> LOOP{for each key}
    LOOP --> DERIVE["derive KEK from\npassword + key salt"]
    DERIVE --> TRY["attempt AES-256-GCM\ndecrypt wrapped key"]
    TRY -- tag OK --> SUCCESS([master key recovered])
    TRY -- tag fail --> LOOP
    LOOP -- no keys left --> FAIL([authentication failed])
```

---

## Security Properties

| Property | Details |
|---|---|
| Confidentiality | AES-256-GCM — no plaintext stored on backend |
| Integrity | GCM authentication tag per blob |
| Password-based KDF | Argon2id — memory-hard, resistant to brute-force |
| Nonce uniqueness | Randomly generated per blob — nonce reuse is statistically negligible |
| Forward secrecy | Not provided — compromising the master key exposes all data |
| Multi-password | Multiple KEKs wrap the same master key independently |

---

## Changing the Repository Password

```bash
# Add new password
squirrel key add --repo /backup/myrepo

# Verify new password works
squirrel snapshots --repo /backup/myrepo

# Remove old password
squirrel key list --repo /backup/myrepo
squirrel key remove <old-key-id> --repo /backup/myrepo
```
