# AIKB Supabase — Database-Level Security

**Database Verified**, via Supabase's security advisor + direct grant/RLS/function inspection. For application-layer security see [../../../architecture/SECURITY.md](../../../architecture/SECURITY.md).

## Advisor findings

| Finding | Level | Scope | Detail |
|---|---|---|---|
| `rls_enabled_no_policy` | INFO | All 8 public tables | See [RLS.md](RLS.md) |
| `function_search_path_mutable` | WARN | `match_documents`, `match_knowledge_chunks` (both overloads, flagged separately) | None of the 3 functions pin `search_path`. Low practical risk since none is `SECURITY DEFINER`. |
| — | — | — | No `auth_leaked_password_protection` finding surfaced for this project in this pass — AIKB has no independent Auth system of its own (it validates Global-project JWTs), so this Supabase Auth-specific check may not apply the same way; **Unresolved**, not independently confirmed. |

## Notable exposed surface

`match_documents` and the legacy 4-arg `match_knowledge_chunks` overload both have `EXECUTE` granted to `anon` and `authenticated` — confirmed directly via `has_function_privilege()` — and are both reachable via PostgREST's `/rest/v1/rpc/*` endpoint by virtue of being in the exposed `public` schema, **regardless of whether application code calls them**. Full risk analysis: [../../audits/SCHEMA_DRIFT.md](../../audits/SCHEMA_DRIFT.md).

## Grants

Same default-schema-wide pattern as the Global project: every table grants full `arwdDxtm` to `postgres`/`anon`/`authenticated`/`service_role`. Both Supabase clients in `services/supabaseService.js` (`aikbSupabase`, `globalSupabase`) use service-role keys.

## Related documents

[RLS.md](RLS.md) · [FUNCTIONS.md](FUNCTIONS.md) · [DATABASE.md](DATABASE.md) · [../global/SECURITY.md](../global/SECURITY.md) · [../../audits/SCHEMA_DRIFT.md](../../audits/SCHEMA_DRIFT.md)
