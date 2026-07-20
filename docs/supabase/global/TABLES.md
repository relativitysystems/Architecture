# Global Supabase — Table Reference

All data **Database Verified** (live schema inspection) unless marked. Row counts are point-in-time snapshots from this audit (2026-07-18) and will drift. Every table has RLS **enabled with zero policies** — see [RLS.md](RLS.md) for what this means in practice. Every table is granted full `arwdDxtm` privileges to `anon`/`authenticated`/`service_role`/`postgres` (Supabase's default schema-wide grant, not a deliberate per-table choice).

---

### `clients`
**Purpose (Code Verified):** the tenant/organization root — every other table hangs off `client_id`. **Rows:** 5. **Migration source:** pre-dates tracked migrations (schema drift — see [../../audits/SCHEMA_DRIFT.md](../../audits/SCHEMA_DRIFT.md)).

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | uuid | NO | gen_random_uuid() |
| name | text | NO | — |
| email | text | YES | — |
| slack_channel_id | text | YES | — |
| dropbox_watch_path | text | YES | — |
| is_active | boolean | NO | true |
| created_at | timestamptz | NO | now() |
| updated_at | timestamptz | NO | now() |
| max_members | integer | NO | 10 |

Indexes: PK on `id`. Trigger: `trg_clients_updated_at` → `set_updated_at()`. Referenced by (ON DELETE): `client_members`, `client_users`, `client_member_sessions` (CASCADE), `client_portal_issues` (CASCADE), `document_import_log` (CASCADE), `folder_states` (CASCADE), `oauth_connections`, `oauth_states`, `slack_collection_access`, `slack_event_log`, `team_invites` (CASCADE), `automation_logs` (SET NULL), `oauth_tokens` (CASCADE).

---

### `client_members`
**Purpose:** current team-membership model (role/status per user per client), replacing `client_users`. **Rows:** 6. **Migration source:** `20260618_team_members.sql`, identity adjustments in `20260619_client_members_identity.sql`.

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | uuid | NO | gen_random_uuid() |
| client_id | uuid | NO | — |
| auth_user_id | uuid | YES | — |
| email | text | NO | — |
| full_name | text | YES | — |
| role | text | NO | 'member' |
| status | text | NO | 'invited' |
| invited_by | uuid | YES | — |
| invited_at | timestamptz | YES | — |
| accepted_at | timestamptz | YES | — |
| last_active_at | timestamptz | YES | — |
| created_at | timestamptz | NO | now() |
| updated_at | timestamptz | NO | now() |

Indexes: PK; unique `(client_id, auth_user_id)`; unique partial `(auth_user_id) WHERE auth_user_id IS NOT NULL`; unique partial `(client_id, lower(email)) WHERE status NOT IN ('disabled','revoked')`; plain index on `auth_user_id`, `client_id`. FKs: `client_id → clients` (CASCADE), `auth_user_id → auth.users` (SET NULL), `invited_by → client_members` (self-referential, SET NULL). **Performance advisor:** `invited_by` FK has no covering index.

---

### `client_users` (legacy)
**Purpose (Historical, per migration comment):** predecessor of `client_members`; explicitly **not dropped** during the team-members migration ("client_users is intentionally NOT dropped here"). **Rows:** 2 — still holds live data, not confirmed dead. **Migration source:** pre-dates tracked migrations.

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | uuid | NO | gen_random_uuid() |
| client_id | uuid | NO | — |
| auth_user_id | uuid | NO | — |
| email | text | NO | — |
| created_at | timestamptz | NO | now() |

Indexes: PK; unique `(auth_user_id)`; unique `(client_id, email)`. FK: `client_id → clients` (CASCADE). Code reference: `services/supabaseService.js` (`.from('client_users')`, backfill source in migration 20260618).

---

### `team_invites`
**Purpose:** pending/accepted invitations to join a client's team. **Rows:** 2. **Migration source:** `20260618_team_members.sql`.

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | uuid | NO | gen_random_uuid() |
| client_id | uuid | NO | — |
| email | text | NO | — |
| role | text | NO | 'member' |
| token | text | NO | — |
| expires_at | timestamptz | NO | — |
| accepted_at | timestamptz | YES | — |
| revoked_at | timestamptz | YES | — |
| invited_by | uuid | YES | — |
| created_at | timestamptz | NO | now() |

Indexes: PK; unique `token`; plain index on `client_id`, `token`. FKs: `client_id → clients` (CASCADE), `invited_by → client_members` (SET NULL, **unindexed**). **Security note (Historical, per existing SECURITY.md):** `token` is stored in plaintext, unlike the hashed `oauth_states.state_hash` pattern. **Backlog M2 (2026-07-19): a migration adding hash-only storage is prepared but not yet applied** — `Relativity/supabase/migrations/20260719_hash_team_invite_tokens.sql` adds `token_hash`, backfills it from the existing plaintext rows, then drops `token` entirely. The table above still reflects the live, pre-migration schema as of this writing.

---

### `client_member_sessions`
**Purpose:** local mapping between a `client_members` row and an AIKB chat session id. **Rows:** 6. **Migration source:** `20260618_team_members.sql`.

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | uuid | NO | gen_random_uuid() |
| client_id | uuid | NO | — |
| member_id | uuid | NO | — |
| aikb_session_id | text | NO | — |
| created_at | timestamptz | NO | now() |

Indexes: PK; unique `(client_id, aikb_session_id)`; index on `member_id`. FKs: `client_id → clients` (CASCADE), `member_id → client_members` (CASCADE).

---

### `client_portal_issues`
**Purpose:** support/issue tickets submitted from the client portal. **Rows:** 0. **Migration source:** not in tracked migrations for this table specifically (created alongside other portal work — exact migration not isolated in this pass).

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | uuid | NO | gen_random_uuid() |
| client_id | uuid | NO | — |
| submitted_by | uuid | YES | — |
| submitted_email | text | YES | — |
| subject | text | NO | — |
| issue_type | text | NO | 'other' |
| message | text | NO | — |
| status | text | NO | 'open' |
| priority | text | NO | 'normal' |
| admin_notes | text | YES | — |
| created_at | timestamptz | NO | now() |
| updated_at | timestamptz | NO | now() |

Indexes: PK only. FK: `client_id → clients` (CASCADE, **unindexed** — performance advisor flagged).

---

### `oauth_tokens` (legacy)
**Purpose (Historical):** plaintext OAuth token storage for Google Drive and Dropbox. As of backlog H2, both providers write through `oauth_connections`/`oauth_credentials` instead (the same model Slack already used) — `upsertToken`/`getToken` in `supabaseService.js` still exist but nothing calls them for any provider anymore. **Rows:** 0 (confirmed empty for every provider both before and after H2 — this was not a live-data migration). **Migration source:** pre-dates tracked migrations.

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | uuid | NO | gen_random_uuid() |
| client_id | uuid | NO | — |
| provider | text | NO | — |
| access_token | text | NO | — |
| refresh_token | text | YES | — |
| expires_at | timestamptz | YES | — |
| created_at | timestamptz | NO | now() |
| updated_at | timestamptz | NO | now() |

Indexes: PK; unique `(client_id, provider)`. Trigger: `trg_oauth_tokens_updated_at` → `set_updated_at()`. FK: `client_id → clients` (CASCADE). **Security note (Historical):** `access_token`/`refresh_token` stored in plaintext — see [SECURITY.md](SECURITY.md).

---

### `oauth_connections` / `oauth_credentials`
**Purpose:** current, encrypted OAuth model (Slack only, as of this audit). Connection metadata (`oauth_connections`) is separated from encrypted secret material (`oauth_credentials`). **Rows:** 2 / 1. **Migration source:** `20260714_oauth_connections.sql`.

`oauth_connections`: `id, client_id, provider, external_account_id, external_account_name, status ('active' default), scopes_granted (text[]), provider_metadata (jsonb), connected_by_member_id, connected_at, updated_at, revoked_at`. Indexes: PK; plain indexes on `client_id`, `provider`, `status`; unique partial `(provider, external_account_id) WHERE status='active'`; unique partial `(client_id, provider) WHERE status='active'`. FKs: `client_id → clients` (CASCADE), `connected_by_member_id → client_members` (SET NULL, **unindexed**). **Performance advisor:** `idx_oauth_connections_client_id` and `idx_oauth_connections_status` both flagged unused (never scanned).

`oauth_credentials`: `id, connection_id, access_token_encrypted (jsonb), refresh_token_encrypted (jsonb), expires_at, encryption_key_version, created_at, updated_at`. Indexes: PK; index on `connection_id`. FK: `connection_id → oauth_connections` (CASCADE).

Write path: `services/oauthConnectionsService.js` via the `replace_active_oauth_connection` RPC — see [FUNCTIONS.md](FUNCTIONS.md).

---

### `oauth_states`
**Purpose:** CSRF protection for the Slack OAuth connect flow — stores only a SHA-256 hash of the state value, 10-minute TTL, single-use. **Rows:** 3. **Migration source:** `20260715_oauth_states.sql`.

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | uuid | NO | gen_random_uuid() |
| state_hash | text | NO | — |
| client_id | uuid | NO | — |
| member_id | uuid | NO | — |
| provider | text | NO | — |
| redirect_after | text | YES | — |
| expires_at | timestamptz | NO | — |
| consumed_at | timestamptz | YES | — |
| created_at | timestamptz | NO | now() |

Indexes: PK; unique `state_hash`; index on `client_id`, `expires_at`. FKs: `client_id → clients` (CASCADE), `member_id → client_members` (CASCADE, **unindexed**). **Performance advisor:** both `idx_oauth_states_client_id` and `idx_oauth_states_expires_at` flagged unused.

---

### `slack_event_log`
**Purpose:** Slack Events dedup, audit trail, and delivery-retry state. **Rows:** 12 (at last count; grows during normal operation). **Migration source:** `20260716_slack_event_log.sql`. **Implemented, [ADR-007](../../../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md):** `status` is unconstrained `text`, so `delivery_failed` was added as a terminal value with no migration — `Relativity/services/slackEventLogService.js#markDeliveryFailed` sets it. On reaching it, `question` is redacted/nulled in the same UPDATE, while `external_event_id`, `client_id`, `status`, `attempt_count`, `error_code`, and `failed_at` are retained for dedup, auditing, and debugging, exactly as this ADR specifies. See [../../../roadmap/FEATURE_BACKLOG.md](../../../roadmap/FEATURE_BACKLOG.md) item H5, completed.

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | uuid | NO | gen_random_uuid() |
| provider | text | NO | 'slack' |
| external_event_id | text | NO | — |
| client_id | uuid | NO | — |
| connection_id | uuid | NO | — |
| event_type | text | NO | — |
| channel_id | text | NO | — |
| event_ts | text | NO | — |
| thread_ts | text | YES | — |
| question | text | YES | — |
| idempotency_key | text | NO | — |
| status | text | NO | 'received' |
| attempt_count | integer | NO | 0 |
| error_code | text | YES | — |
| response_metadata | jsonb | YES | — |
| received_at | timestamptz | NO | now() |
| processing_started_at | timestamptz | YES | — |
| completed_at | timestamptz | YES | — |
| failed_at | timestamptz | YES | — |

Indexes: PK; unique `(provider, external_event_id)` (dedup guarantee); index `(client_id, received_at DESC)`; partial index `(status, received_at) WHERE status IN ('received','enqueued')` (originally added for the sweep's `listStuckForRetry` query — that query no longer exists in either repository after ADR-007's implementation removed the sweep, so this index now has **no known application-code reader**; candidate for a future cleanup pass, not addressed by this documentation update). FKs: `client_id → clients` (CASCADE), `connection_id → oauth_connections` (CASCADE, **unindexed**). **Performance advisor:** `idx_slack_event_log_client_id` flagged unused.

---

### `slack_collection_access`
**Purpose:** per-client allow-list of AIKB `knowledge_collections` that Slack retrieval may search — enforced fail-closed (empty = search nothing). **Rows:** 1. **Migration source:** `20260717_slack_collection_access.sql`.

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | uuid | NO | gen_random_uuid() |
| client_id | uuid | NO | — |
| collection_id | uuid | NO | — (plain UUID, **no FK** — `knowledge_collections` lives in the separate AIKB project) |
| created_at | timestamptz | NO | now() |

Indexes: PK; unique `(client_id, collection_id)`; index on `client_id`. FK: `client_id → clients` (CASCADE). Code: `services/slackCollectionAccessService.js`.

---

### `document_import_log`
**Purpose:** provenance/display record of upload batches — display only, not the ingestion source of truth (that's AIKB's `knowledge_documents`). **Rows:** 6. **Migration source:** `20260705_document_import_log.sql`, widened in `20260712_document_import_log_widen.sql`.

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | uuid | NO | gen_random_uuid() |
| client_id | uuid | NO | — |
| import_batch_id | uuid | NO | — |
| source_type | text | NO | — |
| source_path | text | YES | — |
| file_name | text | NO | — |
| source_file_id | text | NO | — |
| imported_by | uuid | YES | — |
| created_at | timestamptz | NO | now() |

Indexes: PK; index `(client_id, import_batch_id)`, `(client_id, created_at DESC)`, `(client_id, source_file_id)`. FKs: `client_id → clients` (CASCADE), `imported_by → client_members` (SET NULL, **unindexed**). **Performance advisor:** `idx_document_import_log_batch` and `idx_document_import_log_client_source_file` both flagged unused.

---

### `leads`
**Purpose:** public marketing-site lead-capture form submissions. **Rows:** 0.

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | uuid | NO | gen_random_uuid() |
| name | text | NO | — |
| email | text | NO | — |
| phone | text | YES | — |
| company | text | YES | — |
| message | text | NO | — |
| notes | text | YES | — |
| source | text | YES | 'website' |
| status | text | NO | 'new' |
| archived | boolean | NO | false |
| created_at | timestamptz | NO | now() |

Indexes: PK; index on `created_at DESC`, `status`. No FKs. **Performance advisor:** `leads_status_idx` flagged unused.

---

### `folder_states`, `automation_logs` — dead tables

Full audit (schema, historical writes, classification): [../../audits/SCHEMA_DRIFT.md](../../audits/SCHEMA_DRIFT.md). Summary: both back a Dropbox folder-watcher feature added and removed the same day (2026-06-07); zero rows, zero lifetime writes recorded; classified **Safe removal candidate**.

## Related documents

[DATABASE.md](DATABASE.md) · [RLS.md](RLS.md) · [INDEXES.md](INDEXES.md) · [TRIGGERS.md](TRIGGERS.md) · [FUNCTIONS.md](FUNCTIONS.md) · [../../database/](../../database/) (cross-database data-model view) · [../../audits/SCHEMA_DRIFT.md](../../audits/SCHEMA_DRIFT.md)
