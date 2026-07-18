# Global Supabase — Indexes & Performance Advisors

**Database Verified.** Full index list already itemized per-table in [TABLES.md](TABLES.md). This page consolidates the Supabase performance-advisor findings.

## Unindexed foreign keys (7)

| Table | FK column | Constraint |
|---|---|---|
| `automation_logs` | `client_id` | `automation_logs_client_id_fkey` |
| `client_members` | `invited_by` | `client_members_invited_by_fkey` |
| `client_portal_issues` | `client_id` | `client_portal_issues_client_id_fkey` |
| `document_import_log` | `imported_by` | `document_import_log_imported_by_fkey` |
| `oauth_connections` | `connected_by_member_id` | `oauth_connections_connected_by_member_id_fkey` |
| `oauth_states` | `member_id` | `oauth_states_member_id_fkey` |
| `slack_event_log` | `connection_id` | `slack_event_log_connection_id_fkey` |
| `team_invites` | `invited_by` | `team_invites_invited_by_fkey` |

**Impact:** `ON DELETE`/`ON UPDATE` cascade checks and any join/filter on these columns do a full table scan rather than an index lookup. All affected tables are small today (single-digit to low-dozens of rows), so this is low urgency, but worth addressing before `client_members`/`oauth_connections` grow.

## Unused indexes (6)

`leads_status_idx`, `idx_oauth_connections_client_id`, `idx_oauth_connections_status`, `idx_oauth_states_expires_at`, `idx_oauth_states_client_id`, `idx_document_import_log_batch`, `idx_document_import_log_client_source_file` — flagged as never-scanned by the Postgres statistics collector. **Caveat:** this project is low-volume (single-digit to low-dozens of rows per table), so "unused" likely reflects low query volume overall rather than a genuinely unnecessary index — do not drop without confirming against a longer production window. See [../../audits/PERFORMANCE.md](../../audits/PERFORMANCE.md) (queued, not yet written) for a fuller pass.

## Other

- `auth_db_connections_absolute` (INFO): Auth server connection pool is configured as an absolute count (10) rather than percentage-based — relevant only if the Postgres instance is resized.

## Related documents

[TABLES.md](TABLES.md) · [DATABASE.md](DATABASE.md) · [../aikb/INDEXES.md](../aikb/INDEXES.md)
