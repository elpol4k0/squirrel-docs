# Google Cloud Storage

Store repository data in a Google Cloud Storage bucket.

---

## URL Format

```
gs:bucket/prefix
```

---

## Configuration

```yaml
repositories:
  gcs:
    url: gs:my-gcs-bucket/squirrel
    password: ${env:SQUIRREL_PASSWORD}
    env:
      GOOGLE_APPLICATION_CREDENTIALS: /etc/squirrel/gcs-sa.json
```

---

## Authentication

GCS uses Application Default Credentials (ADC). Credentials are resolved in this order:

1. **`GOOGLE_APPLICATION_CREDENTIALS`** — Path to a service account JSON key file.
2. **gcloud CLI** — Credentials from `gcloud auth application-default login`.
3. **Metadata server** — Automatically used on GCE/GKE/Cloud Run.

### Service Account Key File

```yaml
repositories:
  gcs:
    url: gs:my-gcs-bucket/squirrel
    password: ${env:SQUIRREL_PASSWORD}
    env:
      GOOGLE_APPLICATION_CREDENTIALS: /etc/squirrel/gcs-sa.json
```

Create a service account with `Storage Object Admin` on the bucket:

```bash
gcloud iam service-accounts create squirrel-backup \
  --display-name "squirrel backup"

gsutil iam ch \
  serviceAccount:squirrel-backup@PROJECT.iam.gserviceaccount.com:roles/storage.objectAdmin \
  gs://my-gcs-bucket

gcloud iam service-accounts keys create /etc/squirrel/gcs-sa.json \
  --iam-account squirrel-backup@PROJECT.iam.gserviceaccount.com
```

### Workload Identity (GKE)

When running on GKE with Workload Identity configured, no `GOOGLE_APPLICATION_CREDENTIALS` is needed — the metadata server provides credentials automatically.

---

## Notes

- The bucket must exist before running `squirrel init`.
- Enable Uniform bucket-level access on the bucket for simpler IAM management.
- GCS Object Versioning can be enabled for extra protection.
- squirrel uploads blobs using standard GCS Object API (no multipart for small blobs, resumable uploads for large pack files).
