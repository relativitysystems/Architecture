# AIKB Supabase — Functions & RPCs

All **Database Verified**. All three functions owned by `postgres`, none `SECURITY DEFINER`, none pin `search_path` (all three flagged `function_search_path_mutable` by the Supabase linter). All have default `EXECUTE` grants to `anon`/`authenticated`/`service_role` — **confirmed directly** via `has_function_privilege()`, not inferred.

## `match_knowledge_chunks` — 5-arg (current)

```sql
match_knowledge_chunks(
  query_embedding      vector,
  match_client_id      uuid,
  match_threshold      float8 DEFAULT 0.7,
  match_count          int    DEFAULT 5,
  match_collection_ids uuid[] DEFAULT NULL
) RETURNS TABLE(id uuid, document_id uuid, content text, metadata jsonb, similarity float8)
LANGUAGE plpgsql
```
Joins `knowledge_chunks` to `knowledge_documents` and filters by `collection_id = ANY(match_collection_ids)` when supplied; `NULL` means unrestricted. **Called by:** `services/supabaseService.js:searchChunks()`, the sole retrieval path used by both the portal `/query` route and the Slack `/ask` pipeline — always invoked with all 5 named parameters (confirmed via full-repo grep). **Migration source:** `006_knowledge_collections.sql`. **Classification: Active.**

## `match_knowledge_chunks` — 4-arg (legacy, still live)

```sql
match_knowledge_chunks(
  query_embedding vector, match_client_id uuid,
  match_threshold float8 DEFAULT 0.7, match_count int DEFAULT 5
) RETURNS TABLE(id uuid, document_id uuid, client_id uuid, chunk_index int, content text, metadata jsonb, similarity float8)
LANGUAGE sql STABLE
```
Queries `knowledge_chunks` directly with **no collection filtering** and **no join** to `knowledge_documents`. **Migration source:** `001_knowledge_base_schema.sql` — the original implementation, predating collections entirely. Migration `006` added the 5-arg version via `CREATE OR REPLACE FUNCTION` with a different parameter list, which in Postgres creates a **second, independent overload** rather than replacing this one — it was never dropped. **No caller in either repo today.** **Classification: Safe removal candidate**, with a caveat: because it's in a tracked migration (`001`), removing it live requires a companion `DROP FUNCTION` migration to keep replay/rollback consistent with the live database — see [../../audits/SCHEMA_DRIFT.md](../../audits/SCHEMA_DRIFT.md) for the full ambiguity-risk analysis (PostgREST overload resolution).

## `match_documents(query_embedding, match_count, filter)`

```sql
match_documents(
  query_embedding vector, match_count int DEFAULT NULL, filter jsonb DEFAULT '{}'
) RETURNS TABLE(id bigint, content text, metadata jsonb, similarity float8)
LANGUAGE plpgsql
```
Queries the legacy `documents` table (see [TABLES.md](TABLES.md)). Standard Supabase/pgvector-quickstart shape. **Zero references** in either repository's code or git history, on any branch. **EXECUTE granted to `anon` and `authenticated`, confirmed directly** — it is a live, callable-via-PostgREST public API surface today despite being dead code. **Classification: Safe removal candidate.**

## Related documents

[VECTOR_SEARCH.md](VECTOR_SEARCH.md) · [TABLES.md](TABLES.md) · [DATABASE.md](DATABASE.md) · [../../audits/SCHEMA_DRIFT.md](../../audits/SCHEMA_DRIFT.md) · [../global/FUNCTIONS.md](../global/FUNCTIONS.md)
