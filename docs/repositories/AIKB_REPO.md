# AIKB Repository Reference

**Path:** `C:\Users\tenzin\aikb` · **Remote inferred:** `relativitysystems/AIKB` · **Branches:** `main`, `feat/slack-ask-pipeline` · **Runtime:** Node/Express, single Railway process; Inngest runs in-process.

Verification key: **Code Verified** (directly read this audit), **Structure Verified** (confirmed via file listing only), **Historical** (from pre-existing `architecture/` docs), **Inference**.

---

## Entry Points — Structure Verified

`server.js` mounts `/health`, `/api/inngest`, `/api/knowledge` (`routes/knowledge.js`), `/api/slack` (retired, `410`-only per `test/slackEventsRetired.test.js`).

## Folder Layout — Structure Verified

```
aikb/
├── server.js
├── config/index.js
├── middleware/
│   ├── resolveContext.js        # requireMemberContext — validates Supabase JWT vs Global DB
│   └── serviceRequest.js        # requireServiceRequest — HMAC envelope gate
├── migrations/                  # 6 tracked .sql files, 001–006
├── routes/
│   ├── knowledge.js             # x-api-key management routes + JWT-gated query/chat/gaps + /ask + /chat/redact (ADR-007)
│   └── slack.js                 # retired handler, 410-only
├── inngest/
│   ├── client.js
│   └── functions.js              # background ingestion + Slack question pipeline
├── services/                    # see table below
│   └── aikbDatabaseProvider.js  # ADR-008 — sole constructor/resolver of the AIKB Supabase client
├── scripts/                     # testDocxParse.js, testPdfParse.js (manual test scripts)
└── test/                        # 12 test files (74/74 passing as of the ADR-008 implementation)
```

## Services — verification status per row

| Service | Purpose | Status |
|---|---|---|
| `aikbDatabaseProvider.js` | [ADR-008](../../decisions/ADR-008-CLIENT-AIKB-DATABASE-ROUTING.md) — the only module permitted to construct or select the AIKB Supabase client. `getAikbDatabase(clientId)` validates `clientId`, fails closed if missing/empty, and returns `{ supabase, storageBucket, mode: 'shared' }` from a single process-cached client. Every `clientId` resolves to the same shared project today; no dedicated per-client project exists. | **Code Verified** |
| `supabaseService.js` | Primary AIKB-DB data-access layer: `knowledge_documents`, `knowledge_chunks`, `knowledge_ingestion_jobs`, `knowledge_chat_sessions`/`messages`, `knowledge_gaps`, `knowledge_collections`; resolves its AIKB client per-call via `aikbDatabaseProvider.js#getAikbDatabase(clientId)` (ADR-008) rather than constructing one itself; also holds a second, separate, static `globalSupabase` client reading `clients`/`client_members` from the Global project (untouched by ADR-008); `searchChunks()` calls `match_knowledge_chunks` RPC; `deleteLegacyDocumentsForClient()` targets the legacy `documents` table; `redactChatSessionByIdempotencyKey()` implements [ADR-007](../../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md)'s AIKB-side redaction (nulls session title, replaces message content with a fixed marker, nulls sources/metadata; idempotent) | **Code Verified** — read extensively across this and the prior audit turn |
| `documentParser.js` | Parses uploaded files (PDF/DOCX/etc.) into text prior to chunking | **Structure Verified** |
| `chunkService.js` | Splits parsed text into chunks for embedding | **Structure Verified** |
| `openaiService.js` | Embeddings (`text-embedding-3-small` by default) + chat completion for answer generation | **Code Verified** (partial — confirms `documents`/`knowledge_documents` reference lines) |
| `runKnowledgeQuery.js` | Shared retrieval+answer pipeline used by both portal `/query` and Slack `/ask` paths; calls `match_knowledge_chunks` via `searchChunks` | **Code Verified** (partial) |
| `relativityDeliverClient.js` | Signed callback client — AIKB → Relativity `/api/integrations/slack/deliver` | **Structure Verified** |
| `serviceRequestAuth.js` | HMAC-signed envelope verification — identical design to Relativity's copy | **Historical** |
| `googleDriveService.js` | Present but Google Drive sync explicitly retired per README ("Google Drive sync has been intentionally removed to keep the MVP focused on portal-uploaded documents") — likely dead code | **Structure Verified**, flagged for [../audits/](../audits/) legacy review |

## Database — Database Verified

AIKB's own Supabase project. Full reference: [../supabase/aikb/DATABASE.md](../supabase/aikb/DATABASE.md), [TABLES.md](../supabase/aikb/TABLES.md), [VECTOR_SEARCH.md](../supabase/aikb/VECTOR_SEARCH.md).

7 `public` tables, all owned by this repo: `knowledge_documents`, `knowledge_chunks` (pgvector, `ivfflat` cosine index), `knowledge_ingestion_jobs`, `knowledge_chat_sessions`, `knowledge_chat_messages`, `knowledge_gaps`, `knowledge_collections`. The legacy `documents` table (see [../audits/SCHEMA_DRIFT.md](../audits/SCHEMA_DRIFT.md)) has been dropped — backlog L6, `007_drop_legacy_documents.sql`, applied and verified in production (`information_schema.tables` returns 0 rows for it).

Read-only cross-project access into Global's `clients`/`client_members` for JWT-based entitlement checks (`middleware/resolveContext.js`), confirmed via the separate, static `globalSupabase` client in `supabaseService.js` — untouched by the ADR-008 database-routing provider, which governs the AIKB-project client only.

## Migrations — Database & Code Verified

| File | Adds |
|---|---|
| `001_knowledge_base_schema.sql` | `knowledge_documents`, `knowledge_chunks` (+ ivfflat index), `knowledge_ingestion_jobs`, `match_knowledge_chunks` (4-arg) |
| `002_add_storage_path.sql` | `knowledge_documents.storage_path` |
| `003_chat_history.sql` | `knowledge_chat_sessions`, `knowledge_chat_messages`, `knowledge_gaps` |
| `004_member_id.sql` | Adds `member_id` to chat/gaps tables |
| `005_slack_origin_tracking.sql` | Adds `origin`/`origin_metadata`/`idempotency_key` to chat sessions and gaps |
| `006_knowledge_collections.sql` | `knowledge_collections`, `knowledge_documents.collection_id` (+ FK, confirmed live), `match_knowledge_chunks` (5-arg overload, adds `match_collection_ids`) |
| `007_drop_legacy_documents.sql` | Drops `public.documents` and `match_documents()` — backlog L6. Applied to production. |

**Resolved schema drift (L6):** the pre-existing `documents` table (bigint id, `content`/`metadata`/`embedding`, standard pgvector-quickstart shape) and `match_documents()` RPC existed live but were created by **no** tracked migration — a pre-001 artifact. `pg_stat_user_tables` showed 954 historical inserts / 146 deletes against it (real prior use) but 0 rows and no write activity as of the review that found it. `deleteLegacyDocumentsForClient()`, the application-side best-effort cleanup shim, was removed from `services/supabaseService.js`, and `007_drop_legacy_documents.sql` dropped both the table and the RPC — applied to production and verified gone via direct query. See [../audits/SCHEMA_DRIFT.md](../audits/SCHEMA_DRIFT.md).

**Confirmed overload drift:** migration 006 added the 5-arg `match_knowledge_chunks` via `CREATE OR REPLACE FUNCTION` with a different parameter list — this creates a second, independent overload in Postgres rather than replacing the 4-arg original from migration 001. Both are live today; replaying 001→006 reproduces this exactly (not itself drift, but a live ambiguity/security-scoping risk — see [../audits/SCHEMA_DRIFT.md](../audits/SCHEMA_DRIFT.md)).

## Git History Notes — Code Verified

- `git log --all -S"'documents'"` finds the bare `documents` table touched only once in application code history: the `c4c6fe7` "delete stuff" commit (2026-07-01) that added `deleteLegacyDocumentsForClient()`, whose own comment stated no migration in the repo defines the table. That function was removed as part of backlog L6.
- `git log --all -S'match_documents'` returns zero commits in either repo/branch — the function has never been referenced in application code.

## Related documents

[RELATIVITY_REPO.md](RELATIVITY_REPO.md) · [CODE_OWNERSHIP.md](CODE_OWNERSHIP.md) · [../supabase/aikb/](../supabase/aikb/) · [../../architecture/AIKB.md](../../architecture/AIKB.md) (pre-existing narrative doc, **Historical**) · [../../architecture/INGESTION_PIPELINE.md](../../architecture/INGESTION_PIPELINE.md)
