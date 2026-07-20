# Schema Drift & Legacy Object Audit

Covers Phase 9 (migration drift) and Phase 10 (legacy audit) together, since every object found here is both undocumented-in-migrations *and* a legacy-cleanup candidate. All findings **Database Verified** (live schema/stats/advisor inspection) + **Historical** (git log across all branches, both repos) unless marked. Read-only audit — nothing below was changed live.

## Method

For each candidate: full schema, row count, indexes, FKs, RLS, grants, triggers, dependent objects (`pg_depend`/`prosrc` search), `pg_stat_user_tables` write history, `git log --all -S'<name>'` across every branch in both repos, and cross-check against tracked `.sql` migration files.

---

## Global — `public.folder_states`

- **Schema:** `id, client_id (FK→clients CASCADE), day_folder, address_folder, last_count, stable_count, last_seen_at, last_notified_count, last_notified_at, is_deleted, created_at, updated_at`.
- **Indexes:** PK; unique `(client_id, day_folder, address_folder)`.
- **RLS/grants/triggers:** RLS enabled, 0 policies; default full grants; `trg_folder_states_updated_at → set_updated_at()`.
- **Rows:** 0. **Lifetime writes (`pg_stat_user_tables`):** 0 inserts/updates/deletes ever recorded.
- **Migration provenance:** none — not created by any tracked `.sql` file.
- **Git history:** introduced in `b7a7859` ("Add authenticated client portal and Inngest Dropbox Slack workflow", 2026-06-07) backing `services/stateService.js`, a Dropbox folder-stability watcher. Fully removed the same day in `f7cd4e5` ("Remove Inngest workflow from website repo").
- **Current code reference:** one — a defensive entry in `deleteClientFull()`'s `defensiveTables` array (`services/supabaseService.js`), whose comment incorrectly assumes the table "may not exist live."
- **Classification: Safe removal candidate.** No dependents, no historical data, feature fully retired same-day it shipped.

## Global — `public.automation_logs`

- **Schema:** `id, client_id (FK→clients SET NULL), event_type, payload jsonb, created_at`.
- **Indexes:** PK only. **RLS/grants:** enabled/0 policies; default grants. **Triggers:** none.
- **Rows:** 0. **Lifetime writes:** 0.
- **Migration provenance:** none.
- **Git history:** sibling of `folder_states` — `logEvent()` added and removed the same day (`b7a7859` → `f7cd4e5`).
- **Current code reference:** same defensive cleanup array only.
- **Classification: Safe removal candidate.**

## Global — `public.set_updated_at()`

```sql
CREATE OR REPLACE FUNCTION public.set_updated_at() RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN NEW.updated_at = now(); RETURN NEW; END; $$;
```
- **Live triggers (verified, all enabled):** `trg_clients_updated_at` (`clients`), `trg_oauth_tokens_updated_at` (`oauth_tokens`), `trg_folder_states_updated_at` (`folder_states`).
- **Migration provenance:** none — `git log --all -S'set_updated_at'` returns zero commits on any branch; predates the tracked migration history entirely (earliest tracked migration is 2026-06-18; `clients`/`oauth_tokens` existed before that).
- **Classification: Active — required.** Not orphaned. Powers `updated_at` on two live, core tables (`clients`, `oauth_tokens`). If `folder_states` is dropped, its trigger drops with it automatically with no effect on the shared function.
- **Separate finding:** this is undocumented schema drift independent of the cleanup question — recommend a baseline/backfill migration capturing it, regardless of what happens to `folder_states`/`automation_logs`.

---

## AIKB — `public.documents`

- **Schema:** `id bigint (sequence), content text, metadata jsonb, embedding vector` — the standard Supabase/pgvector-quickstart shape, not this codebase's UUID convention.
- **Indexes:** PK only — **no vector index**, so any query against it was always a full sequential scan.
- **FKs:** none in, none out. **RLS/grants:** enabled/0 policies; default grants. **Triggers:** none.
- **Rows:** 0 today. **Lifetime writes:** **954 inserts, 146 deletes recorded**, `last_autoanalyze` 2026-06-03 — concrete evidence of real historical production use, not a never-used stub.
- **Migration provenance:** none — predates `001_knowledge_base_schema.sql`.
- **Git history:** touched in application code exactly once, in `c4c6fe7` ("delete stuff", 2026-07-01), which added `deleteLegacyDocumentsForClient()` — its own comment: *"no migration in this repo defines a bare `documents` table... but a legacy table may still exist live in the DB."*
- **Former code reference:** `deleteLegacyDocumentsForClient()` → `deleteAllClientData()`, a best-effort, error-swallowing cleanup step. **Removed** (backlog L6) — confirmed via direct query that the table held 0 rows and no write activity since the prior audit snapshot (`last_autoanalyze` unchanged), so the one-more-offboarding-cycle wait this doc originally recommended was treated as satisfied by that live check rather than a further calendar wait.
- **Classification: Resolved — dropped.** `services/supabaseService.js` no longer references `documents`; `migrations/007_drop_legacy_documents.sql` dropped the table and `match_documents()` together in one migration (per this doc's original recommendation — never one without the other), applied to production and verified gone (`information_schema.tables` returns 0 rows for `public.documents`).

## AIKB — `public.match_documents(query_embedding, match_count, filter)`

```sql
CREATE OR REPLACE FUNCTION public.match_documents(
  query_embedding extensions.vector, match_count integer DEFAULT NULL, filter jsonb DEFAULT '{}'
) RETURNS TABLE(id bigint, content text, metadata jsonb, similarity double precision)
LANGUAGE plpgsql AS $$ ... select ... from documents where metadata @> filter ... $$;
```
- **Depends on:** `documents` (only function referencing it).
- **Grants — verified directly via `has_function_privilege()`:** `anon = true`, `authenticated = true`, `service_role = true`.
- **Repo references:** zero, in either repository, on any branch (`git log --all -S'match_documents'` empty everywhere).
- **Live exposure:** callable via PostgREST (`/rest/v1/rpc/match_documents`) today regardless of code usage, since it's public-schema + EXECUTE-granted. Practical risk is low (its backing table is empty and RLS blocks anon/authenticated row access even if called), but it is a dead, exposed API surface.
- **Classification: Resolved — dropped (backlog L6).** `migrations/007_drop_legacy_documents.sql` dropped this function alongside `documents`, applied to production and verified gone (`pg_proc` returns 0 rows for `public.match_documents`). The one caveat this doc originally flagged — whether any consumer external to these two repos called it directly via PostgREST (e.g. a customer script holding an anon key) — was never confirmed or ruled out via static repo analysis, but is now moot: the function no longer exists, so any such caller would already be getting an error rather than silently continuing to work. If that caveat matters to anyone downstream, it surfaces now, not later.

## AIKB — `match_knowledge_chunks` — overload pair

| | 4-arg (legacy) | 5-arg (current) |
|---|---|---|
| Signature | `(vector, uuid, float8, int)` | `(vector, uuid, float8, int, uuid[] DEFAULT NULL)` |
| Language | SQL, STABLE | plpgsql, VOLATILE |
| Logic | queries `knowledge_chunks` only, **no collection filtering** | joins `knowledge_documents`, filters by `collection_id` |
| Introduced | `001_knowledge_base_schema.sql` | `006_knowledge_collections.sql` |
| Grants | anon/authenticated/service_role EXECUTE (verified) | same |
| Called by app code | **no** — every call site passes all 5 named params, including `match_collection_ids` (even `null`) | yes, exclusively (`services/supabaseService.js:searchChunks`) |

**Why both are still live:** migration `006` used `CREATE OR REPLACE FUNCTION` with a *different* argument list — in Postgres this creates a second, independent overload rather than replacing the first. The 4-arg version was never explicitly dropped.

**Replay/rollback implication (the one candidate where this matters):** the 4-arg overload **is** in a tracked migration (`001`). Replaying `001 → 006` from an empty database reproduces today's live state exactly (both overloads present) — so live state is *not* drifted relative to history here. But **dropping the 4-arg overload live without a companion migration would itself introduce drift** the other direction. Any removal must ship as a `DROP FUNCTION match_knowledge_chunks(vector, uuid, float8, int);` migration, not a console action.

**Security-relevant:** the 4-arg overload lacks collection scoping entirely — if it were ever hit, it would silently bypass the Slack collection-access-control feature ([ADR-005](../../decisions/ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md)). Verified today that app code cannot reach it (always sends the 5th param). Whether **PostgREST itself** could route an external caller's 4-parameter request there is a separate, unresolved question — see Verification Notes below.

**Classification:** 4-arg → **Safe removal candidate** (via migration, not console). 5-arg → **Active**.

---

## Verification notes (explicitly requested checks)

- **Is `set_updated_at()` attached to a live trigger?** Yes — three, all enabled. Not orphaned.
- **Is the 4-arg `match_knowledge_chunks` callable through PostgREST?** It's exposed with EXECUTE grants, so PostgREST's schema cache lists it. Calling it with just the 4 shared-name parameters is *also* a valid shape for the 5-arg overload (whose 5th param defaults), so PostgREST's overload resolution most likely rejects such a call with an ambiguous-candidate error (`PGRST203`/HTTP 300) rather than silently routing to either function. **Not verified against the live HTTP endpoint** in this audit (out of scope for a DB-only read-only pass) — recommend a direct check before relying on this conclusion.
- **Could overloaded signatures cause ambiguous API resolution?** Yes, per the above — the core risk of leaving both live.
- **Does `match_documents` have execute grants to anon/authenticated/service_role?** Yes, confirmed to all three directly.
- **Do the "unused" tables show recent writes via stats?** `folder_states`/`automation_logs`: no, zero ever. `documents`: yes historically (954/146), though no evidence of *current*-window writes — only recent reads via the legacy-cleanup path.
- **Is `documents` referenced by any FK or legacy ingestion flow?** No FK either direction. No ingestion flow writes to it anymore — sole reference is the defensive cleanup delete.
- **Would deleting any candidate break migration replay?** `folder_states`, `automation_logs`, `set_updated_at()`, `match_documents` are in **no** tracked migration — dropping them live has no replay implication (replay never creates them regardless). The 4-arg `match_knowledge_chunks` overload **is** tracked (migration `001`) — see above.

## Summary table

| Object | Classification | Blast radius if removed carelessly |
|---|---|---|
| `folder_states` | Safe removal candidate | Low |
| `automation_logs` | Safe removal candidate | Low |
| `set_updated_at()` | **Active — required** | **High** if mistakenly dropped (breaks `clients`/`oauth_tokens` timestamps) |
| `documents` (AIKB) | Legacy but still required | Medium — confirm empty-state stability first |
| `match_documents(...)` | Safe removal candidate | Low; unresolved external-caller question |
| `match_knowledge_chunks` (4-arg) | Safe removal candidate | Medium — remove via migration, not console |
| `match_knowledge_chunks` (5-arg) | Active | n/a |

## Related documents

[../supabase/global/TABLES.md](../supabase/global/TABLES.md) · [../supabase/global/FUNCTIONS.md](../supabase/global/FUNCTIONS.md) · [../supabase/global/TRIGGERS.md](../supabase/global/TRIGGERS.md) · [../supabase/aikb/TABLES.md](../supabase/aikb/TABLES.md) · [../supabase/aikb/FUNCTIONS.md](../supabase/aikb/FUNCTIONS.md) · [../repositories/RELATIVITY_REPO.md](../repositories/RELATIVITY_REPO.md) · [../repositories/AIKB_REPO.md](../repositories/AIKB_REPO.md)
