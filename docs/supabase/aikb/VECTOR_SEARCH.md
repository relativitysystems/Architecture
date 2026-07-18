# AIKB Supabase — Vector Search

**Database Verified** (schema/index/function) + **Code Verified** (call sites).

## Extension

`vector` (pgvector) 0.8.0, installed in the `extensions` schema. Not installed on the Global project — see [../global/DATABASE.md](../global/DATABASE.md).

## Index

```sql
CREATE INDEX knowledge_chunks_embedding_idx
  ON knowledge_chunks USING ivfflat (embedding vector_cosine_ops)
  WITH (lists = 100);
```
Cosine-distance approximate nearest-neighbor index on `knowledge_chunks.embedding`, created in `001_knowledge_base_schema.sql`. `lists = 100` is a fixed tuning parameter — appropriate for tens of thousands of rows; at 104 rows today the index provides negligible benefit over a sequential scan, but is correctly sized for expected future growth (Inference — not verified against a specific growth projection).

The legacy `documents.embedding` column has **no vector index at all** — any query against it (via the dead `match_documents()` function) would be a full sequential scan. See [../../audits/SCHEMA_DRIFT.md](../../audits/SCHEMA_DRIFT.md).

## Retrieval function

`match_knowledge_chunks` (5-arg, current) — full signature and behavior in [FUNCTIONS.md](FUNCTIONS.md). Client-scoped (`match_client_id`) and, since migration `006`, collection-scoped (`match_collection_ids`).

## Call path (Code Verified)

`services/supabaseService.js:searchChunks(clientId, queryEmbedding, { threshold, count, allowedCollectionIds })` → `aikbSupabase.rpc('match_knowledge_chunks', {...})`, always passing all 5 named parameters. Called from `services/runKnowledgeQuery.js`, the single shared pipeline used by:

- **Portal `/query`:** `allowedCollectionIds = null` — unrestricted, searches every collection (per existing `architecture/SYSTEM_OVERVIEW.md`, a portal user is already scoped to their own client's full workspace).
- **Slack `/ask` → `slack.question.requested` Inngest job:** `allowedCollectionIds` defaults to an **empty array** if the caller doesn't supply one explicitly — fail-closed (search nothing, not everything). The allow-list itself is Global's `slack_collection_access` table, managed via Relativity's `GET/PUT /api/integrations/slack/collections`. See [../../decisions/ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md](../../../decisions/ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md).

## Embedding generation

`services/openaiService.js` — `OPENAI_EMBEDDING_MODEL` (default `text-embedding-3-small`) generates the query and chunk embeddings. Not independently re-verified line-by-line this pass (**Historical/Structure Verified**).

## Related documents

[FUNCTIONS.md](FUNCTIONS.md) · [TABLES.md](TABLES.md) · [../../audits/SCHEMA_DRIFT.md](../../audits/SCHEMA_DRIFT.md) · [../../decisions/ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md](../../../decisions/ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md)
