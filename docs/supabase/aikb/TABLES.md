# AIKB Supabase — Table Reference

All data **Database Verified**. Row counts are point-in-time (2026-07-18). All 8 tables have RLS **enabled with zero policies** (see [RLS.md](RLS.md)); all grant full `arwdDxtm` to `anon`/`authenticated`/`service_role` (Supabase default).

---

### `knowledge_documents`
**Purpose:** one row per ingested source file. **Rows:** 16. **Migration source:** `001_knowledge_base_schema.sql`, `storage_path` added in `002`, `collection_id` added in `006`.

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | uuid | NO | gen_random_uuid() |
| client_id | uuid | NO | — |
| source_provider | text | NO | — |
| source_file_id | text | NO | — |
| file_name | text | NO | — |
| mime_type | text | YES | — |
| content_hash | text | YES | — |
| status | text | NO | 'indexing' |
| error_message | text | YES | — |
| last_indexed_at | timestamptz | YES | — |
| created_at | timestamptz | NO | now() |
| updated_at | timestamptz | NO | now() |
| storage_path | text | YES | — |
| collection_id | uuid | NO | — |

Indexes: PK; unique `(client_id, source_provider, source_file_id)` (dedup); index on `client_id`, `collection_id`. FK: `collection_id → knowledge_collections` (RESTRICT — confirmed live, `fk_knowledge_documents_collection`). Referenced by: `knowledge_chunks.document_id` (CASCADE), `knowledge_ingestion_jobs.document_id` (SET NULL, no FK constraint found — see below). **Performance advisor:** `knowledge_docs_collection_idx` flagged unused.

---

### `knowledge_chunks`
**Purpose:** chunked, embedded text — the retrieval unit. **Rows:** 104. **Migration source:** `001_knowledge_base_schema.sql`.

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | uuid | NO | gen_random_uuid() |
| document_id | uuid | NO | — |
| client_id | uuid | NO | — |
| chunk_index | integer | NO | — |
| content | text | NO | — |
| embedding | vector | YES | — |
| metadata | jsonb | YES | '{}' |
| created_at | timestamptz | NO | now() |

Indexes: PK; unique `(document_id, chunk_index)`; index on `client_id`, `document_id`; **`ivfflat (embedding vector_cosine_ops) WITH (lists=100)`** — the vector similarity index, see [VECTOR_SEARCH.md](VECTOR_SEARCH.md). FK: `document_id → knowledge_documents` (CASCADE).

---

### `knowledge_ingestion_jobs`
**Purpose:** queue/status tracking for the Inngest ingestion pipeline. **Rows:** 17. **Migration source:** `001_knowledge_base_schema.sql`.

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | uuid | NO | gen_random_uuid() |
| client_id | uuid | NO | — |
| source_file_id | text | NO | — |
| document_id | uuid | YES | — |
| status | text | NO | 'queued' |
| error_message | text | YES | — |
| created_at | timestamptz | NO | now() |
| updated_at | timestamptz | NO | now() |

Indexes: PK; index on `client_id`. **No FK constraint** on `document_id → knowledge_documents` despite the obvious relationship — confirmed absent from `pg_constraint`. **Performance advisor:** flagged as an unindexed FK-shaped column (`knowledge_ingestion_jobs_document_id_fkey` listed in the advisor output, though `pg_constraint` shows no such named constraint exists — likely the advisor is flagging the *implicit* relationship pattern rather than a declared, unindexed FK; needs a closer look, marked **Unresolved**).

---

### `knowledge_chat_sessions`
**Purpose:** one row per chat conversation (portal or Slack). **Rows:** 28. **Migration source:** `003_chat_history.sql`; `member_id` added in `004`; `origin`/`origin_metadata`/`idempotency_key` added in `005`.

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | uuid | NO | gen_random_uuid() |
| client_id | uuid | NO | — |
| title | text | YES | — |
| created_at | timestamptz | NO | now() |
| updated_at | timestamptz | NO | now() |
| deleted_at | timestamptz | YES | — |
| member_id | uuid | YES | — |
| origin | text | YES | — |
| origin_metadata | jsonb | YES | — |
| idempotency_key | text | YES | — |

Indexes: PK; index `(client_id, created_at DESC)`; index `(client_id, member_id, deleted_at, updated_at DESC)`; unique partial `(idempotency_key) WHERE idempotency_key IS NOT NULL` (dedup for retried Slack events). **Performance advisor:** `idx_chat_sessions_client_member` flagged unused. No FK on `member_id` (cross-project reference to Global's `client_members` — plain UUID by design, per the same pattern documented in migration 006 for `slack_collection_access`).

---

### `knowledge_chat_messages`
**Purpose:** individual messages within a chat session. **Rows:** 116. **Migration source:** `003_chat_history.sql`; `member_id` added in `004`.

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | uuid | NO | gen_random_uuid() |
| client_id | uuid | NO | — |
| session_id | uuid | NO | — |
| role | text | NO | — (CHECK: `user`/`assistant`/`system`) |
| content | text | NO | — |
| sources | jsonb | YES | — |
| metadata | jsonb | YES | — |
| created_at | timestamptz | NO | now() |
| deleted_at | timestamptz | YES | — |
| member_id | uuid | YES | — |

Indexes: PK; index `(client_id, member_id, session_id, created_at)`, `(client_id, session_id, created_at)`. FK: `session_id → knowledge_chat_sessions` (CASCADE, **unindexed** per advisor — though a covering composite index exists, it doesn't lead with `session_id` alone, which is likely why the linter still flags it).

---

### `knowledge_gaps`
**Purpose:** questions the retrieval pipeline couldn't answer — feeds the knowledge-gap-detection product feature. **Rows:** 0. **Migration source:** `003_chat_history.sql`; `member_id` added in `004`; `origin` fields added in `005`.

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | uuid | NO | gen_random_uuid() |
| client_id | uuid | NO | — |
| session_id | uuid | YES | — |
| message_id | uuid | YES | — |
| question | text | NO | — |
| reason | text | YES | — |
| status | text | NO | 'open' (CHECK: `open`/`reviewed`/`resolved`/`ignored`) |
| created_at | timestamptz | NO | now() |
| updated_at | timestamptz | NO | now() |
| member_id | uuid | YES | — |
| origin | text | YES | — |
| origin_metadata | jsonb | YES | — |
| idempotency_key | text | YES | — |

Indexes: PK; index `(client_id, member_id, created_at DESC)`, `(client_id, status, created_at DESC)`; unique partial `(idempotency_key) WHERE idempotency_key IS NOT NULL`. FKs: `session_id → knowledge_chat_sessions` (SET NULL, **unindexed**), `message_id → knowledge_chat_messages` (SET NULL, **unindexed**).

---

### `knowledge_collections`
**Purpose:** org-scoped grouping of documents (Milestone 5 — Slack Knowledge Collections). **Rows:** 6. **Migration source:** `006_knowledge_collections.sql`.

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | uuid | NO | gen_random_uuid() |
| client_id | uuid | NO | — |
| name | text | NO | — |
| is_default | boolean | NO | false |
| created_at | timestamptz | NO | now() |
| updated_at | timestamptz | NO | now() |

Indexes: PK; unique `(client_id, name)`; unique partial `(client_id) WHERE is_default = true` (guarantees exactly one default "General" collection per client); index on `client_id`. Referenced by `knowledge_documents.collection_id` (RESTRICT — a collection can't be deleted while documents still reference it).

---

### `documents` — legacy table

Full audit: [../../audits/SCHEMA_DRIFT.md](../../audits/SCHEMA_DRIFT.md). Summary: `id bigint, content text, metadata jsonb, embedding vector` (standard pgvector-quickstart shape), pre-dates the `001` migration, 0 rows today but 954 historical inserts / 146 deletes recorded — genuinely used before this schema replaced it. Only surviving code reference is a defensive legacy-cleanup delete path. Classified **Legacy but still required** (borderline safe-removal, pending one more confirmed-empty offboarding cycle).

## Related documents

[DATABASE.md](DATABASE.md) · [RLS.md](RLS.md) · [INDEXES.md](INDEXES.md) · [TRIGGERS.md](TRIGGERS.md) · [FUNCTIONS.md](FUNCTIONS.md) · [VECTOR_SEARCH.md](VECTOR_SEARCH.md) · [../../audits/SCHEMA_DRIFT.md](../../audits/SCHEMA_DRIFT.md)
