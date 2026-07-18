# ADR-006: OAuth credentials must be encrypted at rest

## Status
Accepted — Implemented for Slack; legacy providers not yet migrated

## Date
Undated (decision); implemented 2026-07-14 (Milestone 2)

## Context

Relativity's original Slack OAuth flow stored the exchanged bot token in plaintext (`oauth_tokens.access_token`), via `supabaseService.upsertToken`. A `SLACK_TOKEN_ENCRYPTION_KEY` variable existed in `.env.example` but was never referenced by any code — encryption was declared as an intent but never implemented. Google Drive and Dropbox tokens were, and remain, stored the same way: plaintext, in the same shared `oauth_tokens` table. Any reader with database access to that table could use the tokens directly.

## Decision

**Provider OAuth credentials must be encrypted at rest, with connection metadata and secret material stored in separate tables.** For Slack, this was implemented as part of rebuilding the OAuth flow entirely (see [ADR-003](ADR-003-SLACK-EVENTS-LIVE-IN-RELATIVITY.md)):

- `oauth_connections` (metadata: `client_id`, `provider`, `status`, `external_account_id`, `provider_metadata jsonb`, etc.) is split from `oauth_credentials` (`connection_id`, `access_token_ciphertext`, `iv`, `auth_tag`, `encryption_key_version`) — different tables, different access patterns (metadata read often, credentials read only at send-time).
- Encryption: AES-256-GCM, implemented in `services/integrationCredentialEncryption.js`. A fresh 12-byte IV is generated per encryption call; a 16-byte auth tag is verified on every decrypt; there is **no plaintext fallback** under any failure mode — a corrupted ciphertext or auth-tag mismatch throws.
- The encryption key (`INTEGRATION_CREDENTIAL_ENCRYPTION_KEY`, with a deprecated temporary fallback to the older `SLACK_TOKEN_ENCRYPTION_KEY` name) is validated as exactly 64 hex characters (32 bytes) and re-read from `process.env` on every call, not a cached config snapshot.
- `encryption_key_version` is stored per credential row from day one, as the seam for a future key-rotation mechanism — but rotation tooling itself was not built; the column is hard-coded to `1` everywhere today.
- The existing plaintext Slack rows in `oauth_tokens` were deleted outright as part of the same migration (`DELETE FROM oauth_tokens WHERE provider = 'slack'`) — not migrated forward, since nothing in the codebase ever read them (Relativity's old Slack OAuth flow captured a token but never consumed it).

## Alternatives Considered

- **Add encrypted columns directly to the existing `oauth_tokens` table**: rejected — that table holds Slack/Dropbox/Google Drive rows together with no column indicating whether a given row's `access_token` is encrypted, making a partial migration ambiguous and error-prone (an `is_encrypted` flag is easy to get wrong at read time).
- **A bespoke `slack_connections` table**: rejected — fast to build, but Slack-specific column names would need to be re-derived for the next provider (Teams), which is exactly the kind of one-off structure the platform is trying to avoid. The `oauth_connections`/`oauth_credentials` pair is deliberately provider-generic (a `provider` CHECK constraint, not a table per provider).
- **Migrate Google Drive and Dropbox onto the encrypted model in the same change**: deferred — out of scope for the Slack rebuild; tracked as separate future work since it has no dependency on Slack shipping.

## Consequences

- Slack bot tokens are never stored or logged in plaintext anywhere in the platform, and are decrypted only in memory, immediately before use, inside Relativity's delivery service — AIKB never sees a Slack credential.
- Google Drive and Dropbox tokens remain plaintext in the legacy `oauth_tokens` table today — this decision's scope covers Slack only; migrating the other two providers is tracked, unresolved work. See [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md) item H2.
- Key rotation is not automated — a compromised or rotated encryption key would require a manual re-encryption sweep using the existing `encryption_key_version` column, which is not yet built. See [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md) item M3.
- Any workspace connected under the old, discarded plaintext OAuth flow had no migration path forward — reconnecting through the new flow was required, and there is no way to recover a deleted plaintext token except from a pre-migration database backup.

## Implementation Evidence

- `services/integrationCredentialEncryption.js` — `encryptCredential`/`decryptCredential`, AES-256-GCM, no plaintext fallback.
- Migration `supabase/migrations/20260714_oauth_connections.sql` — `oauth_connections`/`oauth_credentials` schema, the `replace_active_oauth_connection` RPC (atomic revoke-old→insert-new), and the destructive `DELETE FROM oauth_tokens WHERE provider = 'slack'` statement.
- Test suite: 25+ tests covering the full encryption contract (round-trip, random IV per call, missing/malformed/wrong-length key, corrupted ciphertext/auth-tag, no plaintext leakage in the envelope or error messages, exact JSONB persistence round-trip).
- `oauth_tokens` (legacy) still holds plaintext Google Drive and Dropbox credentials, confirmed unchanged as of Milestone 4.

## Related Documents

- [../architecture/SECURITY.md](../architecture/SECURITY.md)
- [../architecture/CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md)
- [ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md](ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md)
- [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md)
- [../history/ARCHITECTURE_REVIEW_PHASES.md](../history/ARCHITECTURE_REVIEW_PHASES.md)
