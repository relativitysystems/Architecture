# AIKB Supabase — Triggers

**Database Verified.** **Zero triggers exist** in the `public` schema of this project (confirmed via `pg_trigger`, not just `information_schema.triggers`).

This contrasts with the Global project, which has 3 (`set_updated_at()` on `clients`/`oauth_tokens`/`folder_states` — see [../global/TRIGGERS.md](../global/TRIGGERS.md)). Every `updated_at` column in AIKB's schema (`knowledge_documents`, `knowledge_chat_sessions`, `knowledge_gaps`, `knowledge_collections`, `knowledge_ingestion_jobs`) is therefore maintained entirely by application code setting it explicitly on every write — confirmed pattern in `services/supabaseService.js` (e.g. `updated_at: new Date().toISOString()` on update calls). There is no database-level backstop if a future write path forgets to set it.

## Related documents

[DATABASE.md](DATABASE.md) · [FUNCTIONS.md](FUNCTIONS.md) · [../global/TRIGGERS.md](../global/TRIGGERS.md)
