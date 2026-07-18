# AIKB Supabase — Indexes & Performance Advisors

**Database Verified.** Full per-table index list in [TABLES.md](TABLES.md); the vector index specifically in [VECTOR_SEARCH.md](VECTOR_SEARCH.md).

## Unindexed foreign keys (4)

| Table | FK column | Constraint |
|---|---|---|
| `knowledge_chat_messages` | `session_id` | `knowledge_chat_messages_session_id_fkey` |
| `knowledge_gaps` | `message_id` | `knowledge_gaps_message_id_fkey` |
| `knowledge_gaps` | `session_id` | `knowledge_gaps_session_id_fkey` |
| `knowledge_ingestion_jobs` | `document_id` | flagged by advisor; **no matching constraint found in `pg_constraint`** — see note in [TABLES.md](TABLES.md), marked **Unresolved** |

## Unused indexes (2)

`idx_chat_sessions_client_member` (`knowledge_chat_sessions`), `knowledge_docs_collection_idx` (`knowledge_documents`) — never scanned per Postgres statistics. Same caveat as the Global project: low overall query volume today, not necessarily genuinely unneeded — `knowledge_docs_collection_idx` in particular backs a feature (`collection_id` filtering) that's recent (migration 006) and likely to see more traffic as Slack Knowledge Collections usage grows.

## Related documents

[TABLES.md](TABLES.md) · [VECTOR_SEARCH.md](VECTOR_SEARCH.md) · [DATABASE.md](DATABASE.md) · [../global/INDEXES.md](../global/INDEXES.md)
