# AIKB Supabase — Storage

**Database Verified.**

| Bucket | Public | Created | Size limit | MIME allow-list |
|---|---|---|---|---|
| `aikb-documents` | false (private) | 2026-06-11 | none set at bucket level | none set at bucket level |

This is the platform's only storage bucket — the Global project has none (see [../global/STORAGE.md](../global/STORAGE.md)). Per the existing `AIKB/README.md`: files are accessed using the service-role key only, never from the browser; upload path convention is `uploads/<clientId>/<unique-filename>.<ext>`. Relativity's portal upload flow writes directly into this bucket (Historical, per `architecture/INGESTION_PIPELINE.md` — not re-verified line-by-line this pass).

No bucket-level size or MIME restrictions exist in Supabase itself — any limits enforced today are application-layer (`multer` `fileFilter` in Relativity, per the existing `architecture/SECURITY.md`), not a database/storage-level control.

## Related documents

[DATABASE.md](DATABASE.md) · [../global/STORAGE.md](../global/STORAGE.md) · [../../repositories/AIKB_REPO.md](../../repositories/AIKB_REPO.md)
