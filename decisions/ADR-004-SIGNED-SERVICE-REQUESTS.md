# ADR-004: Signed service requests between Relativity and AIKB

## Status
Partially Implemented — the additive envelope now covers `/ask`, `/deliver`, and (backlog H4, completed) all 14 of AIKB's clientId-scoped management/reporting routes; no AIKB route relies solely on the shared `x-api-key` anymore (see Consequences), but the full platform described below remains **Proposed**.

## Date
Undated (proposal); narrow implementation shipped 2026-07-14 to 2026-07-16 (Milestone 4); extended to ten management routes 2026-07-19 (backlog H4, first pass); extended to the remaining four reporting routes 2026-07-19 (backlog H4, completed)

## Context

Every route under AIKB's `/api/knowledge` sits behind one static, shared, non-expiring `x-api-key` (`requireApiKey`). This key alone grants broad trust: any holder can read/ingest/delete any client's documents on the routes that rely only on it, because the check confirms `clientId` is *active*, not that the caller is *entitled* to that specific client. `/query`, `/chat/*`, and `/gaps` additionally require a human Supabase JWT (`requireMemberContext`), which is a meaningfully stronger check — but it only exists on those three routes, and it cannot be produced by a non-human caller like a Slack event handler.

When Slack's Milestone 4 work needed AIKB to trust a per-request `clientId` and idempotency key from a caller that is not a human with a JWT, the shared API key alone was insufficient — it would have been the first machine-to-machine caller AIKB ever trusted with a client-scoped write, with no additional integrity guarantee beyond "holds the same static secret every other caller holds."

## Decision

**A minimal, additive HMAC-signed envelope was built, scoped only to `POST /api/knowledge/ask` and its `POST /api/integrations/slack/deliver` reverse callback** — not a rewrite of the shared `x-api-key`, and not the full future signed `ServiceRequest` platform originally proposed (no `entitledCollectionIds`, no multi-origin principal registry, no asymmetric signing, no contract versioning).

Envelope shape: `{ requestId, issuedAt, expiresAt, clientId, idempotencyKey, signature }`, signed with `HMAC-SHA256` over `requestId.issuedAt.expiresAt.clientId.idempotencyKey.sha256(payload)`, 60-second TTL, verified with `crypto.timingSafeEqual`. New shared secret: `SERVICE_REQUEST_SIGNING_SECRET`, distinct from `AIKB_API_KEY`, `SLACK_SIGNING_SECRET`, and `INTEGRATION_CREDENTIAL_ENCRYPTION_KEY`.

**Backlog H4 (2026-07-19, first pass) extended the same narrow envelope — not the full platform — to ten more clientId-scoped routes**: `POST /ingest`, `POST /reindex`, `DELETE /document/:id`, `DELETE /client/:clientId`, `GET /documents/:clientId`, `GET /collections/:clientId`, `POST /collections`, `PATCH /collections/:id`, `DELETE /collections/:id`, and `PATCH /document/:id/collection`. Two new things had no precedent before H4 and were built for it: (1) a GET-with-signed-JSON-body pattern for the two GET routes — `requireServiceRequest` itself needed no changes, since `express.json()` parses `req.body` on every verb; Relativity's caller sends the envelope via axios's `data` option on the GET request — and (2) a URL-param-vs-envelope `clientId` cross-check (400 on mismatch) on every route that also has `clientId` in its URL path, since trust now comes from the envelope, not the param.

**Backlog H4 (2026-07-19, second pass) closed the four remaining routes**: `GET /jobs/:clientId`, `GET /summary/:clientId`, `GET /analytics/:clientId`, `GET /stats/:clientId` — read-only aggregate/reporting endpoints that had the identical underlying trust gap (and `recentKnowledgeGaps`/`failedIngestionJobs` do return sensitive text, e.g. gap question content). These reuse the same GET-with-signed-JSON-body pattern and `clientId` cross-check established in the first pass; no new mechanism was needed. No AIKB clientId-scoped route now relies solely on the shared `x-api-key`.

**The full future platform remains proposed, not built:** a unified signed envelope carrying a resolved principal, signed `entitledCollectionIds`, `origin`, and contract-versioning fields (`schemaVersion`, `issuer`, `audience`) — none of which exist in either repository as of this writing.

## Alternatives Considered

- **Extend the shared `x-api-key` model as-is to Slack traffic**: rejected — it provides no caller-identity guarantee beyond "holds the secret," which is not sufficient for the first non-human, client-scoped write path.
- **Build the full future `ServiceRequest` platform immediately** (signed `entitledCollectionIds`, multi-origin principal resolution, contract versioning): rejected for the Slack MVP — there is no collection-entitlement concept to sign yet at this stage, and building the full platform without an end-to-end-testable consumer would leave nothing verifiable. Deferred as future work, tracked below.
- **A JWT-based service identity** (mirroring `requireMemberContext`'s machinery): considered but not chosen for the MVP scope — a narrower, purpose-built HMAC envelope was judged sufficient and faster to ship correctly for exactly one route pair.

## Consequences

- `/jobs`, `/summary`, `/analytics`, and `/stats` no longer have a weaker authentication guarantee than the rest of AIKB's clientId-scoped routes — the inconsistency H4's first pass left open was closed in its second pass.
- This is a breaking wire-protocol change for all 14 routes it touches: Relativity's callers now sign every request, and AIKB's handlers reject any of those 14 without a valid envelope, with no shared-key-only fallback. Both repositories were updated together in the same work session (for each pass) specifically to avoid a window where one side expects the envelope and the other doesn't send it — but actual production deploy ordering (this repo has no visibility into either service's hosting/CI) is a real operational risk to plan for before pushing, not something this change can guarantee on its own.
- The envelope has no built-in rotation story (`SERVICE_REQUEST_SIGNING_SECRET` cannot be rotated without a coordinated deploy to both repositories) — accepted as a known gap for the MVP.
- Any future connector needing a similar non-human, client-scoped call (Teams, Gmail push notifications) can reuse this same envelope shape rather than inventing a new one, but doing so for a *collection-scoped* call still requires building out the signed-`entitledCollectionIds` piece that was deliberately deferred here.

## Implementation Evidence

- `services/serviceRequestAuth.js`, byte-for-byte identical in both repositories (signing-string format must match exactly).
- AIKB: `middleware/serviceRequest.js` (`requireServiceRequest`), gating `POST /ask` and (backlog H4, completed) 14 more routes in `routes/knowledge.js`, all additive to the existing `requireApiKey`.
- Relativity: `middleware/requireServiceRequest.js`, gating `POST /api/integrations/slack/deliver` (reversed direction — AIKB signs, Relativity verifies); `services/aikbService.js`'s `signedEnvelope()` helper (backlog H4) signs every call to all 14 newly-gated routes.
- Both passes of H4 verified via a live smoke test against a running AIKB server with real (dev) data: confirmed a correctly signed envelope succeeds, a missing envelope 401s, a signature computed for the wrong secret 401s, a tampered payload 401s (proving the signature covers the payload, not just clientId), and a URL-param/envelope `clientId` mismatch 400s.
- No `entitledCollectionIds`, principal registry, or contract versioning exists anywhere in either repository as of this writing.

## Related Documents

- [../architecture/SERVICE_CONTRACTS.md](../architecture/SERVICE_CONTRACTS.md)
- [../architecture/SECURITY.md](../architecture/SECURITY.md)
- [ADR-003-SLACK-EVENTS-LIVE-IN-RELATIVITY.md](ADR-003-SLACK-EVENTS-LIVE-IN-RELATIVITY.md)
- [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md)
- [../history/ARCHITECTURE_REVIEW_PHASES.md](../history/ARCHITECTURE_REVIEW_PHASES.md)
