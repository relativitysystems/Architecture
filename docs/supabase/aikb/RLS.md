# AIKB Supabase — Row Level Security

**Database Verified.** Identical pattern to the Global project (see [../global/RLS.md](../global/RLS.md)): **RLS is enabled on all 8 `public` tables, with zero policies on any of them.**

| Table | RLS enabled | Policy count |
|---|---|---|
| `documents`, `knowledge_chat_messages`, `knowledge_chat_sessions`, `knowledge_chunks`, `knowledge_collections`, `knowledge_documents`, `knowledge_gaps`, `knowledge_ingestion_jobs` | true (all 8) | **0** (all 8) |

Confirmed independently via Supabase's `rls_enabled_no_policy` advisor for all 8 tables, not a sample.

## Practical effect

Same analysis as [../global/RLS.md](../global/RLS.md): deny-all for `anon`/`authenticated` in principle, moot because every client in `services/supabaseService.js` is constructed with the service-role key (both `aikbSupabase` and `globalSupabase`), which bypasses RLS entirely. Tenant isolation (`client_id` filtering) is enforced **only** at the application/SQL-function layer — most notably inside `match_knowledge_chunks` itself, which takes `match_client_id` as a parameter rather than deriving it from a session variable.

## Related documents

[DATABASE.md](DATABASE.md) · [SECURITY.md](SECURITY.md) · [../global/RLS.md](../global/RLS.md) · [../../../architecture/SECURITY.md](../../../architecture/SECURITY.md)
