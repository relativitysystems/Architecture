# Global Supabase — Row Level Security

**Database Verified.** This corrects a claim in the pre-existing `architecture/SECURITY.md`: that document states "There is no Row-Level Security anywhere in either repository," based on a grep for `CREATE POLICY`/`ENABLE ROW LEVEL SECURITY` across tracked `.sql` migration files. Direct live-database inspection (via `pg_policies` and Supabase's own security advisor) shows the more precise picture:

**RLS is enabled on every one of the 16 `public` tables in this project — but zero policies exist on any of them.**

| Table | RLS enabled | Policy count |
|---|---|---|
| `automation_logs`, `client_member_sessions`, `client_members`, `client_portal_issues`, `client_users`, `clients`, `document_import_log`, `folder_states`, `leads`, `oauth_connections`, `oauth_credentials`, `oauth_states`, `oauth_tokens`, `slack_collection_access`, `slack_event_log`, `team_invites` | true (all 16) | **0** (all 16) |

Confirmed via Supabase's built-in linter (`rls_enabled_no_policy`, INFO level) independently for all 16 tables — not just a sample.

## Why this doesn't change the practical security posture (Historical, per existing SECURITY.md — logic re-verified against grants)

RLS with zero policies means: **deny all rows to any role subject to RLS** (`anon`, `authenticated`), because Postgres RLS defaults to deny when enabled and no permissive policy matches. However:

- Every table's grants show `postgres`, `anon`, `authenticated`, and `service_role` all holding full `arwdDxtm` privileges (Supabase's default schema-wide grant) — grants are not the blocking layer here, RLS is.
- Every Supabase client constructed in either repo uses the **service-role key**, which has `BYPASSRLS` and ignores RLS entirely regardless of policy count.

**Net effect:** RLS being "enabled" here provides zero actual protection today, because the only role that ever queries these tables (service_role) bypasses it. The corrected, precise statement is: *tenant isolation has no database-level backstop — RLS is enabled but vacuous, not simply absent.* This distinction matters if a future `anon`/`authenticated`-scoped Supabase client is ever introduced (e.g., direct-from-browser queries) — it would silently see zero rows rather than failing loudly, which could look like a data-loss bug rather than a missing-policy bug.

## Recommendation

Tracked as a known gap in the existing `architecture/SECURITY.md` Future Improvements list ("Introduce Row-Level Security as defense-in-depth... without removing existing application-layer checks"). This audit adds: any future policy work should also address the enabled-but-empty state directly, since a naive read of "RLS is enabled" (e.g., from a compliance checklist) would currently be misleading without the "zero policies" caveat.

## Related documents

[DATABASE.md](DATABASE.md) · [TABLES.md](TABLES.md) · [SECURITY.md](SECURITY.md) · [../aikb/RLS.md](../aikb/RLS.md) (identical pattern in the AIKB project) · [../../../architecture/SECURITY.md](../../../architecture/SECURITY.md) (pre-existing narrative security doc)
