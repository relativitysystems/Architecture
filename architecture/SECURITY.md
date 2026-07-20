# Security Architecture

Source repositories: `relativitysystems/Relativity` and `relativitysystems/AIKB`. Cross-reference [AIKB.md](AIKB.md) for AIKB's own route-level gating and [CONNECTOR_FRAMEWORK.md](CONNECTOR_FRAMEWORK.md) for the Slack-specific security mechanisms summarized here.

## Overview

Security in this platform is layered across distinct, purpose-built mechanisms rather than a single unified auth system: Supabase JWTs for human users, a static shared key for service-to-service calls, HMAC-signed envelopes for the two highest-trust cross-service paths (the Slack event pipeline and AIKB's `/ask` route), and AES-256-GCM envelope encryption for OAuth credentials at rest. Tenant isolation is enforced entirely at the application layer in both repositories — Row-Level Security is technically enabled on every table in both Supabase projects (backlog M11, database-verified), but with zero policies defined anywhere, and every application Supabase client uses the service-role key (which bypasses RLS entirely) — so RLS being "on" provides no practical database-level backstop today. See [docs/supabase/global/RLS.md](../docs/supabase/global/RLS.md) and [docs/supabase/aikb/RLS.md](../docs/supabase/aikb/RLS.md).

## Authentication

| Mechanism | Enforced by | Protects |
|---|---|---|
| Supabase JWT (`Authorization: Bearer <token>`) | `Relativity/middleware/clientAuth.js`, `aikb/middleware/resolveContext.js` (`requireMemberContext`) | Portal user identity. Both repos independently call Supabase's `auth.getUser(token)` — there is no local `jsonwebtoken` signature verification in either repo (a repo-wide search for `jwt.verify` returns zero matches); verification is fully delegated to the Supabase Auth API. |
| Static shared `x-api-key` | `Relativity/middleware/apiKey.js`, inline `requireApiKey` in `aikb/routes/knowledge.js` | Service-to-service calls between Relativity and AIKB, and a small number of externally-automated Relativity routes. One key value is shared by every legitimate caller — see Gaps below. |
| Admin HMAC pseudo-token | `Relativity/middleware/adminAuth.js` | The internal admin console (`/admin/*`) — a single shared password issues a 24-hour HMAC-signed token (`base64url(payload).base64url(HMAC-SHA256(payload, secret))`), not per-user credentials. |
| Slack Events signature | `Relativity/services/slackSignatureService.js` | `POST /api/integrations/slack/events` — `v0=HMAC-SHA256(signingSecret, "v0:{timestamp}:{rawBody}")` over the exact raw request bytes, 300-second replay window, `crypto.timingSafeEqual` comparison. |
| Service-request HMAC envelope | `services/serviceRequestAuth.js` (implemented identically in both repos) | AIKB's `POST /ask`, `POST /chat/redact`, and (backlog H4, completed) 14 more clientId-scoped routes (`/ingest`, `/reindex`, document delete, client delete, documents/collections listing, collections CRUD, and the `/jobs`/`/summary`/`/analytics`/`/stats` reporting routes) — all Relativity → AIKB; and Relativity's `POST /api/integrations/slack/deliver` (AIKB → Relativity, reversed direction). Signature = `HMAC-SHA256(secret, "requestId.issuedAt.expiresAt.clientId.idempotencyKey.sha256(payload)")`, 60-second TTL, `crypto.timingSafeEqual`. `clientId` is read only from the verified envelope, never from the raw request body/URL param, in every direction — routes with `clientId` in their URL path additionally cross-check the param against the envelope (400 on mismatch). `POST /chat/redact` (ADR-007, implemented) reuses this exact mechanism to let Relativity tell AIKB which chat session to redact by `idempotencyKey` after a Slack event reaches `delivery_failed`. See [ADR-004](../decisions/ADR-004-SIGNED-SERVICE-REQUESTS.md). |

**The cron sweep bearer secret (`cronSweepAuthService.js`, `CRON_SECRET`) no longer exists.** `GET /api/integrations/slack/sweep` and its auth were removed outright as part of implementing [ADR-007](../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md)'s bounded-retry/`delivery_failed` design — this is not a deprecated-but-present mechanism; the file and route are gone from `relativitysystems/Relativity`.

## Authorization / Tenant Isolation

Every tenant-scoped table in both repositories carries a `client_id` column, and every query is expected to filter on it explicitly in application code (`.eq('client_id', ...)`). **RLS is enabled but vacuous everywhere, not simply absent (backlog M11, database-verified)** — a repository-wide search for `CREATE POLICY`/`ENABLE ROW LEVEL SECURITY` across every `.sql` migration file in both repos (8 files in Relativity, 6 in AIKB) does return zero matches, but direct live-database inspection (`pg_tables`/`pg_policies`, and Supabase's own `rls_enabled_no_policy` advisor) shows RLS is actually enabled — likely via a Supabase project-level default applied outside of migration files — on all 16 Global-project tables and all 7 AIKB-project tables, with **zero policies on every one of them**. Since RLS-with-no-policy denies all rows to any role subject to RLS, and all Supabase clients in both repos are constructed with the **service-role key** (which has `BYPASSRLS` and ignores RLS entirely regardless of policy count), the net practical effect is identical to RLS being absent: the only role that ever queries these tables bypasses it completely. See [docs/supabase/global/RLS.md](../docs/supabase/global/RLS.md) and [docs/supabase/aikb/RLS.md](../docs/supabase/aikb/RLS.md) for the full per-table audit.

Tenant isolation is therefore a single-layer control: correctness depends entirely on every query author remembering the `client_id` filter, with no database-level backstop against a missed one.

AIKB's vector-search RPC (`match_knowledge_chunks`) is a partial exception in spirit — the `client_id` filter is enforced inside the SQL function itself rather than only in the calling JavaScript — but the value passed in is still application-supplied, not derived from a database session variable (e.g. `auth.uid()`), so it is still an application-layer control, just placed closer to the data.

## Client Isolation Gaps

Previously flagged gap, **resolved by removal (backlog L9):** `GET /api/google-drive/files/:clientId` and `GET /api/google-drive/file/:clientId/:fileId` (`Relativity/routes/api.js`) were gated only by the shared `apiKey` middleware, with `clientId` taken unchecked from the URL path — no check that the caller holding the shared API key was entitled to that specific client. On investigation this was reclassified from a live gap to dead code (see [FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md) L9): `API_KEY` was configured nowhere in the repo (so the routes 401'd unconditionally) and no caller anywhere in the codebase — including the portal's real Google Drive import path (`picker-config` + `import`, a different, unrelated flow) — ever reached them. Both routes have now been deleted outright rather than wired up, closing the question either way.

AIKB's own `x-api-key`-gated routes (ingest/list/delete under `/api/knowledge`) — **resolved (backlog H4, completed).** All 14 of 14 routes (`/ingest`, `/reindex`, document delete, client delete, documents/collections listing, collections CRUD, and the `/jobs`, `/summary`, `/analytics`, `/stats` reporting routes) now additionally require the signed HMAC envelope already used by `/ask` (`requireServiceRequest`), which cryptographically binds `clientId` to the request — a leaked shared key alone can no longer act on an arbitrary client through these routes. They also still perform their own defense-in-depth ownership check (`doc.client_id !== clientId → 403`) where applicable. The four reporting routes were the item's last residual scope, closed in a second pass using the same GET-with-signed-JSON-body pattern already established for `/documents/:clientId` and `/collections/:clientId`. See [FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md) H4, [ADR-004](../decisions/ADR-004-SIGNED-SERVICE-REQUESTS.md).

## Supabase RLS

**Resolved (backlog M11) — enabled everywhere, verified live, with the "zero policies" caveat documented.** See Authorization above and the per-project audits: [docs/supabase/global/RLS.md](../docs/supabase/global/RLS.md), [docs/supabase/aikb/RLS.md](../docs/supabase/aikb/RLS.md). Every existing migration that introduces a new table includes an explicit code comment noting RLS as a tracked-but-not-yet-implemented improvement in the migration's own text — that comment is now stale/inaccurate on the "not yet implemented" point (RLS is in fact already enabled at the project level, just with no explicit policies), though the substance of the concern it raises (no policy-driven backstop beyond the service-role bypass) remains accurate.

## Collection Permissions

Collection-level access control is implemented and enforced inside AIKB's retrieval SQL itself (see [AIKB.md](AIKB.md)), not merely filtered after the fact in application code:

- The Slack query path defaults `allowedCollectionIds` to an **empty array** if the caller does not supply an explicit array — fail-closed, meaning "search nothing" rather than "search everything."
- The allow-list is managed per client via the `slack_collection_access` table, exposed through `GET/PUT /api/integrations/slack/collections`, with the write route additionally gated by `requireOwnerAdmin`.
- The portal chat path (`/query`) defaults `allowedCollectionIds` to `null` — no restriction, by design, since a portal user is already scoped to their own client's full workspace.

There is currently no per-member or per-role collection restriction within the portal itself — collections today only gate Slack's retrieval scope, not the in-portal chat's retrieval scope.

## Secrets Management

All secrets are read from environment variables only; no secrets-manager or vault integration exists in either repository. Notable variables include (Relativity): `ADMIN_PASSWORD`, `ADMIN_JWT_SECRET`, `GLOBAL_SUPABASE_SERVICE_ROLE_KEY`, `AIKB_API_KEY`, `DROPBOX_APP_SECRET`, `SLACK_CLIENT_SECRET`/`SLACK_SIGNING_SECRET`, `INTEGRATION_CREDENTIAL_ENCRYPTION_KEY` (with a deprecated fallback to the older `SLACK_TOKEN_ENCRYPTION_KEY`), `SERVICE_REQUEST_SIGNING_SECRET`, `GOOGLE_CLIENT_SECRET`; and (AIKB): `AIKB_SUPABASE_SERVICE_KEY`, `GLOBAL_SUPABASE_SERVICE_KEY`, `OPENAI_API_KEY`, `INNGEST_SIGNING_KEY`, `API_KEY`, `SLACK_SIGNING_SECRET`. `CRON_SECRET` was removed from Relativity's `.env.example` along with the sweep endpoint it protected ([ADR-007](../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md)).

**OAuth credentials at rest (Slack, Google Drive, and Dropbox — backlog H2)** are encrypted with AES-256-GCM in `Relativity/services/integrationCredentialEncryption.js`:

```js
const ALGORITHM = 'aes-256-gcm';
const KEY_BYTES = 32; const IV_BYTES = 12; const AUTH_TAG_BYTES = 16;

function encryptCredential(plaintext) {
  const key = validateEncryptionKey();
  const iv = crypto.randomBytes(IV_BYTES);
  const cipher = crypto.createCipheriv(ALGORITHM, key, iv, { authTagLength: AUTH_TAG_BYTES });
  const ciphertext = Buffer.concat([cipher.update(plaintext, 'utf8'), cipher.final()]);
  const authTag = cipher.getAuthTag();
  return { version: ENVELOPE_VERSION, algorithm: ALGORITHM, iv: iv.toString('base64'), authTag: authTag.toString('base64'), ciphertext: ciphertext.toString('base64') };
}
```

The key is validated as exactly 64 hex characters (32 bytes) and re-read from `process.env` on every call rather than a cached config snapshot; decryption verifies the GCM auth tag and fails closed with a generic error on mismatch — there is no plaintext fallback. ~~**Key rotation is not implemented**~~ — **implemented (backlog M3).** `encryption_key_version` is read from the new, optional `INTEGRATION_CREDENTIAL_ENCRYPTION_KEY_VERSION` env var (defaulting to `1`, so every pre-existing environment needs no config change) rather than a hard-coded constant; `decryptCredential` now resolves the key matching each row's own stored version instead of always the current one — fixing a latent bug where `getDecryptedCredentialForConnection` read the column but never passed it through, so a completed rotation would have silently broken decryption for not-yet-re-encrypted rows. `oauthConnectionsService.js#reencryptCredentialForConnection` plus `scripts/rotateCredentialEncryptionKey.js` drive an actual rotation to completion. See [FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md) M3.

**This encryption now applies to all three providers this repo writes.** Google Drive and Dropbox were migrated onto the same `oauth_connections`/`oauth_credentials` model Slack already used (backlog H2) — the legacy plaintext `oauth_tokens` table is no longer written to by any of them; `upsertToken`/`getToken` remain in `supabaseService.js` only because the table itself hasn't been dropped, not because anything still calls them.

## API Security

- **CORS**: no `cors` middleware or package dependency exists in either repository; both apps mount only `express.json()`. No explicit cross-origin policy is configured.
- **Rate limiting**: no general-purpose rate-limiting middleware exists in either repository. The only rate limit found is a hand-rolled, in-memory (non-distributed, process-local) limiter on password-reset requests (3 per email per 10-minute window) — it resets on every redeploy and does not protect any other endpoint, including admin login.
- **Input validation**: ad hoc per-route checks (UUID regex on `clientId`, name-length caps, email regex) rather than a shared validation layer. File uploads are constrained via `multer` `fileFilter` plus extension/MIME allow-lists.
- **Request size limits**: `multer` enforces per-upload-type size caps (file/audio/ZIP). Neither `app.js` nor `aikb/server.js` sets an explicit JSON body-size limit — both rely on Express's default (100kb).

## Portal Security

- **Session handling**: the frontend uses the Supabase JS client's own session management (default localStorage-backed) and sends the resulting access token as a `Bearer` header on every API call; the portal code does not implement custom cookie or token storage.
- **Password reset**: always returns a generic success response regardless of whether the account exists (anti-enumeration), rate-limited, generates the recovery link server-side via Supabase Admin, and never logs the link URL. Client-side enforces a minimum password length of 8 characters; no additional server-side password policy was found beyond Supabase Auth's own defaults.
- **Owner invite / team invite flows**: owner invites embed `client_id` in Supabase user metadata at creation time, read server-side only (never trusted from the browser) on completion. Team-member invite tokens are 32 random bytes, hex-encoded, with a 7-day expiry and single-use semantics enforced by status-transition guards. ~~The token itself is stored **in plaintext** in `team_invites.token`~~ — **resolved (backlog M2), same pattern as OAuth state below.** `team_invites.token_hash` (SHA-256 hex) replaces the plaintext column entirely; the raw token is still what's generated and emailed, only its at-rest storage changed. Migration applied to production and verified via direct query. See [FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md) M2.
- **OAuth state (CSRF protection on connect flows)**: Slack's flow stores only a SHA-256 hash of a 32-byte random state value, with a 10-minute TTL and atomic single-use consumption. ~~Google Drive's and Dropbox's OAuth `state` parameters remain unsigned base64-encoded JSON~~ — **now identical to Slack's pattern (backlog M1, completed).** Both providers' `/start`/`callback` routes use the same hashed, server-stored, single-use `oauthStateService`. See [FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md) M1.

## Current Risks

1. ~~**Cross-tenant Google Drive access via the shared API key**~~ — **reclassified as dead code, not a live risk.** `GET /api/google-drive/files/:clientId` / `/file/:clientId/:fileId` are indeed gated only by the shared static key with no per-client entitlement check, but `API_KEY` is configured nowhere in either repo's env, so both routes 401 unconditionally — and nothing calls them anyway (the portal's real Drive import path uses `clientAuth` + a browser-obtained Picker token via different routes/services entirely). No active exploit path today. Tracked as dead-code cleanup: [FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md) L9.
2. ~~**Non-constant-time secret comparisons**~~ — **resolved.** The shared API key checks in both repos and the admin password/session-signature checks in `Relativity/middleware/adminAuth.js` and `routes/admin.js` now use `crypto.timingSafeEqual` (length-checked first), matching the pattern already used by the HMAC/cron code. See [FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md) H3.
3. ~~**No RLS anywhere**~~ — **more precise, verified statement (backlog M11): RLS is enabled everywhere but with zero policies**, and since every application Supabase client uses the service-role key (`BYPASSRLS`), it provides no practical backstop today regardless. A single missed `.eq('client_id', ...)` filter in future code is still a full cross-tenant data leak — enabling RLS with no policy doesn't change that outcome, only a real policy scoped to a non-bypassing role would. See [docs/supabase/global/RLS.md](../docs/supabase/global/RLS.md), [docs/supabase/aikb/RLS.md](../docs/supabase/aikb/RLS.md).
4. **No CORS policy and no general rate limiting** on either app, including on the admin login and Supabase-backed portal login paths (only password-reset has any throttling, and it is process-local/non-distributed).
5. **Unsigned OAuth state for Google Drive and Dropbox**, unlike the hardened, hashed, server-stored state design already built and proven for Slack. Not addressed by backlog H2 (which migrated token *storage*, not the *state* parameter) — tracked separately as M1.
6. **No credential encryption-key rotation mechanism** — a compromised or rotated `INTEGRATION_CREDENTIAL_ENCRYPTION_KEY` would require manual re-encryption of every stored row; no automated process exists.
7. **Plaintext team-invite tokens at rest** — unlike OAuth state (hash-only), the 32-byte team-invite token is stored unhashed and usable directly by anyone with read access to that table row.
8. ~~**Google Drive and Dropbox OAuth tokens remain plaintext at rest**~~ — **resolved (backlog H2).** Both now write through the same encrypted `oauth_connections`/`oauth_credentials` model as Slack. See [ADR-006](../decisions/ADR-006-OAUTH-CREDENTIAL-ENCRYPTION.md).
9. ~~**4 of AIKB's clientId-scoped routes remain shared-key-only** (`/jobs`, `/summary`, `/analytics`, `/stats`)~~ — **resolved (backlog H4, completed).** All 14 of AIKB's `x-api-key`-only routes now also require the signed HMAC envelope. See item 33 above (Client Isolation Gaps) and [FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md) H4.

## Future Improvements

- ~~Extend AES-256-GCM encrypted storage (`oauth_connections`/`oauth_credentials`) to Google Drive and Dropbox~~ — done, backlog H2. `oauth_tokens` itself has not been dropped (kept as a defensive landing spot; confirmed empty for every provider as of this writing).
- Remove or properly wire up the orphaned Google Drive `files`/`file` routes (see Current Risks item 1 above).
- ~~Introduce Row-Level Security as defense-in-depth on the highest-sensitivity tables (`oauth_tokens`, `oauth_credentials`, `knowledge_chunks`), without removing the existing application-layer checks~~ — **verified already true (backlog M11).** RLS is enabled on both tables in the Global project and on `knowledge_chunks` in AIKB (in fact on every table in both projects), with existing application-layer checks fully unchanged. The remaining real gap — no policy scoped to a non-bypassing role, only a `service_role` bypass — is a materially different, larger undertaking (would require introducing a non-service-role Supabase client somewhere, which no code path currently does) and is not what this item asked for.
- Add general-purpose rate limiting (login, admin login, and the AIKB API-key-gated routes) and an explicit CORS policy.
- Backport the hashed, server-stored OAuth-state pattern from Slack to Google Drive and Dropbox.
- Introduce key-rotation support for `INTEGRATION_CREDENTIAL_ENCRYPTION_KEY`, using the already-reserved `encryption_key_version` column.
- Hash team-invite tokens at rest, matching the OAuth-state pattern.
- ~~Extend backlog H4's signed-envelope requirement to AIKB's remaining 4 shared-key-only routes (`/jobs`, `/summary`, `/analytics`, `/stats`)~~ — done, backlog H4 (completed).
