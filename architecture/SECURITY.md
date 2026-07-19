# Security Architecture

Source repositories: `relativitysystems/Relativity` and `relativitysystems/AIKB`. Cross-reference [AIKB.md](AIKB.md) for AIKB's own route-level gating and [CONNECTOR_FRAMEWORK.md](CONNECTOR_FRAMEWORK.md) for the Slack-specific security mechanisms summarized here.

## Overview

Security in this platform is layered across distinct, purpose-built mechanisms rather than a single unified auth system: Supabase JWTs for human users, a static shared key for service-to-service calls, HMAC-signed envelopes for the two highest-trust cross-service paths (the Slack event pipeline and AIKB's `/ask` route), and AES-256-GCM envelope encryption for OAuth credentials at rest. Tenant isolation is enforced entirely at the application layer in both repositories — there is no database-level Row-Level Security anywhere in this schema.

## Authentication

| Mechanism | Enforced by | Protects |
|---|---|---|
| Supabase JWT (`Authorization: Bearer <token>`) | `Relativity/middleware/clientAuth.js`, `aikb/middleware/resolveContext.js` (`requireMemberContext`) | Portal user identity. Both repos independently call Supabase's `auth.getUser(token)` — there is no local `jsonwebtoken` signature verification in either repo (a repo-wide search for `jwt.verify` returns zero matches); verification is fully delegated to the Supabase Auth API. |
| Static shared `x-api-key` | `Relativity/middleware/apiKey.js`, inline `requireApiKey` in `aikb/routes/knowledge.js` | Service-to-service calls between Relativity and AIKB, and a small number of externally-automated Relativity routes. One key value is shared by every legitimate caller — see Gaps below. |
| Admin HMAC pseudo-token | `Relativity/middleware/adminAuth.js` | The internal admin console (`/admin/*`) — a single shared password issues a 24-hour HMAC-signed token (`base64url(payload).base64url(HMAC-SHA256(payload, secret))`), not per-user credentials. |
| Slack Events signature | `Relativity/services/slackSignatureService.js` | `POST /api/integrations/slack/events` — `v0=HMAC-SHA256(signingSecret, "v0:{timestamp}:{rawBody}")` over the exact raw request bytes, 300-second replay window, `crypto.timingSafeEqual` comparison. |
| Cron sweep bearer secret (deprecated) | `Relativity/services/cronSweepAuthService.js` | `GET /api/integrations/slack/sweep` — strict `Bearer <CRON_SECRET>`, fails **closed** (503) if unconfigured, distinct from a 401 mismatch; `crypto.timingSafeEqual`. The sweep endpoint this protects is superseded by the bounded-retry/`delivery_failed` design ([ADR-007](../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md)) and is targeted for removal, not restoration — this row documents the current, still-live code, not the target architecture. |
| Service-request HMAC envelope | `services/serviceRequestAuth.js` (implemented identically in both repos) | The two highest-trust machine-to-machine calls: AIKB's `POST /ask` (Relativity → AIKB) and Relativity's `POST /api/integrations/slack/deliver` (AIKB → Relativity, reversed direction). Signature = `HMAC-SHA256(secret, "requestId.issuedAt.expiresAt.clientId.idempotencyKey.sha256(payload)")`, 60-second TTL, `crypto.timingSafeEqual`. `clientId` is read only from the verified envelope, never from the raw request body, in both directions. |

## Authorization / Tenant Isolation

Every tenant-scoped table in both repositories carries a `client_id` column, and every query is expected to filter on it explicitly in application code (`.eq('client_id', ...)`). **There is no Row-Level Security anywhere in either repository** — a repository-wide search for `CREATE POLICY` and `ENABLE ROW LEVEL SECURITY` across every `.sql` migration file in both repos (8 files in Relativity, 6 in AIKB) returns zero actual policy statements; the only occurrences of either phrase are migration comments explicitly noting their absence. All Supabase clients in both repos are constructed with the **service-role key**, which bypasses RLS even where it might otherwise exist.

Tenant isolation is therefore a single-layer control: correctness depends entirely on every query author remembering the `client_id` filter, with no database-level backstop against a missed one.

AIKB's vector-search RPC (`match_knowledge_chunks`) is a partial exception in spirit — the `client_id` filter is enforced inside the SQL function itself rather than only in the calling JavaScript — but the value passed in is still application-supplied, not derived from a database session variable (e.g. `auth.uid()`), so it is still an application-layer control, just placed closer to the data.

## Client Isolation Gaps

One confirmed gap: `GET /api/google-drive/files/:clientId` and `GET /api/google-drive/file/:clientId/:fileId` (`Relativity/routes/api.js`) are gated only by the shared `apiKey` middleware, with `clientId` taken unchecked from the URL path — there is no check that the caller holding the shared API key is entitled to that specific client. Any holder of the single `API_KEY` value can enumerate or download **any** client's connected Google Drive files by varying the URL parameter.

By contrast, AIKB's own `x-api-key`-gated routes (ingest/list/delete under `/api/knowledge`) are deliberately documented as relying on Relativity (the sole intended caller) to have already enforced per-client entitlement, and additionally perform their own defense-in-depth ownership check (`doc.client_id !== clientId → 403`) before acting — this is a documented trust assumption, not an unrecognized gap, provided Relativity itself validates entitlement upstream. The Google Drive route above is the one place that assumption is not honored on the Relativity side.

## Supabase RLS

Confirmed absent in both repositories (see Authorization above). Every existing migration that introduces a new table includes an explicit code comment acknowledging this and stating that RLS as defense-in-depth is a tracked but not-yet-implemented improvement, so as not to introduce an inconsistent pattern relative to the rest of the schema.

## Collection Permissions

Collection-level access control is implemented and enforced inside AIKB's retrieval SQL itself (see [AIKB.md](AIKB.md)), not merely filtered after the fact in application code:

- The Slack query path defaults `allowedCollectionIds` to an **empty array** if the caller does not supply an explicit array — fail-closed, meaning "search nothing" rather than "search everything."
- The allow-list is managed per client via the `slack_collection_access` table, exposed through `GET/PUT /api/integrations/slack/collections`, with the write route additionally gated by `requireOwnerAdmin`.
- The portal chat path (`/query`) defaults `allowedCollectionIds` to `null` — no restriction, by design, since a portal user is already scoped to their own client's full workspace.

There is currently no per-member or per-role collection restriction within the portal itself — collections today only gate Slack's retrieval scope, not the in-portal chat's retrieval scope.

## Secrets Management

All secrets are read from environment variables only; no secrets-manager or vault integration exists in either repository. Notable variables include (Relativity): `ADMIN_PASSWORD`, `ADMIN_JWT_SECRET`, `GLOBAL_SUPABASE_SERVICE_ROLE_KEY`, `AIKB_API_KEY`, `DROPBOX_APP_SECRET`, `SLACK_CLIENT_SECRET`/`SLACK_SIGNING_SECRET`, `INTEGRATION_CREDENTIAL_ENCRYPTION_KEY` (with a deprecated fallback to the older `SLACK_TOKEN_ENCRYPTION_KEY`), `SERVICE_REQUEST_SIGNING_SECRET`, `CRON_SECRET`, `GOOGLE_CLIENT_SECRET`; and (AIKB): `AIKB_SUPABASE_SERVICE_KEY`, `GLOBAL_SUPABASE_SERVICE_KEY`, `OPENAI_API_KEY`, `INNGEST_SIGNING_KEY`, `API_KEY`, `SLACK_SIGNING_SECRET`.

**OAuth credentials at rest (Slack only)** are encrypted with AES-256-GCM in `Relativity/services/integrationCredentialEncryption.js`:

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

The key is validated as exactly 64 hex characters (32 bytes) and re-read from `process.env` on every call rather than a cached config snapshot; decryption verifies the GCM auth tag and fails closed with a generic error on mismatch — there is no plaintext fallback. **Key rotation is not implemented**; `encryption_key_version` is stored per credential but hard-coded to `1` everywhere today.

**This encryption applies to Slack only.** Google Drive and Dropbox tokens remain in the legacy `oauth_tokens` table **in plaintext**, via functions explicitly marked `@deprecated for new providers` but still the only live path for these two providers.

## API Security

- **CORS**: no `cors` middleware or package dependency exists in either repository; both apps mount only `express.json()`. No explicit cross-origin policy is configured.
- **Rate limiting**: no general-purpose rate-limiting middleware exists in either repository. The only rate limit found is a hand-rolled, in-memory (non-distributed, process-local) limiter on password-reset requests (3 per email per 10-minute window) — it resets on every redeploy and does not protect any other endpoint, including admin login.
- **Input validation**: ad hoc per-route checks (UUID regex on `clientId`, name-length caps, email regex) rather than a shared validation layer. File uploads are constrained via `multer` `fileFilter` plus extension/MIME allow-lists.
- **Request size limits**: `multer` enforces per-upload-type size caps (file/audio/ZIP). Neither `app.js` nor `aikb/server.js` sets an explicit JSON body-size limit — both rely on Express's default (100kb).

## Portal Security

- **Session handling**: the frontend uses the Supabase JS client's own session management (default localStorage-backed) and sends the resulting access token as a `Bearer` header on every API call; the portal code does not implement custom cookie or token storage.
- **Password reset**: always returns a generic success response regardless of whether the account exists (anti-enumeration), rate-limited, generates the recovery link server-side via Supabase Admin, and never logs the link URL. Client-side enforces a minimum password length of 8 characters; no additional server-side password policy was found beyond Supabase Auth's own defaults.
- **Owner invite / team invite flows**: owner invites embed `client_id` in Supabase user metadata at creation time, read server-side only (never trusted from the browser) on completion. Team-member invite tokens are 32 random bytes, hex-encoded, with a 7-day expiry and single-use semantics enforced by status-transition guards — but the token itself is stored **in plaintext** in `team_invites.token`, unlike the newer OAuth state pattern below which stores only a hash.
- **OAuth state (CSRF protection on connect flows)**: Slack's flow stores only a SHA-256 hash of a 32-byte random state value, with a 10-minute TTL and atomic single-use consumption. Google Drive's and Dropbox's OAuth `state` parameters remain unsigned base64-encoded JSON, not HMAC-signed or server-stored — the same pattern the Slack flow explicitly replaced for Slack, but not yet backported to these two remaining providers.

## Current Risks

1. **Cross-tenant Google Drive access via the shared API key** — `GET /api/google-drive/files/:clientId` / `/file/:clientId/:fileId` are gated only by the shared static key, with no per-client entitlement check on the `clientId` path parameter.
2. **Non-constant-time secret comparisons** in several places, contrasted with the newer HMAC/cron code that correctly uses `crypto.timingSafeEqual`: the shared API key checks in both repos (`key !== process.env.API_KEY` style), and the admin password/session-signature checks in `Relativity/middleware/adminAuth.js` and `routes/admin.js`.
3. **No RLS anywhere**, combined with universal service-role-key usage — tenant isolation has no database-level backstop; a single missed `.eq('client_id', ...)` filter in future code is a full cross-tenant data leak.
4. **No CORS policy and no general rate limiting** on either app, including on the admin login and Supabase-backed portal login paths (only password-reset has any throttling, and it is process-local/non-distributed).
5. **Unsigned OAuth state for Google Drive and Dropbox**, unlike the hardened, hashed, server-stored state design already built and proven for Slack.
6. **No credential encryption-key rotation mechanism** — a compromised or rotated `INTEGRATION_CREDENTIAL_ENCRYPTION_KEY` would require manual re-encryption of every stored row; no automated process exists.
7. **Plaintext team-invite tokens at rest** — unlike OAuth state (hash-only), the 32-byte team-invite token is stored unhashed and usable directly by anyone with read access to that table row.
8. **Google Drive and Dropbox OAuth tokens remain plaintext at rest** in the legacy `oauth_tokens` table, in contrast to Slack's AES-256-GCM-encrypted storage.

## Future Improvements

- Extend AES-256-GCM encrypted storage (`oauth_connections`/`oauth_credentials`) to Google Drive and Dropbox, retiring the legacy plaintext `oauth_tokens` path entirely.
- Add a per-client entitlement check to the Google Drive `files`/`file` routes, closing the shared-API-key cross-tenant gap.
- Replace non-constant-time secret comparisons with `crypto.timingSafeEqual` consistently across both repositories.
- Introduce Row-Level Security as defense-in-depth on the highest-sensitivity tables (`oauth_tokens`, `oauth_credentials`, `knowledge_chunks`), without removing the existing application-layer checks.
- Add general-purpose rate limiting (login, admin login, and the AIKB API-key-gated routes) and an explicit CORS policy.
- Backport the hashed, server-stored OAuth-state pattern from Slack to Google Drive and Dropbox.
- Introduce key-rotation support for `INTEGRATION_CREDENTIAL_ENCRYPTION_KEY`, using the already-reserved `encryption_key_version` column.
- Hash team-invite tokens at rest, matching the OAuth-state pattern.
