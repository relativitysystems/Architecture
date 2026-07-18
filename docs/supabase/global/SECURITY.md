# Global Supabase — Database-Level Security

**Database Verified**, via Supabase's security advisor + direct grant/RLS/function inspection. For application-layer security (auth mechanisms, encryption, API security, portal security) see the pre-existing [../../../architecture/SECURITY.md](../../../architecture/SECURITY.md) — this page covers only what the database itself reports, and corrects one claim in that document (see [RLS.md](RLS.md)).

## Advisor findings

| Finding | Level | Scope | Detail |
|---|---|---|---|
| `rls_enabled_no_policy` | INFO | All 16 public tables | RLS enabled, zero policies — see [RLS.md](RLS.md) for full analysis |
| `function_search_path_mutable` | WARN | `set_updated_at`, `replace_active_oauth_connection` | Neither function pins `search_path`; low practical risk since neither is `SECURITY DEFINER`, but flagged by Supabase's linter as a hygiene issue — a future refactor to `SECURITY DEFINER` without also adding `SET search_path = ''` would be a real privilege-escalation risk |
| `auth_leaked_password_protection` | WARN | Project-wide Auth config | HaveIBeenPwned checking is disabled for new passwords — not enforced today |

## Grants

Every `public` table grants full `arwdDxtm` to `postgres`, `anon`, `authenticated`, `service_role` — Supabase's default schema-wide grant, not a per-table decision. Since every application Supabase client uses the **service-role key** (confirmed: `createClient(supabaseConfig.url, supabaseConfig.serviceKey)` in `services/supabaseService.js`), RLS bypass is total and grants are the only theoretical backstop — but `service_role` also bypasses grant restrictions in practice since it has superuser-equivalent access to `public` schema data.

## Function security posture

Both application functions (`set_updated_at`, `replace_active_oauth_connection`) run with invoker's privileges (`SECURITY INVOKER`, the default — neither is `SECURITY DEFINER`). Both are owned by `postgres`. Since every caller is the service-role client, this distinction has limited practical effect today, but matters if either function is ever called by a lower-privilege role.

## Known gaps carried from existing SECURITY.md (Historical, not re-verified this pass)

The existing document lists: cross-tenant Google Drive access via shared API key, non-constant-time secret comparisons in several places, no CORS/rate-limiting, unsigned OAuth state for Google Drive/Dropbox, no encryption-key rotation, plaintext team-invite tokens, plaintext legacy `oauth_tokens`. These are **application-layer**, not database-layer, findings — cross-reference [../../../architecture/SECURITY.md](../../../architecture/SECURITY.md) directly rather than duplicating here.

## Related documents

[RLS.md](RLS.md) · [FUNCTIONS.md](FUNCTIONS.md) · [DATABASE.md](DATABASE.md) · [../aikb/SECURITY.md](../aikb/SECURITY.md) · [../../audits/MASTER_ARCHITECTURE_AUDIT.md](../../audits/MASTER_ARCHITECTURE_AUDIT.md) (queued)
