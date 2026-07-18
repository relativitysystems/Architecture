# Global Supabase — Storage

**Database Verified.** `storage.buckets` is empty in this project — **no buckets exist**.

All platform file storage lives in the AIKB project's `aikb-documents` bucket instead. Relativity's upload flow (`POST /api/knowledge/upload` → `services/aikbService.js` per the existing `architecture/INGESTION_PIPELINE.md`) writes directly to that bucket using AIKB's storage credentials (`AIKB_SUPABASE_URL`/`AIKB_SUPABASE_SERVICE_ROLE_KEY`, held in Relativity's own config per `config/index.js`), not a Global-project bucket.

See [../aikb/STORAGE.md](../aikb/STORAGE.md) for the bucket itself.

## Related documents

[DATABASE.md](DATABASE.md) · [../aikb/STORAGE.md](../aikb/STORAGE.md)
