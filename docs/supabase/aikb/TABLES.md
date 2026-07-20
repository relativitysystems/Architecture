# AIKB Supabase — Table Reference

All data **Database Verified**. Row counts are point-in-time (2026-07-20). All 9 tables (8 live + 1 dropped legacy, see below) have RLS **enabled with zero policies** (see [RLS.md](RLS.md)); all grant full `arwdDxtm` to `anon`/`authenticated`/`service_role` (Supabase default).

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
**Purpose:** one row per chat conversation. **Rows:** 17 (1 `portal`, 16 `origin IS NULL` pre-Milestone-4 legacy). **Migration source:** `003_chat_history.sql`; `member_id` added in `004`; `origin`/`origin_metadata`/`idempotency_key` added in `005`. **Backlog M13 (revised):** Slack-originated conversations (`origin: 'slack'`/`'slack_dm'`) are no longer persisted here at all — `services/runKnowledgeQuery.js`'s `persistConversation: false` mode (used for all Slack traffic) skips session/message creation entirely, rather than creating a row and filtering it out of portal reads as the original M13 implementation did. 14 pre-existing Slack-origin rows (12 `slack` + 2 `slack_dm`) were hard-deleted as part of this revision; the `origin` column and its values remain valid for any future non-Slack caller.

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
**Purpose:** individual messages within a chat session. **Rows:** 94. **Migration source:** `003_chat_history.sql`; `member_id` added in `004`. **Backlog M13 (revised):** 28 pre-existing Slack-origin messages (tied to the 14 purged `knowledge_chat_sessions` rows above) were hard-deleted; no new Slack message rows are created going forward (see `knowledge_chat_sessions` above).

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

### `knowledge_slack_request_log`
**Purpose:** minimal operational/dedup metadata for the Slack ask flow (Backlog M13, revised) — replaces the old "look up the chat session by idempotency_key" mechanism now that Slack conversations are never persisted (see `knowledge_chat_sessions` above). **Rows:** 0 (new table). **Migration source:** `009_slack_request_log.sql`.

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | uuid | NO | gen_random_uuid() |
| client_id | uuid | NO | — |
| idempotency_key | text | NO | — |
| origin | text | NO | — (CHECK: `slack`/`slack_dm`) |
| status | text | NO | 'processing' (CHECK: `processing`/`delivered`/`failed`) |
| attempt_count | integer | NO | 1 |
| error_category | text | YES | — |
| created_at | timestamptz | NO | now() |
| updated_at | timestamptz | NO | now() |

Indexes: PK; unique `(idempotency_key)` (claim-before-enqueue dedup — `routes/knowledge.js` `POST /ask` claims a row before sending the `knowledge/slack.question.requested` Inngest event, so a retried `/ask` call never triggers a second LLM call/Slack reply); index `(client_id, created_at DESC)`. No FK on `client_id` (cross-project reference to Global's `clients`, plain UUID by design, same pattern as `knowledge_chat_sessions.member_id`). **Never stores** a question, answer, citation, or chunk/document text — only the columns listed above.

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

### `documents` — legacy table, **dropped**

Was `id bigint, content text, metadata jsonb, embedding vector` (standard pgvector-quickstart shape), pre-dating the `001` migration; held 954 historical inserts / 146 deletes before being superseded by this schema. Dropped in production via `007_drop_legacy_documents.sql` (backlog L6) alongside `match_documents()`, its only remaining caller. Full history: [../../audits/SCHEMA_DRIFT.md](../../audits/SCHEMA_DRIFT.md).

## Related documents

[DATABASE.md](DATABASE.md) · [RLS.md](RLS.md) · [INDEXES.md](INDEXES.md) · [TRIGGERS.md](TRIGGERS.md) · [FUNCTIONS.md](FUNCTIONS.md) · [VECTOR_SEARCH.md](VECTOR_SEARCH.md) · [../../audits/SCHEMA_DRIFT.md](../../audits/SCHEMA_DRIFT.md)
