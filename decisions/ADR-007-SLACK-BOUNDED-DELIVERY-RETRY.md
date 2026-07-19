# ADR-007: Bounded Slack delivery retries and a terminal `delivery_failed` state, not a scheduled retry sweep

## Status
**Accepted — Implemented.** The design below is shipped in both `relativitysystems/Relativity` and `relativitysystems/aikb`, verified against the current code on both `main` branches. The sweep endpoint this ADR supersedes has been removed outright (not merely disabled). See [Implementation Status](#implementation-status) below for the verified, file-referenced detail, and [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md) item H5, now marked completed.

## Date
2026-07-18

## Context

Milestone 4 shipped a Slack delivery-retry sweep (`GET /api/integrations/slack/sweep`) intended to recover `slack_event_log` rows stuck in `received`/`enqueued` past a timeout, on a Vercel Cron schedule. That schedule was rejected by the project's Vercel Hobby plan tier at deploy time (any sub-daily cron entry fails the deploy outright) and was removed rather than downgraded, leaving the sweep code live but unscheduled — see [../history/ARCHITECTURE_REVIEW_PHASES.md](../history/ARCHITECTURE_REVIEW_PHASES.md) (Milestone 4, "Production deployment — deviations from plan") for the full account of what happened and why. Subsequent documentation tracked "restore the sweep's scheduler" (a Vercel Pro upgrade, an external scheduler, or a Railway-side scheduled job in AIKB) as the highest-priority open item, on the grounds that it closed a regression from Milestone 4's own definition of done.

That restoration plan is no longer the plan. On reflection, operating a scheduler solely to retry a small number of stuck Slack deliveries adds ongoing operational surface (an extra scheduled job, an extra secret, an extra failure mode to monitor) for a low-frequency failure that a simpler design can absorb without it. A scheduled sweep also has an undesirable product property independent of its operational cost: a delayed answer can land in a Slack thread well after the requester has moved on to something else, which reads as confusing or stale rather than helpful. Finally, the sweep as designed retains the full question, generated answer, and retrieved context indefinitely while a row is stuck — customer knowledge content with no clear retention story, kept only because a scheduler might eventually need it.

## Decision

**Relativity will not restore the Slack delivery-retry sweep's scheduler.** In its place, Slack delivery gets a small, bounded number of immediate retry attempts inside the same processing flow that generated the answer, followed by a terminal state if delivery still doesn't succeed. No background process, cron job, or scheduled sweep of any kind revisits a Slack event after that flow completes.

**AIKB generation failure vs. Slack delivery failure are handled differently, and documentation must keep them distinct:**

- **AIKB generation failure** — AIKB cannot produce an answer, but the Slack API is still reachable. Relativity sends a brief error response to the Slack user, marks the processing attempt failed, and terminates. Because this failure mode doesn't involve Slack being unreachable, an error message can reliably be sent.
- **Slack delivery failure** — AIKB generated an answer, but delivering it to Slack (the `/deliver` callback and/or the `chat.postMessage` call) fails. The handler retries delivery a small, bounded number of times with short backoff, still within the current processing pass. If every attempt fails, the event is marked with a terminal status, `delivery_failed`, and is never processed again. **An error message cannot always be sent to the user in this case**, because sending it depends on the same Slack API that just failed to accept the answer — this is a structural limitation of the failure mode, not an implementation gap to close.

**On reaching `delivery_failed`:**

- The Slack user's question, the generated answer, retrieved document chunks/context, and any full prompts associated with the event are redacted, nulled, or removed. This spans both Relativity's `slack_event_log.question` column and any corresponding AIKB-side chat session/message content tied to the same idempotency key.
- Only technical metadata needed for Slack event deduplication, basic debugging, delivery-attempt auditing, and client identification is retained. Conceptually: the Slack event ID (`external_event_id`), `client_id`, `status = 'delivery_failed'`, `attempt_count`, a sanitized error code and message, `failed_at`, and any other timestamps needed for auditing or cleanup.
- The user can ask the question again manually if they still need an answer.

**No exact retry count or retention duration is decided by this ADR.** A starting point of 2–3 total delivery attempts and 7–30 days of retained technical-failure metadata is a reasonable implementation default, but both remain implementation decisions to be finalized during the work tracked in [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md) item H5, not decisions this ADR makes final. **Resolved during implementation:** 3 total attempts (initial + 2 retries) with 2s/5s backoff, both configurable via environment variables — see [Implementation Status](#implementation-status). Retention duration for terminal-state technical metadata remains unresolved — see that section's Known Gap.

**This design does not guarantee delivery.** It explicitly favors operational simplicity and predictable, bounded failure behavior over an eventual-delivery guarantee. A very small number of Slack answers may be lost during temporary Slack or network outages; the product accepts that, on the grounds that the user can simply ask again.

## Alternatives Considered

- **Restore the originally-planned scheduled sweep** (Vercel Pro upgrade, an external scheduler such as cron-job.org, or a Railway-side scheduled job in AIKB): rejected. It reintroduces a permanent piece of scheduling infrastructure, a secret to protect (`CRON_SECRET`), and a monitoring surface, solely to chase a low-frequency failure mode — and it risks delivering an answer to Slack well after the requester has moved on, which is a worse user experience than no delayed answer at all.
- **Retry indefinitely with exponential backoff inside a background job, no terminal state**: rejected — this is a more elaborate version of the scheduled-sweep problem (still requires durable background scheduling) and still doesn't resolve the "answer arrives too late to be useful" issue; it also has no natural point at which stored customer content gets redacted.
- **Keep full question/answer/context indefinitely on any Slack event, regardless of delivery outcome**: rejected — retaining customer knowledge content indefinitely for events that will never be retried has no purpose once no future retry will read it, and needlessly increases the surface of stored customer data.
- **Never retry at all — one delivery attempt, then terminal failure**: rejected — most Slack delivery failures are transient (a momentary network blip or a rate limit), and a small number of immediate, in-flow retries with short backoff resolves the common case without any scheduling infrastructure.

## Consequences

- No Slack-delivery-related scheduled job, cron entry, or external scheduler is needed anywhere in the platform. The `GET /api/integrations/slack/sweep` endpoint, its `CRON_SECRET`/`cronSweepAuthService.js` bearer-secret auth, and its `vercel.json` cron intent (already removed once) have been **removed outright** — see [Implementation Status](#implementation-status) below, [../architecture/CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md), and [../architecture/SECURITY.md](../architecture/SECURITY.md).
- `slack_event_log.status`, currently described as a `received → enqueued → answered → delivered/failed` state machine, gains `delivery_failed` as a distinct terminal value from the answer-generation `failed` path, so the two failure modes in this ADR remain distinguishable in the data, not just in prose. The `status` column is unconstrained `text` (no `CHECK` constraint), so this is an application-logic change, not a migration. See [../docs/supabase/global/TABLES.md](../docs/supabase/global/TABLES.md).
- Redaction after `delivery_failed` must reach both Relativity's `slack_event_log` row and any AIKB-side chat session/message content created for that idempotency key — a cross-repository concern, since AIKB owns conversation content and Relativity owns the event log. Both application repositories will need changes to implement this; this ADR authorizes the design but does not itself change either codebase.
- A resent Slack event with the same `external_event_id`, arriving after the original reached `delivery_failed`, must still be recognized as a duplicate via the existing `UNIQUE(provider, external_event_id)` constraint and acknowledged without generating a second answer — deduplication metadata surviving redaction is what makes this possible, and is why it is retained rather than deleted outright.
- Tests are required for: successful first-attempt delivery; a temporary failure followed by a successful bounded retry; all bounded attempts failing and the event reaching `delivery_failed`; question/answer/prompt/context redaction after terminal failure; deduplication metadata remaining intact and functional after redaction; a resent Slack event with the same event ID after terminal failure being rejected as a duplicate rather than reprocessed; no scheduled process later retrying a `delivery_failed` event; no delayed answer appearing in Slack after terminal failure; AIKB-generation failure successfully notifying the Slack user; and Slack-delivery failure being unable to guarantee a user-facing error message. See [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md) item H5.

## Implementation Status

Verified directly against `relativitysystems/Relativity` (`main`, commit `fd62758`) and `relativitysystems/aikb` (`main`, commit `b0a0420`). Every claim below is sourced from a specific file; nothing here is inferred from this ADR's own prior design intent.

### Sweep removal — done, not merely disabled

- `GET /api/integrations/slack/sweep` no longer exists. The route was deleted from `Relativity/routes/integrations/slack.js`, not left mounted-but-gated.
- `Relativity/services/cronSweepAuthService.js` (the `CRON_SECRET` bearer-auth gate) was deleted outright, along with its two dedicated test files.
- `Relativity/services/slackEventsService.js`'s `runDeliverySweep`/`retryStuckRow`/`bestEffortFailureReply` functions and `Relativity/services/slackEventLogService.js`'s `listStuckForRetry`/`incrementAttempt` helpers — the sweep's only callers — were deleted as dead code once the route was gone.
- No `vercel.json` cron entry exists (none had existed since the Milestone 4 deviation this ADR responds to). `CRON_SECRET` was removed from `Relativity/.env.example`.

### Bounded, immediate, in-flow retries — implemented, and extended slightly beyond this ADR's literal scope

- `Relativity/services/retryWithBackoff.js` is a small, generic, injectable-sleep retry helper (attempt count and per-attempt backoff both configurable, defaulting to real `setTimeout`-based waits in production and a no-op sleep in tests).
- Default configuration (`Relativity/config/index.js`): **3 total attempts** (initial + 2 retries) via `SLACK_DELIVERY_MAX_ATTEMPTS` (default `3`), backoff `[2000, 5000]` ms via `SLACK_DELIVERY_BACKOFF_MS` (default `"2000,5000"`) — i.e. attempt 1 → wait 2s → attempt 2 → wait 5s → attempt 3 → terminal. Both are environment-configurable, resolving the "no exact retry count" point this ADR originally left open.
- Applied in `Relativity/services/slackDeliverService.js` to the real-answer delivery leg of `POST /api/integrations/slack/deliver` (the `chat.postMessage` call this ADR's Decision section names directly).
- **Implementation decision beyond this ADR's literal text, made during H5 and recorded here for traceability:** the same bounded-retry-then-`delivery_failed` treatment was also applied in `Relativity/services/slackEventsService.js` to (a) the initial synchronous accept-and-enqueue call to AIKB's `POST /api/knowledge/ask`, and (b) the empty-question direct Slack reply. Rationale: once the sweep is removed, a row stuck at `received` after an `/ask` failure would otherwise have **no** recovery path and would never reach a terminal state or get its `question` redacted — a regression relative to this ADR's own redaction guarantee. Both paths reuse `retryWithBackoff` and the same terminal-state/redaction logic described below. Where Slack itself is still reachable at the point of exhaustion (both of these cases — only AIKB or the reply itself failed, not the Slack API), a best-effort single-attempt "couldn't complete that request" notice is sent first, mirroring the AIKB-generation-failure UX, before the row is marked terminal.

### Terminal `delivery_failed` and redaction — implemented

- `Relativity/services/slackEventLogService.js` adds `STATUS.DELIVERY_FAILED = 'delivery_failed'`, distinct from the pre-existing `STATUS.FAILED`, exactly as this ADR's Consequences section specified — an application-logic change, no migration (`status` remains unconstrained `text`).
- `markDeliveryFailed(id, { errorCode, attemptCount })` sets `status`, `failed_at`, `error_code`, `attempt_count`, **and `question = null` in the same UPDATE** — a row can never be observed in the `delivery_failed` state with its question still attached.
- `Relativity/services/slackDeliveryFailureService.js` is the shared orchestration point: it calls `markDeliveryFailed` and then best-effort calls the AIKB redaction callback (below). Used identically by `slackDeliverService.js` and `slackEventsService.js` so the "reaching `delivery_failed` always redacts" guarantee is implemented exactly once, not duplicated per call site.
- **AIKB generation failure is kept distinct, per this ADR's explicit instruction**: `slackDeliverService.js` branches on `payload.error === true` and, for that branch, retries are never attempted, the status on failure remains the pre-existing generic `'failed'` (never `delivery_failed`), and no redaction is triggered — there is no answer to redact, and this ADR's own text describes this failure mode as one where "an error message can reliably be sent," not one requiring the same bounded-retry/redaction treatment.

### AIKB-side redaction — implemented as a best-effort callback, with the limitation this ADR anticipated

- `relativitysystems/aikb` adds `POST /api/knowledge/chat/redact` (`aikb/routes/knowledge.js`), authenticated identically to `POST /ask` (`requireServiceRequest` additive to the router-level `x-api-key`) — `clientId`/`idempotencyKey` come only from the verified signed envelope, never the request body.
- `aikb/services/supabaseService.js#redactChatSessionByIdempotencyKey` nulls the session `title` and replaces every message's `content` with a fixed redaction marker (the column is `NOT NULL`, so it cannot be nulled outright) and nulls `sources`/`metadata` — the latter is where the question text, retrieval query, and retrieved-chunk/document references were stored, so this covers "prompts and retrieved context" as this ADR requires. Session/message rows, ids, timestamps, `client_id`/`session_id` linkage, `origin`/`origin_metadata`, and `idempotency_key` are all left untouched, matching this ADR's "technical metadata" retention list. The operation is idempotent — safe to call more than once.
- `Relativity/services/aikbRedactClient.js` signs and sends this callback from `slackDeliveryFailureService.js`, immediately after `markDeliveryFailed` succeeds.
- **This callback is best-effort and single-attempt, not bounded-retried like Slack delivery itself.** If it fails (AIKB unreachable, timeout, etc.), the failure is caught and logged (`console.error`) but never re-thrown, never retried, and never surfaced to the Slack-facing response. **This is a real, currently-accepted limitation, not an oversight this ADR hides**: on that failure path, Relativity's own `slack_event_log.question` is still correctly redacted (that happens first, unconditionally, in the same operation that sets `delivery_failed`), but the AIKB-side chat session/message content for that idempotency key can remain un-redacted until a future, not-yet-built retry or cleanup mechanism exists. There is currently no scheduled process of any kind — consistent with this ADR's core decision not to run one — so nothing will automatically retry a failed redaction callback today.

### Idempotency and deduplication — implemented and verified under the specific scenario this ADR calls out

- Relativity's `UNIQUE(provider, external_event_id)` constraint on `slack_event_log` is untouched by any of the above — redaction only nulls content columns, never the row or its dedup key.
- Verified by test (`Relativity/test/slackEventsService.test.js`): a Slack resend of the same `event_id`, arriving **after** the original event already reached `delivery_failed`, is deduped at `insertReceived` and never reaches AIKB again.
- Verified by test (`Relativity/test/slackDeliverService.test.js`): a `/deliver` callback arriving after the row already reached `delivery_failed` is a safe no-op (the row is no longer in the `enqueued` state `claimForDelivery` requires) — no second Slack post ever happens.

### Automated test coverage

- `relativitysystems/Relativity`: **266/266** passing, including new/updated files `test/retryWithBackoff.test.js`, `test/aikbRedactClient.test.js`, `test/slackDeliveryFailureService.test.js`, and rewritten coverage in `test/slackDeliverService.test.js` and `test/slackEventsService.test.js` for: first-attempt success, retry-then-success, all-attempts-exhausted → `delivery_failed`, question redaction, the AIKB redact-client call (and its failure being swallowed), the two dedup/idempotency-after-`delivery_failed` scenarios above, and the AIKB-generation-failure path being unchanged (single attempt, no redaction).
- `relativitysystems/aikb`: **47/47** passing, including new files `test/knowledgeRedactRoute.test.js` (auth gating, matching the existing `/ask` route's own test convention) and `test/redactChatSession.test.js` (the actual redaction logic — question/answer/sources/metadata redacted for the matching session, an unrelated session/client untouched, idempotent on repeat calls, safe no-op for an idempotency key with no session).

### Known gaps against this ADR (documented, not hidden)

- **No guaranteed delivery, as this ADR always intended** — this is the accepted design, not a bug: a small number of Slack answers can still be lost during a genuine, sustained Slack outage that outlasts 3 attempts over ~7 seconds.
- **AIKB-side redaction can silently fail** (see above) — Relativity's own state and question redaction is unconditional and verified; AIKB's is best-effort only.
- **Technical-metadata retention duration is unresolved.** `Relativity/services/slackEventLogService.js` carries an explicit `TODO(ADR-007 metadata retention)` — this ADR's suggested 7–30 days has not been implemented as a cleanup mechanism in either repository. See [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md).
- **No monitoring/alerting exists yet** for a sustained rise in `delivery_failed` events or for AIKB redaction-callback failures specifically — both are currently only visible via application logs. Tracked as a follow-up, not blocking, since ADR-007 never required a monitoring solution to ship alongside the core behavior.

## Related Documents

- [../architecture/CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md) — current Slack implementation detail and the verification checklist for this design
- [../architecture/SYSTEM_OVERVIEW.md](../architecture/SYSTEM_OVERVIEW.md) — deployment topology and current limitations
- [../architecture/SECURITY.md](../architecture/SECURITY.md) — confirms the sweep's cron-secret auth mechanism is gone, not merely deprecated
- [../architecture/SERVICE_CONTRACTS.md](../architecture/SERVICE_CONTRACTS.md) — `/ask`/`/deliver`/`/chat/redact` contracts and error handling
- [../docs/supabase/global/TABLES.md](../docs/supabase/global/TABLES.md) — `slack_event_log` schema, including the now-implemented `delivery_failed` value
- [ADR-003-SLACK-EVENTS-LIVE-IN-RELATIVITY.md](ADR-003-SLACK-EVENTS-LIVE-IN-RELATIVITY.md)
- [ADR-004-SIGNED-SERVICE-REQUESTS.md](ADR-004-SIGNED-SERVICE-REQUESTS.md)
- [../roadmap/MASTER_ROADMAP.md](../roadmap/MASTER_ROADMAP.md)
- [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md) — item H5, now completed, plus its operational follow-ups
- [../history/ARCHITECTURE_REVIEW_PHASES.md](../history/ARCHITECTURE_REVIEW_PHASES.md) — Milestone 4 deployment deviations, and the current-status table entry this ADR supersedes
