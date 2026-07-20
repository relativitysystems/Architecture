# ADR-006: OAuth credentials must be encrypted at rest

## Status
Accepted — Implemented for all three providers this repo writes (Slack, then Google Drive and Dropbox per backlog H2)

## Date
Undated (decision); implemented for Slack 2026-07-14 (Milestone 2); extended to Google Drive and Dropbox 2026-07-19 (backlog H2)

## Context

Relativity's original Slack OAuth flow stored the exchanged bot token in plaintext (`oauth_tokens.access_token`), via `supabaseService.upsertToken`. A `SLACK_TOKEN_ENCRYPTION_KEY` variable existed in `.env.example` but was never referenced by any code — encryption was declared as an intent but never implemented. Google Drive and Dropbox tokens were stored the same way: plaintext, in the same shared `oauth_tokens` table. Any reader with database access to that table could use the tokens directly.

## Decision

**Provider OAuth credentials must be encrypted at rest, with connection metadata and secret material stored in separate tables.** For Slack, this was implemented as part of rebuilding the OAuth flow entirely (see [ADR-003](ADR-003-SLACK-EVENTS-LIVE-IN-RELATIVITY.md)):

- `oauth_connections` (metadata: `client_id`, `provider`, `status`, `external_account_id`, `provider_metadata jsonb`, etc.) is split from `oauth_credentials` (`connection_id`, `access_token_ciphertext`, `iv`, `auth_tag`, `encryption_key_version`) — different tables, different access patterns (metadata read often, credentials read only at send-time).
- Encryption: AES-256-GCM, implemented in `services/integrationCredentialEncryption.js`. A fresh 12-byte IV is generated per encryption call; a 16-byte auth tag is verified on every decrypt; there is **no plaintext fallback** under any failure mode — a corrupted ciphertext or auth-tag mismatch throws.
- The encryption key (`INTEGRATION_CREDENTIAL_ENCRYPTION_KEY`, with a deprecated temporary fallback to the older `SLACK_TOKEN_ENCRYPTION_KEY` name) is validated as exactly 64 hex characters (32 bytes) and re-read from `process.env` on every call, not a cached config snapshot.
- `encryption_key_version` is stored per credential row from day one, as the seam for a future key-rotation mechanism — but rotation tooling itself was not built; the column is hard-coded to `1` everywhere today.
- The existing plaintext Slack rows in `oauth_tokens` were deleted outright as part of the same migration (`DELETE FROM oauth_tokens WHERE provider = 'slack'`) — not migrated forward, since nothing in the codebase ever read them (Relativity's old Slack OAuth flow captured a token but never consumed it).

**Backlog H2 (2026-07-19) extended this same model to Google Drive and Dropbox**, unchanged in shape from what Slack already established — no new table, no new encryption logic, since `oauth_connections`/`oauth_credentials`/`integrationCredentialEncryption.js` were already fully provider-generic (`SUPPORTED_PROVIDERS` already listed both). Concretely:

- `routes/auth.js`'s `/dropbox/callback` and `/google/callback` now call `oauthConnectionsService.createOrReplaceConnection(...)` instead of `supabaseService.upsertToken(...)`.
- One new function was needed, not present for Slack: `updateCredentialForConnection(connectionId, {...})`, an in-place `UPDATE` on `oauth_credentials` only. Slack's bot token never expires, so `createOrReplaceConnection` (which always revokes the old connection row and inserts a new one) was never exercised as a *refresh* path. Google Drive's token does expire and needs silent background refresh — using `createOrReplaceConnection` for that would have churned the connection's id/`connected_at` on every expiring-token API call, which is wrong for what should be a transparent refresh. `googleDriveService.js#getValidAccessToken` uses the new function; on Google's refresh grant (which often omits a new `refresh_token`), the previous one is explicitly passed through so it's never overwritten with `null`.
- Unlike the Slack migration, **this was not a live-credential migration with real breakage risk**: a direct query of the Global Supabase project confirmed `oauth_tokens` held zero rows for any provider, including `google_drive` and `dropbox`, before this change — there were no active connections to preserve or lose. `migrations/20260719_gdrive_dropbox_oauth_connections.sql` deletes any (confirmed-empty) legacy rows for these two providers, included for symmetry with the Slack migration rather than because data was known to exist.
- `getClientConnectionStatus` and `getAllClientsWithStatus` (`supabaseService.js`) — which already special-cased Slack to read `oauth_connections` while Dropbox/Google Drive fell through to the legacy `getToken` path — now read all three providers from `oauth_connections` uniformly.
- Dropbox's token is write-only in this codebase (no route or service ever reads it back — `dropboxService.js#listFiles()` was already confirmed orphaned and deleted, see backlog L2), so only its write path needed migrating.

## Alternatives Considered

- **Add encrypted columns directly to the existing `oauth_tokens` table**: rejected — that table holds Slack/Dropbox/Google Drive rows together with no column indicating whether a given row's `access_token` is encrypted, making a partial migration ambiguous and error-prone (an `is_encrypted` flag is easy to get wrong at read time).
- **A bespoke `slack_connections` table**: rejected — fast to build, but Slack-specific column names would need to be re-derived for the next provider (Teams), which is exactly the kind of one-off structure the platform is trying to avoid. The `oauth_connections`/`oauth_credentials` pair is deliberately provider-generic (a `provider` CHECK constraint, not a table per provider).
- **Migrate Google Drive and Dropbox onto the encrypted model in the same change as Slack**: deferred at the time — out of scope for the Slack rebuild. Done separately as backlog H2, once confirmed low-risk (zero live rows to migrate).

## Consequences

- Slack, Google Drive, and Dropbox tokens are never stored or logged in plaintext anywhere in the platform, and are decrypted only in memory, immediately before use.
- Key rotation is not automated — a compromised or rotated encryption key would require a manual re-encryption sweep using the existing `encryption_key_version` column, which is not yet built. See [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md) item M3.
- Any workspace/account connected under an old, discarded plaintext OAuth flow had no migration path forward — reconnecting through the new flow was required. For Slack this meant an actual reconnect requirement; for Google Drive/Dropbox it was moot, since no live connections existed at migration time.
- `microsoft` and `gmail` remain in `SUPPORTED_PROVIDERS`/the DB `CHECK` constraint with no adapter built yet — the credential-storage half of a future connector requires no new migration, only a new OAuth adapter following this same pattern.

## Implementation Evidence

- `services/integrationCredentialEncryption.js` — `encryptCredential`/`decryptCredential`, AES-256-GCM, no plaintext fallback. Unchanged by H2 — already fully provider-generic.
- Migration `supabase/migrations/20260714_oauth_connections.sql` — `oauth_connections`/`oauth_credentials` schema, the `replace_active_oauth_connection` RPC (atomic revoke-old→insert-new), and the destructive `DELETE FROM oauth_tokens WHERE provider = 'slack'` statement.
- Migration `supabase/migrations/20260719_gdrive_dropbox_oauth_connections.sql` (backlog H2) — defensive `DELETE FROM oauth_tokens WHERE provider IN ('google_drive', 'dropbox')` against confirmed-empty rows.
- Test suite: 25+ existing tests covering the full encryption contract, plus new tests (backlog H2) for `updateCredentialForConnection` — rejects a missing connectionId/empty accessToken before touching the database, encrypts before writing, never touches `oauth_connections`, and documents the null-refresh-token overwrite contract callers must guard against.
- `oauth_tokens` (legacy) confirmed empty for every provider via a direct query of the live Global Supabase project as of this writing (2026-07-19) — table not yet dropped, kept as a defensive landing spot only.

## Related Documents

- [../architecture/SECURITY.md](../architecture/SECURITY.md)
- [../architecture/CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md)
- [ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md](ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md)
- [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md)
- [../history/ARCHITECTURE_REVIEW_PHASES.md](../history/ARCHITECTURE_REVIEW_PHASES.md)
