# Global Supabase Project — Database Overview

MCP alias: `supabase-global`. Owning repo: [Relativity](../../repositories/RELATIVITY_REPO.md). All statements below are **Database Verified** via direct MCP inspection unless marked otherwise.

## Schemas

| Schema | Kind |
|---|---|
| `public` | Application data — see [TABLES.md](TABLES.md) |
| `auth`, `storage`, `realtime`, `vault`, `extensions`, `graphql`, `graphql_public`, `pgbouncer` | Supabase-managed infrastructure, not application-owned |

## Extensions enabled (non-default)

| Extension | Schema | Version |
|---|---|---|
| `pgcrypto` | extensions | 1.3 |
| `uuid-ossp` | extensions | 1.1 |
| `supabase_vault` | vault | 0.3.1 |
| `pg_stat_statements` | extensions | 1.11 |
| `plpgsql` | pg_catalog | 1.0 (default) |

`vector` (pgvector) is **not** installed on this project — confirms Global performs no embedding/vector work, consistent with AIKB owning retrieval. Dozens of other extensions are available (`postgis`, `pg_cron`, `pg_net`, etc.) but not installed.

## Views

Only system views exist: `extensions.pg_stat_statements`, `extensions.pg_stat_statements_info`, `vault.decrypted_secrets`. **No application-defined views.**

## Tables

16 tables in `public`, all Relativity-owned. Full reference: [TABLES.md](TABLES.md).

## Functions / RPCs

2 application functions: `replace_active_oauth_connection`, `set_updated_at()`. Full reference: [FUNCTIONS.md](FUNCTIONS.md).

## Storage

**No storage buckets exist in this project** (`storage.buckets` is empty). All file storage for the platform lives in the AIKB project's `aikb-documents` bucket — see [../aikb/STORAGE.md](../aikb/STORAGE.md). Relativity uploads directly to that bucket using AIKB's storage credentials, not its own.

## Migration history

Supabase's tracked-migrations table is **empty** (`list_migrations` returns `[]`) despite 8 `.sql` files existing in `Relativity/supabase/migrations/`. This is consistent with migrations being applied manually (SQL editor / CLI without recording) rather than through a tracked migration pipeline. See [../../audits/SCHEMA_DRIFT.md](../../audits/SCHEMA_DRIFT.md) for the specific undocumented objects this produced.

## Security & Performance advisors

Supabase's built-in linter was queried directly. Full detail: [SECURITY.md](SECURITY.md), [INDEXES.md](INDEXES.md).

- 16 `rls_enabled_no_policy` findings (every public table)
- 2 `function_search_path_mutable` findings (`set_updated_at`, `replace_active_oauth_connection`)
- 1 `auth_leaked_password_protection` finding (disabled)
- 7 `unindexed_foreign_keys`, 6 `unused_index` findings

## Related documents

[TABLES.md](TABLES.md) · [FUNCTIONS.md](FUNCTIONS.md) · [RLS.md](RLS.md) · [INDEXES.md](INDEXES.md) · [STORAGE.md](STORAGE.md) · [TRIGGERS.md](TRIGGERS.md) · [SECURITY.md](SECURITY.md) · [../../repositories/RELATIVITY_REPO.md](../../repositories/RELATIVITY_REPO.md) · [../aikb/DATABASE.md](../aikb/DATABASE.md)
