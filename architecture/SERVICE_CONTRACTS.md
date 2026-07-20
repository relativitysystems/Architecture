# Service Contracts

Source repositories: `relativitysystems/Relativity` and `relativitysystems/AIKB`. This is the canonical document for how the two repositories talk to each other over HTTP. See [SYSTEM_OVERVIEW.md](SYSTEM_OVERVIEW.md) for the platform-level view, [SECURITY.md](SECURITY.md) for the authentication mechanisms in full, and [CONNECTOR_FRAMEWORK.md](CONNECTOR_FRAMEWORK.md) for the Slack-specific routes summarized here.

## Purpose

This document lists every route that crosses the Relativity↔AIKB boundary, what authenticates it today, and what it does. It distinguishes **implemented** behavior from **proposed** behavior — several of these routes were originally designed as part of a much larger signed-request platform that has not been built. Where the report that originated this documentation proposed a contract that later implementation simplified or diverged from, this document describes what was actually shipped; the original proposal is preserved in [../history/ARCHITECTURE_REVIEW_PHASES.md](../history/ARCHITECTURE_REVIEW_PHASES.md).

## Overview: Authentication Models in Use

There is no single unified service-to-service authentication mechanism. Three distinct models coexist, by route:

| Model | Applies to | Mechanism |
|---|---|---|
| Shared static API key | Every route under `/api/knowledge` (router-level) | `x-api-key` header, compared with `!==` (not constant-time) |
| Shared key + member JWT | `/query`, `/chat/*`, `/gaps` | `x-api-key` **plus** `requireMemberContext` — Supabase JWT re-validated against Global DB `client_members` |
| Shared key + signed HMAC envelope | `/ask` (Slack path), and in reverse, Relativity's `/deliver` | `x-api-key` **plus** `requireServiceRequest` — an additive, narrowly-scoped HMAC envelope (`services/serviceRequestAuth.js`, identical in both repos), 60-second TTL, `crypto.timingSafeEqual` |

**Important distinction:** Phase 2 of the original architecture review proposed a full signed `ServiceRequest` envelope carrying `organizationId`, a resolved principal, signed `entitledCollectionIds`, `origin`, and an idempotency key — replacing the shared API key everywhere. **That full platform was never built.** What exists is a much smaller, additive HMAC envelope built specifically for Slack's `/ask`/`/deliver` pair (Milestone 4). Every other AIKB management route (`/ingest`, `/reindex`, `/documents/:clientId`, `/jobs/:clientId`, `/summary/:clientId`, `/analytics/:clientId`, `DELETE /client/:clientId`) still relies solely on the shared `x-api-key` plus Relativity's own upstream entitlement check — this is a known, tracked gap, not a silent oversight. See [SECURITY.md](SECURITY.md) and [ADR-004](../decisions/ADR-004-SIGNED-SERVICE-REQUESTS.md).

## Idempotency

Only two paths carry an idempotency guarantee today:

- **Slack's `/ask` → `/deliver` round trip**: `idempotencyKey` is derived from Slack's own `event_id`, checked against `slack_event_log.UNIQUE(provider, external_event_id)` in Relativity and against a partial unique index on `knowledge_chat_sessions.idempotency_key` in AIKB (belt-and-suspenders — either dedup layer alone would catch a redelivery). This holds even after a `delivery_failed` terminal state — verified by test — and `/chat/redact` (below) reuses the same `idempotencyKey` to identify what to redact, retained specifically to keep this dedup guarantee intact through redaction ([ADR-007](../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md)).
- **`/ingest`**: content-hash deduplication (not a caller-supplied idempotency key) — re-ingesting identical content is a no-op if the hash matches an existing document.

Every other route (`/query`, `/gaps`, `/reindex`, `DELETE /document/:id`) has no idempotency key and no dedup guarantee — a retried request is processed again. `/chat/redact` has no dedup key of its own but is naturally idempotent (redacting twice is a safe no-op).

## Error Handling, Retries, Timeouts

- **Relativity → AIKB (`/ask`)**: `AIKB_ASK_TIMEOUT_MS` (default 4000ms). This call itself is now retried up to 3 total attempts (2s/5s backoff) by `services/slackEventsService.js` before the `slack_event_log` row is marked `delivery_failed` — implemented as part of [ADR-007](../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md), extending its bounded-retry design to this leg specifically so a row is never left permanently stuck at `received` now that the sweep no longer exists. AIKB's own Inngest `onFailure` callback (the AIKB-generation-failure path) remains a separate, unchanged recovery mechanism for the case where AIKB accepted the question but couldn't answer it. See [CONNECTOR_FRAMEWORK.md](CONNECTOR_FRAMEWORK.md).
- **AIKB → Relativity (`/deliver`)**: `RELATIVITY_DELIVER_TIMEOUT_MS` (default 8000ms). AIKB's Inngest function retries the whole step up to 3 times on failure; after retries are exhausted, `onFailure` posts an error payload to `/deliver` so the user isn't left silent.
- **Portal path (`/query`)**: no automatic retry; a failed request surfaces as an error to the browser.
- No route in either direction implements exponential backoff or a circuit breaker.

## Routes

### POST `/api/knowledge/query`

**Owner:** AIKB
**Caller:** Relativity (portal)

**Authentication:** `x-api-key` (router-level) + `requireMemberContext` (Supabase JWT validated against Global DB `client_members`).

**Request:** `{ question, sessionId? }`, `clientId`/`memberId`/`memberRole` resolved server-side from the JWT, never from the request body.

**Response:** `{ answer, sources[], sessionId, isKnowledgeGap, gapReason? }`.

**Authorization rules:** `allowedCollectionIds` defaults to `null` (unrestricted — searches every collection in the client's workspace).

**Idempotency:** none.

**Failure behavior:** synchronous; a failure returns an error response directly to the caller.

**Notes:** internally calls the shared `runKnowledgeQuery({ ..., origin: 'portal' })` function — the same function the Slack path (`/ask`) calls with `origin: 'slack'`. See [AIKB.md](AIKB.md) for the full retrieval pipeline.

---

### POST `/api/knowledge/ask`

**Owner:** AIKB
**Caller:** Relativity (Slack path only)

**Authentication:** `x-api-key` + `requireServiceRequest` (additive HMAC envelope, §Overview above). No human JWT — this is the first machine-to-machine caller AIKB trusts with a client-scoped write without one.

**Request:** signed envelope `{ requestId, issuedAt, expiresAt, clientId, idempotencyKey, signature }` wrapping `{ question, origin: 'slack', originMetadata: { teamId, channelId, threadTs, eventId } }`.

**Response:** `{ accepted: true }` — this route only enqueues; it does not wait for or return the answer.

**Authorization rules:** `allowedCollectionIds` defaults to `[]` if not explicitly supplied — fail-closed (see [ADR-005](../decisions/ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md)). `clientId` is read only from the verified envelope, never the raw request body.

**Idempotency:** `idempotencyKey` derived from Slack's `event_id`; checked against a partial unique index on `knowledge_chat_sessions.idempotency_key` before `runKnowledgeQuery` re-runs retrieval/generation.

**Failure behavior:** enqueues a `knowledge/slack.question.requested` Inngest event and returns immediately; the actual retrieval/generation happens asynchronously. Inngest retries the step up to 3 times; on final failure, `onFailure` calls back to `/deliver` with an error payload.

**Notes:** internally calls the same `runKnowledgeQuery` function `/query` uses — there is no separate Slack retrieval implementation. See [ADR-003](../decisions/ADR-003-SLACK-EVENTS-LIVE-IN-RELATIVITY.md).

---

### POST `/api/integrations/slack/deliver`

**Owner:** Relativity
**Caller:** AIKB (reversed direction)

**Authentication:** `requireServiceRequest`, same HMAC envelope as `/ask` but signed by AIKB and verified by Relativity.

**Request:** `{ idempotencyKey, clientId, answer, sources[], isKnowledgeGap, error? }`.

**Response:** `{ delivered: true }` or a safe error acknowledgement.

**Authorization rules:** looks up the `slack_event_log` row by `idempotencyKey`, rejects a `clientId` mismatch (cross-tenant safety check), and only proceeds if the row's status is `enqueued`.

**Idempotency:** a conditional `UPDATE ... WHERE status = 'enqueued'` claims the row atomically — only the first delivery attempt ever proceeds, verified under concurrent calls in tests. A callback arriving after the row already reached a terminal state (`delivered`, `failed`, or `delivery_failed`) is a safe no-op — verified by test for the `delivery_failed` case specifically.

**Failure behavior (implemented, [ADR-007](../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md)):** a revoked/inactive Slack connection is never used for delivery; `chat.postMessage` failures are mapped to safe internal error codes and logged, never surfaced with raw Slack error text. For a real generated answer (`payload.error` not `true`), a `chat.postMessage` failure is retried up to 3 total attempts (2s/5s backoff, configurable) via `services/retryWithBackoff.js` before the row is marked `delivery_failed` and its `question` redacted; a best-effort, non-retried callback to AIKB's `POST /chat/redact` (below) then redacts the corresponding AIKB-side chat content. For an AIKB-generation-failure notification (`payload.error === true`), behavior is unchanged from before this ADR: a single attempt, the pre-existing generic `failed` status on failure, no redaction.

**Notes:** decrypts the org's Slack bot token in memory immediately before use; the token is never sent to or seen by AIKB.

---

### POST `/api/knowledge/chat/redact`

**Owner:** AIKB
**Caller:** Relativity

**Authentication:** `x-api-key` (router-level) + `requireServiceRequest` (same additive HMAC envelope as `/ask`). `clientId`/`idempotencyKey` come only from the verified envelope, never the request body.

**Request:** signed envelope `{ requestId, issuedAt, expiresAt, clientId, idempotencyKey, signature }` wrapping an empty payload (`{}`) — the envelope alone identifies what to redact.

**Response:** `{ redacted: true, sessionId }` if a matching chat session existed, or `{ redacted: false, reason: 'not_found' }` if none did (e.g. AIKB was never reached for this event).

**Authorization rules:** none beyond the signed envelope — this is a narrowly-scoped, machine-to-machine cleanup call, not a customer-facing route.

**Idempotency:** safe to call more than once for the same `idempotencyKey` — redacting an already-redacted (or never-created) session is a no-op.

**Failure behavior:** called by Relativity's `services/slackDeliveryFailureService.js` immediately after a `slack_event_log` row is marked `delivery_failed`, via `services/aikbRedactClient.js`. **This call is best-effort and single-attempt on Relativity's side — not itself bounded-retried.** If it fails, the failure is logged and swallowed; Relativity's own redaction (the `slack_event_log.question` column) has already happened unconditionally by this point, but the AIKB-side chat session/message content can remain un-redacted with no automatic follow-up, since [ADR-007](../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md) rules out any scheduled recovery process. See that ADR's Implementation Status section for the full tradeoff.

**Notes:** implemented in `aikb/services/supabaseService.js#redactChatSessionByIdempotencyKey` — nulls the session `title` and replaces every message's `content` with a fixed marker (the column is `NOT NULL`) and nulls `sources`/`metadata`. Session/message rows, ids, timestamps, `origin`/`origin_metadata`, and `idempotency_key` are retained.

---

### POST `/api/knowledge/ingest`

**Owner:** AIKB
**Caller:** Relativity

**Authentication:** `x-api-key` only (`requireApiKey`, router-level).

**Request:** `{ clientId, sourceProvider: 'portal_upload', sourceFileId, storagePath }`.

**Response:** `202 { queued: true, eventId }`.

**Authorization rules:** `requireActiveClient` confirms `clientId` is active in the Global DB — does **not** independently verify the caller is entitled to that specific client (relies on Relativity having already checked upstream). See [SECURITY.md](SECURITY.md).

**Idempotency:** content-hash dedup — re-ingesting identical content for the same `(clientId, sourceProvider, sourceFileId)` is a no-op unless `forceReindex` is set.

**Failure behavior:** enqueues `knowledge/document.ingest`; failures surface via `knowledge_ingestion_jobs.status = 'failed'`, polled by the portal.

**Notes:** see [INGESTION_PIPELINE.md](INGESTION_PIPELINE.md) for the full downstream flow.

---

### POST `/api/knowledge/reindex`

**Owner:** AIKB · **Caller:** Relativity
**Authentication:** `x-api-key` only.
**Request/Response:** thin wrapper — validates metadata, re-sends `knowledge/document.ingest` with `forceReindex: true`.
**Authorization rules / Idempotency:** same as `/ingest`, with dedup bypassed by design (`forceReindex`).
**Failure behavior:** same as `/ingest`.

---

### DELETE `/api/knowledge/document/:id`

**Owner:** AIKB · **Caller:** Relativity
**Authentication:** `x-api-key` only.
**Request:** `{ clientId }` (body or query).
**Authorization rules:** AIKB performs its own defense-in-depth ownership check (`doc.client_id !== clientId → 403`) before acting, in addition to the shared-key gate.
**Idempotency:** none beyond the ownership check.
**Failure behavior:** enqueues `knowledge/document.delete` (find-document → delete-chunks → mark-deleted → delete-storage-file).

---

### GET `/api/knowledge/documents/:clientId`, `/jobs/:clientId`, `/summary/:clientId`, `/analytics/:clientId`, `/stats/:clientId`

**Owner:** AIKB · **Caller:** Relativity
**Authentication:** `x-api-key` only.
**Authorization rules:** same trust-Relativity-upstream model as `/ingest` — no independent per-caller entitlement check. See [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md) item H4.
**Notes:** `/stats` (backlog L5) is a superset of `/summary` + `/analytics` + `/jobs`, added so a caller needing more than one of them for the same client (Relativity's admin routes) can get all of it in one round trip instead of independently re-querying the same tables — `getClientKnowledgeStats` computes each underlying table exactly once. `/summary`, `/analytics`, and `/jobs` are unchanged and still used standalone elsewhere (the client portal calls `/analytics` alone). See [../product/KNOWLEDGE_ANALYTICS.md](../product/KNOWLEDGE_ANALYTICS.md).

---

### GET/POST/PATCH/DELETE `/api/knowledge/collections[...]`

**Owner:** AIKB · **Caller:** Relativity (portal, owner/admin only for mutations)
**Authentication:** `x-api-key` only (mutation authorization — owner/admin role — is enforced by Relativity before forwarding).
**Authorization rules:** server-side and DB-level (`ON DELETE RESTRICT`) protection against deleting the default collection or a non-empty collection.
**Notes:** see [AIKB.md](AIKB.md) — Knowledge Collections.

---

### POST `/api/knowledge/gaps`

**Owner:** AIKB · **Caller:** Relativity (portal, explicit user action only)
**Authentication:** `x-api-key` + `requireMemberContext`.
**Request:** `{ sessionId, question, reason }`.
**Authorization rules:** none beyond member context.
**Idempotency:** none — a plain `INSERT`; the `idempotency_key` column exists on `knowledge_gaps` but is not used by this write path. See [../product/KNOWLEDGE_GAP_DETECTION.md](../product/KNOWLEDGE_GAP_DETECTION.md).
**Notes:** this is the **only** write path to `knowledge_gaps` today — neither `/query` nor `/ask` writes a gap automatically.

---

### GET/PATCH/DELETE `/api/knowledge/chat/sessions[...]`

**Owner:** AIKB · **Caller:** Relativity (portal)
**Authentication:** `x-api-key` + `requireMemberContext`.
**Notes:** session-rename (`PATCH .../title`) exists on the backend with no portal UI control — see [../product/CLIENT_PORTAL.md](../product/CLIENT_PORTAL.md).

## Proposed, Not Implemented

The following were proposed during the original architecture review and are **not implemented**. They are preserved here only to make clear they are not current behavior; see [../history/ARCHITECTURE_REVIEW_PHASES.md](../history/ARCHITECTURE_REVIEW_PHASES.md) for the full original proposal.

- A unified, signed `ServiceRequest` envelope (`organizationId`, resolved principal, signed `entitledCollectionIds`, `origin`, idempotency key) replacing the shared `x-api-key` on **every** route. Only a narrow subset (the `/ask`/`/deliver` pair) was built. See [ADR-004](../decisions/ADR-004-SIGNED-SERVICE-REQUESTS.md).
- `POST /resolve-collections` (Relativity asks AIKB to validate a collection-id list before signing it) — not built; there is no signed collection list to validate yet.
- `GET /sources/{answerId}` (re-fetch citation detail after the fact) — not built; citations are only ever returned inline on `/query`/`/ask`.
- Contract-versioning fields (`schemaVersion`, `issuer`, `audience`) on the service-request envelope — not built.

## Related Documents

- [SYSTEM_OVERVIEW.md](SYSTEM_OVERVIEW.md) — platform-level request flows
- [SECURITY.md](SECURITY.md) — full authentication/authorization detail
- [CONNECTOR_FRAMEWORK.md](CONNECTOR_FRAMEWORK.md) — Slack route detail
- [AIKB.md](AIKB.md) — `runKnowledgeQuery` and retrieval detail
- [../decisions/ADR-004-SIGNED-SERVICE-REQUESTS.md](../decisions/ADR-004-SIGNED-SERVICE-REQUESTS.md)
- [../history/ARCHITECTURE_REVIEW_PHASES.md](../history/ARCHITECTURE_REVIEW_PHASES.md)
