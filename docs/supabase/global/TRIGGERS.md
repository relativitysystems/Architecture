# Global Supabase — Triggers

**Database Verified.** Exactly 3 triggers exist in the `public` schema, all invoking the same function.

| Trigger | Table | Timing | Function |
|---|---|---|---|
| `trg_clients_updated_at` | `clients` | BEFORE UPDATE | `set_updated_at()` |
| `trg_oauth_tokens_updated_at` | `oauth_tokens` | BEFORE UPDATE | `set_updated_at()` |
| `trg_folder_states_updated_at` | `folder_states` | BEFORE UPDATE | `set_updated_at()` |

All three are enabled (`tgenabled = 'O'`). None are created by any tracked migration in the repository — see [../../audits/SCHEMA_DRIFT.md](../../audits/SCHEMA_DRIFT.md). Full function detail: [FUNCTIONS.md](FUNCTIONS.md).

No other public table has an `updated_at`-maintenance trigger — `client_members`, `oauth_connections`, `oauth_credentials`, `knowledge_collections`-adjacent tables, etc. all rely on the application setting `updated_at` explicitly in each write (confirmed pattern in `services/supabaseService.js` and `services/oauthConnectionsService.js`), not a database trigger. This is an inconsistency worth noting: two tables (`clients`, `oauth_tokens`) get `updated_at` "for free" at the database level while every other table depends on every write path remembering to set it.

## Related documents

[FUNCTIONS.md](FUNCTIONS.md) · [TABLES.md](TABLES.md) · [../../audits/SCHEMA_DRIFT.md](../../audits/SCHEMA_DRIFT.md)
