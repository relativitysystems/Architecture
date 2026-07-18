# Global Supabase — Functions & RPCs

All **Database Verified**. Both functions owned by `postgres`, neither `SECURITY DEFINER`, neither has `search_path` pinned (flagged by the Supabase linter as `function_search_path_mutable`, WARN level — low practical risk here since neither is a definer function, but still worth fixing for hygiene). Both have default `EXECUTE` grants to `anon`/`authenticated`/`service_role`.

## `set_updated_at()`

```sql
CREATE OR REPLACE FUNCTION public.set_updated_at() RETURNS trigger
LANGUAGE plpgsql AS $$
BEGIN NEW.updated_at = now(); RETURN NEW; END; $$;
```

- **Volatility:** VOLATILE. **Called by:** Postgres trigger mechanism only — no application code calls it directly (confirmed: zero matches across both repos' full git history for the string `set_updated_at`).
- **Live triggers (all enabled):** `trg_clients_updated_at` (`clients`), `trg_oauth_tokens_updated_at` (`oauth_tokens`), `trg_folder_states_updated_at` (`folder_states`). See [TRIGGERS.md](TRIGGERS.md).
- **Classification: Active — required.** Powers `updated_at` on the live, core `clients` and `oauth_tokens` tables. **Not orphaned**, despite having zero provenance in tracked migrations (schema drift — see [../../audits/SCHEMA_DRIFT.md](../../audits/SCHEMA_DRIFT.md)).

## `replace_active_oauth_connection(...)`

Full signature (11 params): `p_client_id uuid, p_provider text, p_external_account_id text, p_external_account_name text, p_scopes_granted text[], p_provider_metadata jsonb, p_connected_by_member_id uuid, p_access_token_encrypted jsonb, p_refresh_token_encrypted jsonb, p_expires_at timestamptz, p_encryption_key_version integer`.

- **Purpose (Code Verified):** atomically replaces a client's active OAuth connection for a provider — revokes the prior active row and inserts the new `oauth_connections`/`oauth_credentials` pair in one transaction, avoiding a window where a client has zero or two active connections.
- **Called by:** `Relativity/services/oauthConnectionsService.js` via `client.rpc('replace_active_oauth_connection', {...})`. Also referenced in `Relativity/test/oauthConnectionsService.test.js`, which parses the migration SQL directly to assert the function signature matches.
- **Migration source:** `20260714_oauth_connections.sql`.
- **Classification: Active — required.** Sole write path for the current (non-legacy) OAuth model.

## Related documents

[DATABASE.md](DATABASE.md) · [TRIGGERS.md](TRIGGERS.md) · [SECURITY.md](SECURITY.md) · [../../audits/SCHEMA_DRIFT.md](../../audits/SCHEMA_DRIFT.md)
