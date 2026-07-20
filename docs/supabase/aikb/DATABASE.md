# AIKB Supabase Project — Database Overview

MCP alias: `supabase-aikb`. Owning repo: [AIKB](../../repositories/AIKB_REPO.md). All statements **Database Verified** unless marked otherwise.

## Schemas

| Schema | Kind |
|---|---|
| `public` | Application data — see [TABLES.md](TABLES.md) |
| `auth`, `storage`, `realtime`, `vault`, `extensions`, `graphql`, `graphql_public`, `pgbouncer` | Supabase-managed infrastructure |

## Extensions enabled (non-default)

| Extension | Schema | Version |
|---|---|---|
| `vector` (pgvector) | extensions | 0.8.0 |
| `pgcrypto` | extensions | 1.3 |
| `uuid-ossp` | extensions | 1.1 |
| `supabase_vault` | vault | 0.3.1 |
| `pg_stat_statements` | extensions | 1.11 |
| `plpgsql` | pg_catalog | 1.0 (default) |

`vector` being installed here (and **not** in the Global project) is the clearest schema-level confirmation that AIKB, not Relativity, owns retrieval/embeddings.

## Views

Only system views (`extensions.pg_stat_statements*`, `vault.decrypted_secrets`). No application-defined views.

## Tables

7 tables in `public`. Full reference: [TABLES.md](TABLES.md).

## Functions / RPCs

2 distinct functions, one with two overloads: `match_knowledge_chunks` (×2 overloads). `match_documents` was dropped (backlog L6, `007_drop_legacy_documents.sql`). Full reference: [FUNCTIONS.md](FUNCTIONS.md), [VECTOR_SEARCH.md](VECTOR_SEARCH.md).

## Storage

One bucket: `aikb-documents` (private, created 2026-06-11, no explicit size/MIME limits set at the bucket level). Full detail: [STORAGE.md](STORAGE.md).

## Migration history

Supabase's tracked-migrations table is empty despite 7 `.sql` files in `aikb/migrations/` (001–007). Same drift pattern as the Global project. The previously-confirmed pre-migration artifact — the `documents` table and `match_documents()` function — has been resolved: `007_drop_legacy_documents.sql` dropped both (backlog L6), applied to this live database and verified gone — see [../../audits/SCHEMA_DRIFT.md](../../audits/SCHEMA_DRIFT.md).

## Security & Performance advisors

- 7 `rls_enabled_no_policy` findings — every public table
- 2 `function_search_path_mutable` findings — **both** `match_knowledge_chunks` overloads
- 4 `unindexed_foreign_keys`, 2 `unused_index` findings

Full detail: [SECURITY.md](SECURITY.md), [INDEXES.md](INDEXES.md).

## Related documents

[TABLES.md](TABLES.md) · [FUNCTIONS.md](FUNCTIONS.md) · [VECTOR_SEARCH.md](VECTOR_SEARCH.md) · [RLS.md](RLS.md) · [INDEXES.md](INDEXES.md) · [STORAGE.md](STORAGE.md) · [TRIGGERS.md](TRIGGERS.md) · [SECURITY.md](SECURITY.md) · [../global/DATABASE.md](../global/DATABASE.md)
