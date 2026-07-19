# ADR-007: Bounded Slack delivery retries and a terminal `delivery_failed` state, not a scheduled retry sweep

## Status
Proposed — Not Implemented. This ADR records an approved product decision and target architecture; the sweep endpoint it supersedes still exists in code (deprecated), and the bounded-retry/`delivery_failed` behavior described below has not yet been built. See [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md) item H5 for the implementation task.

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

**No exact retry count or retention duration is decided by this ADR.** A starting point of 2–3 total delivery attempts and 7–30 days of retained technical-failure metadata is a reasonable implementation default, but both remain implementation decisions to be finalized during the work tracked in [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md) item H5, not decisions this ADR makes final.

**This design does not guarantee delivery.** It explicitly favors operational simplicity and predictable, bounded failure behavior over an eventual-delivery guarantee. A very small number of Slack answers may be lost during temporary Slack or network outages; the product accepts that, on the grounds that the user can simply ask again.

## Alternatives Considered

- **Restore the originally-planned scheduled sweep** (Vercel Pro upgrade, an external scheduler such as cron-job.org, or a Railway-side scheduled job in AIKB): rejected. It reintroduces a permanent piece of scheduling infrastructure, a secret to protect (`CRON_SECRET`), and a monitoring surface, solely to chase a low-frequency failure mode — and it risks delivering an answer to Slack well after the requester has moved on, which is a worse user experience than no delayed answer at all.
- **Retry indefinitely with exponential backoff inside a background job, no terminal state**: rejected — this is a more elaborate version of the scheduled-sweep problem (still requires durable background scheduling) and still doesn't resolve the "answer arrives too late to be useful" issue; it also has no natural point at which stored customer content gets redacted.
- **Keep full question/answer/context indefinitely on any Slack event, regardless of delivery outcome**: rejected — retaining customer knowledge content indefinitely for events that will never be retried has no purpose once no future retry will read it, and needlessly increases the surface of stored customer data.
- **Never retry at all — one delivery attempt, then terminal failure**: rejected — most Slack delivery failures are transient (a momentary network blip or a rate limit), and a small number of immediate, in-flow retries with short backoff resolves the common case without any scheduling infrastructure.

## Consequences

- No Slack-delivery-related scheduled job, cron entry, or external scheduler is needed anywhere in the platform. The existing `GET /api/integrations/slack/sweep` endpoint, its `CRON_SECRET`/`cronSweepAuthService.js` bearer-secret auth, and its `vercel.json` cron intent (already removed once) become permanently unnecessary and are targeted for removal or safe disabling, not restoration. See [../architecture/CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md) and [../architecture/SECURITY.md](../architecture/SECURITY.md).
- `slack_event_log.status`, currently described as a `received → enqueued → answered → delivered/failed` state machine, gains `delivery_failed` as a distinct terminal value from the answer-generation `failed` path, so the two failure modes in this ADR remain distinguishable in the data, not just in prose. The `status` column is unconstrained `text` (no `CHECK` constraint), so this is an application-logic change, not a migration. See [../docs/supabase/global/TABLES.md](../docs/supabase/global/TABLES.md).
- Redaction after `delivery_failed` must reach both Relativity's `slack_event_log` row and any AIKB-side chat session/message content created for that idempotency key — a cross-repository concern, since AIKB owns conversation content and Relativity owns the event log. Both application repositories will need changes to implement this; this ADR authorizes the design but does not itself change either codebase.
- A resent Slack event with the same `external_event_id`, arriving after the original reached `delivery_failed`, must still be recognized as a duplicate via the existing `UNIQUE(provider, external_event_id)` constraint and acknowledged without generating a second answer — deduplication metadata surviving redaction is what makes this possible, and is why it is retained rather than deleted outright.
- Tests are required for: successful first-attempt delivery; a temporary failure followed by a successful bounded retry; all bounded attempts failing and the event reaching `delivery_failed`; question/answer/prompt/context redaction after terminal failure; deduplication metadata remaining intact and functional after redaction; a resent Slack event with the same event ID after terminal failure being rejected as a duplicate rather than reprocessed; no scheduled process later retrying a `delivery_failed` event; no delayed answer appearing in Slack after terminal failure; AIKB-generation failure successfully notifying the Slack user; and Slack-delivery failure being unable to guarantee a user-facing error message. See [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md) item H5.

## Implementation Evidence

None yet — this ADR documents an approved decision, not shipped code. The current implementation state (deprecated, unscheduled sweep endpoint; no `delivery_failed` status; no bounded in-flow retry; no redaction path) is described in [../architecture/CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md). Do not treat this ADR as evidence that bounded retries or `delivery_failed` exist in either repository today.

## Related Documents

- [../architecture/CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md) — current Slack implementation, including the deprecated sweep and the target design
- [../architecture/SYSTEM_OVERVIEW.md](../architecture/SYSTEM_OVERVIEW.md) — deployment topology and known limitations
- [../architecture/SECURITY.md](../architecture/SECURITY.md) — the sweep's cron-secret auth mechanism
- [../architecture/SERVICE_CONTRACTS.md](../architecture/SERVICE_CONTRACTS.md) — `/ask`/`/deliver` error handling and retry behavior
- [../docs/supabase/global/TABLES.md](../docs/supabase/global/TABLES.md) — `slack_event_log` schema
- [ADR-003-SLACK-EVENTS-LIVE-IN-RELATIVITY.md](ADR-003-SLACK-EVENTS-LIVE-IN-RELATIVITY.md)
- [ADR-004-SIGNED-SERVICE-REQUESTS.md](ADR-004-SIGNED-SERVICE-REQUESTS.md)
- [../roadmap/MASTER_ROADMAP.md](../roadmap/MASTER_ROADMAP.md)
- [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md) — item H5
- [../history/ARCHITECTURE_REVIEW_PHASES.md](../history/ARCHITECTURE_REVIEW_PHASES.md) — Milestone 4 deployment deviations, and the current-status table entry this ADR supersedes
