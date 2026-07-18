# ADR-004: Signed service requests between Relativity and AIKB

## Status
Partially Implemented — a narrow, additive envelope exists for one route pair; the full platform described below remains **Proposed**.

## Date
Undated (proposal); narrow implementation shipped 2026-07-14 to 2026-07-16 (Milestone 4)

## Context

Every route under AIKB's `/api/knowledge` sits behind one static, shared, non-expiring `x-api-key` (`requireApiKey`). This key alone grants broad trust: any holder can read/ingest/delete any client's documents on the routes that rely only on it, because the check confirms `clientId` is *active*, not that the caller is *entitled* to that specific client. `/query`, `/chat/*`, and `/gaps` additionally require a human Supabase JWT (`requireMemberContext`), which is a meaningfully stronger check — but it only exists on those three routes, and it cannot be produced by a non-human caller like a Slack event handler.

When Slack's Milestone 4 work needed AIKB to trust a per-request `clientId` and idempotency key from a caller that is not a human with a JWT, the shared API key alone was insufficient — it would have been the first machine-to-machine caller AIKB ever trusted with a client-scoped write, with no additional integrity guarantee beyond "holds the same static secret every other caller holds."

## Decision

**A minimal, additive HMAC-signed envelope was built, scoped only to `POST /api/knowledge/ask` and its `POST /api/integrations/slack/deliver` reverse callback** — not a rewrite of the shared `x-api-key`, and not the full future signed `ServiceRequest` platform originally proposed (no `entitledCollectionIds`, no multi-origin principal registry, no asymmetric signing, no contract versioning).

Envelope shape: `{ requestId, issuedAt, expiresAt, clientId, idempotencyKey, signature }`, signed with `HMAC-SHA256` over `requestId.issuedAt.expiresAt.clientId.idempotencyKey.sha256(payload)`, 60-second TTL, verified with `crypto.timingSafeEqual`. New shared secret: `SERVICE_REQUEST_SIGNING_SECRET`, distinct from `AIKB_API_KEY`, `SLACK_SIGNING_SECRET`, and `INTEGRATION_CREDENTIAL_ENCRYPTION_KEY`.

**The full future platform remains proposed, not built:** a unified signed envelope carrying a resolved principal, signed `entitledCollectionIds`, `origin`, contract-versioning fields (`schemaVersion`, `issuer`, `audience`), and replacing the shared `x-api-key` on every route — including `/ingest`, `/reindex`, `/documents/:clientId`, `/jobs/:clientId`, `/summary/:clientId`, `/analytics/:clientId`, and `DELETE /client/:clientId`, all of which still rely solely on the shared key today.

## Alternatives Considered

- **Extend the shared `x-api-key` model as-is to Slack traffic**: rejected — it provides no caller-identity guarantee beyond "holds the secret," which is not sufficient for the first non-human, client-scoped write path.
- **Build the full future `ServiceRequest` platform immediately** (signed `entitledCollectionIds`, multi-origin principal resolution, contract versioning): rejected for the Slack MVP — there is no collection-entitlement concept to sign yet at this stage, and building the full platform without an end-to-end-testable consumer would leave nothing verifiable. Deferred as future work, tracked below.
- **A JWT-based service identity** (mirroring `requireMemberContext`'s machinery): considered but not chosen for the MVP scope — a narrower, purpose-built HMAC envelope was judged sufficient and faster to ship correctly for exactly one route pair.

## Consequences

- `/ask` and `/deliver` have a materially stronger authentication guarantee than every other AIKB management route — a real, if narrow, inconsistency in the platform's security posture that is explicitly tracked, not hidden.
- Refactoring the remaining shared-`x-api-key`-only routes onto a similar (or the same) signed-envelope model is a separate, tracked hardening item — not silently resolved by this decision. See [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md) item H4.
- The envelope has no built-in rotation story (`SERVICE_REQUEST_SIGNING_SECRET` cannot be rotated without a coordinated deploy to both repositories) — accepted as a known gap for the MVP.
- Any future connector needing a similar non-human, client-scoped call (Teams, Gmail push notifications) can reuse this same envelope shape rather than inventing a new one, but doing so for a *collection-scoped* call still requires building out the signed-`entitledCollectionIds` piece that was deliberately deferred here.

## Implementation Evidence

- `services/serviceRequestAuth.js`, byte-for-byte identical in both repositories (signing-string format must match exactly).
- AIKB: `middleware/serviceRequest.js` (`requireServiceRequest`), gating `POST /ask` in addition to the existing `requireApiKey`.
- Relativity: `middleware/requireServiceRequest.js`, gating `POST /api/integrations/slack/deliver` (reversed direction — AIKB signs, Relativity verifies).
- Confirmed live in production between the two deployed services as of 2026-07-16.
- No `entitledCollectionIds`, principal registry, or contract versioning exists anywhere in either repository as of this writing.

## Related Documents

- [../architecture/SERVICE_CONTRACTS.md](../architecture/SERVICE_CONTRACTS.md)
- [../architecture/SECURITY.md](../architecture/SECURITY.md)
- [ADR-003-SLACK-EVENTS-LIVE-IN-RELATIVITY.md](ADR-003-SLACK-EVENTS-LIVE-IN-RELATIVITY.md)
- [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md)
- [../history/ARCHITECTURE_REVIEW_PHASES.md](../history/ARCHITECTURE_REVIEW_PHASES.md)
