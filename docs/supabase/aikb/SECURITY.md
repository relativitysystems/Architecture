# AIKB Supabase — Database-Level Security

**Database Verified**, via Supabase's security advisor + direct grant/RLS/function inspection. For application-layer security see [../../../architecture/SECURITY.md](../../../architecture/SECURITY.md).

## Advisor findings

| Finding | Level | Scope | Detail |
|---|---|---|---|
| `rls_enabled_no_policy` | INFO | All 7 public tables | See [RLS.md](RLS.md) |
| `function_search_path_mutable` | WARN | `match_knowledge_chunks` (both overloads, flagged separately) | Neither overload pins `search_path`. Low practical risk since neither is `SECURITY DEFINER`. (`match_documents` was dropped — backlog L6 — and no longer appears in this finding.) |
| — | — | — | No `auth_leaked_password_protection` finding surfaced for this project in this pass — AIKB has no independent Auth system of its own (it validates Global-project JWTs), so this Supabase Auth-specific check may not apply the same way; **Unresolved**, not independently confirmed. |

## Notable exposed surface

The legacy 4-arg `match_knowledge_chunks` overload has `EXECUTE` granted to `anon` and `authenticated` — confirmed directly via `has_function_privilege()` — and is reachable via PostgREST's `/rest/v1/rpc/*` endpoint by virtue of being in the exposed `public` schema, **regardless of whether application code calls it**. (`match_documents` had the same exposure and has been dropped — backlog L6.) Full risk analysis: [../../audits/SCHEMA_DRIFT.md](../../audits/SCHEMA_DRIFT.md).

## Grants

Same default-schema-wide pattern as the Global project: every table grants full `arwdDxtm` to `postgres`/`anon`/`authenticated`/`service_role`. Both Supabase clients in `services/supabaseService.js` (`aikbSupabase`, `globalSupabase`) use service-role keys.

## Related documents

[RLS.md](RLS.md) · [FUNCTIONS.md](FUNCTIONS.md) · [DATABASE.md](DATABASE.md) · [../global/SECURITY.md](../global/SECURITY.md) · [../../audits/SCHEMA_DRIFT.md](../../audits/SCHEMA_DRIFT.md)
