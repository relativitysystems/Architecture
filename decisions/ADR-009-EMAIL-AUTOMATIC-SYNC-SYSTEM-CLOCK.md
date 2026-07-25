# ADR-009: AIKB contributes a scheduling "clock" for email automatic sync, via a system-scoped signed envelope

## Status
Implemented — 2026-07-25 (EM8, `EMAIL_INGESTION.md` §18.3, §31).

## Date
2026-07-25

## Context

[ADR-001](ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md) establishes that Relativity, not AIKB, owns every provider integration: AIKB never receives or stores a provider credential and never contains provider-specific logic. This boundary has held without exception since Slack shipped.

The email-ingestion feature's `automatic` sync mode (`EMAIL_INGESTION.md` §15.1) needs *something* to trigger a sync without a member clicking "Sync now" — but neither repository has ever had a scheduler. Relativity has none today, and [ADR-007](ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md) deliberately removed the one scheduled job (Slack's redelivery sweep) it ever had, choosing bounded in-flow retries instead specifically to avoid the operational cost of running a scheduler. AIKB, by contrast, already runs an always-on Express process with Inngest, which natively supports cron triggers — a capability that exists in this codebase for the first time only because AIKB already needed Inngest for its document-ingestion pipeline.

The email-ingestion architecture document (`EMAIL_INGESTION.md`) flagged this as a real, if narrow, exception to ADR-001's boundary from the moment EM8 was first scoped, and required (§31 EM8 entry) that this ADR be written as part of EM8's own deliverable, not deferred further.

## Decision

**AIKB contributes exactly one thing to email automatic sync: a clock.** A new, cron-triggered Inngest function (`email-sync-tick`, `aikb/inngest/functions.js`) fires on a fixed interval (`EMAIL_SYNC_TICK_CRON_SCHEDULE`, default every 20 minutes) and makes exactly one HTTP call to Relativity's `POST /api/integrations/email/sync/tick`. That call carries **zero client-specific or provider-specific data** — AIKB does not know which clients have email connections, never touches `oauth_connections`, and never becomes provider-aware. All real work (enumerating due `automatic`-mode connections, fetching mail, evaluating policy, ingesting) happens entirely inside Relativity's handler (`emailSyncService.runTick`), reusing the exact same code path a member's manual "Sync now" click already used.

The call is authenticated by a **system-scoped variant of the existing signed service-request envelope** ([ADR-004](ADR-004-SIGNED-SERVICE-REQUESTS.md)): `signSystemServiceRequest`/`verifySystemServiceRequest`, added to `services/serviceRequestAuth.js` in both repositories. Mechanically identical to the existing HMAC/TTL envelope, but the signing string's `clientId` slot is always the hardcoded literal `'SYSTEM'`, never a caller-supplied field — so a system-scoped envelope can never be replayed as a valid clientId-scoped envelope, or vice versa; the two are structurally distinct inputs to the same HMAC, not one envelope with an optional field. A dedicated Relativity middleware, `requireSystemServiceRequest`, verifies it and attaches `req.systemRequest = {requestId, idempotencyKey}` — no `clientId`, by construction.

## Alternatives Considered

- **Build a scheduler inside Relativity instead** (e.g. Vercel Cron, a new always-on process): rejected. This is precisely the operational surface ADR-007 already argued against taking on for a narrower need (Slack redelivery); email automatic sync doesn't change that calculus, and AIKB already pays this operational cost today for its own ingestion pipeline.
- **Let AIKB's tick carry a `clientId` list or otherwise become provider-/client-aware** (e.g. AIKB queries which clients have automatic sync enabled and tells Relativity): rejected — this would be a much larger breach of ADR-001 than "contributes a clock," turning AIKB into a second source of truth for email-connection state it has no other reason to know about.
- **Provider push notifications** (Gmail Pub/Sub `watch`, Graph change-notification webhooks) as the triggering mechanism instead of polling: rejected for this milestone, not merely deferred — push solves latency, not the actual blocking gap (having no trigger at all), at higher operational cost (topic/subscription management, renewal cycles). `EMAIL_INGESTION.md`'s own Decision Log covers this in more detail.
- **Reuse the existing clientId-scoped `signServiceRequest`/`verifyServiceRequest` with `clientId` made optional**: rejected — an optional field one caller correctly omits and another might accidentally supply (or a route accidentally accept) is a weaker guarantee than two structurally separate functions that make the client-scoped/system-scoped distinction impossible to blur by omission.

## Consequences

- This is now the second, and only other, narrow exception to ADR-001's "AIKB owns nothing provider-specific" boundary — the first being none (Slack's delivery callback carries `clientId` but no provider credential or provider-specific logic either, so it never needed this ADR). Any future connector needing a similar unattended trigger should reuse this same system-scoped envelope shape rather than inventing a new one.
- AIKB gains its first cron-triggered function and, with it, a small new operational dependency: if AIKB's Inngest app is down or misconfigured, `automatic`-mode email connections silently stop receiving unattended syncs (manual "Sync now" is unaffected — it never depended on the tick). No alerting for this exists in either repository today, consistent with the platform's documented absence of monitoring/alerting infrastructure generally (`SECURITY.md`).
- `SERVICE_REQUEST_SIGNING_SECRET` now gates three call shapes instead of two (`/ask`, `/deliver`, and system-scoped `/sync/tick`) — no new secret was introduced; the existing shared secret already crosses both repositories.
- A tick that fails partway (e.g. a transient AIKB-to-Relativity network failure) simply waits for the next interval — there is no retry-the-same-tick mechanism, deliberately, mirroring ADR-007's "bounded retry, not a scheduler resurrection" philosophy at one layer up.

## Implementation Evidence

- `Relativity/services/serviceRequestAuth.js` / `aikb/services/serviceRequestAuth.js` — `signSystemServiceRequest`/`verifySystemServiceRequest`, byte-for-byte identical signing-string logic in both repositories (per the existing convention this file's header comment already documents for the clientId-scoped envelope).
- `Relativity/middleware/requireSystemServiceRequest.js` — gates `POST /api/integrations/email/sync/tick` (`routes/integrations/email.js`), the only route in that file with a different auth shape than `clientAuth`.
- `aikb/services/relativityTickClient.js` — signs and issues the outbound call; `aikb/inngest/functions.js`'s `emailSyncTick` — the cron-triggered function itself, no concurrency key (a single global tick, not a per-client job).
- Tests: `Relativity/test/serviceRequestAuth.test.js` and `aikb/test/serviceRequestAuth.test.js` (system-envelope round-trip, tamper/expiry, and — the point of this ADR — explicit cross-envelope-isolation tests proving a clientId-scoped envelope is rejected by the system-scoped verifier and vice versa); `Relativity/test/emailRoutes.test.js` (HTTP-level: no envelope, forged signature, and a well-formed-but-wrong-envelope-type request all 401 on `/sync/tick`); `aikb/test/relativityTickClient.test.js` (asserts no `clientId` field ever appears in the outbound request body).
- No `entitledCollectionIds`, principal registry, or contract-versioning fields exist for this envelope either — same honest scope note as ADR-004's own narrow, additive envelope.

## Related Documents

- [ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md](ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md)
- [ADR-004-SIGNED-SERVICE-REQUESTS.md](ADR-004-SIGNED-SERVICE-REQUESTS.md)
- [ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md](ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md)
- [../architecture/EMAIL_INGESTION.md](../architecture/EMAIL_INGESTION.md) — §9, §18.3, §30 item 5, §31 EM8, and the EM8 Implementation Record
- [../architecture/SECURITY.md](../architecture/SECURITY.md)
