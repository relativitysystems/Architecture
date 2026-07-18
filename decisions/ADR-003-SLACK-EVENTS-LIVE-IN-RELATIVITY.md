# ADR-003: Slack Events ingestion lives in Relativity

## Status
Accepted — Implemented

## Date
Undated (decision); implemented and deployed to production 2026-07-16

## Context

At the time of the original architecture discovery, AIKB had a live, undocumented Slack Events handler (`aikb/routes/slack.js`) completely disconnected from Relativity's Slack OAuth flow:

- It used a single static global `SLACK_BOT_TOKEN`, not the per-client token Relativity's OAuth flow captured.
- It derived a fake `clientId` by SHA1-hashing the Slack channel ID — not a real tenant lookup against `clients`/`client_members`.
- It bypassed AIKB's own session/member/knowledge-gap logic, calling `searchChunks` directly.
- Relativity's captured Slack OAuth token was never consumed anywhere — `chat.postMessage` was never called from Relativity.

This was "two incompatible half-built Slack integrations in two different repositories," not a usable foundation for a real Slack Q&A feature. A decision was required on which repository should own Slack Events ingestion going forward.

## Decision

**Slack Events ingestion, signature verification, tenant mapping, and delivery all live in Relativity.** Relativity acts as a thin Slack adapter and calls AIKB's shared, normalized knowledge-query pipeline (`runKnowledgeQuery`, via `/ask`) — the same pipeline the portal uses.

AIKB's legacy handler was **retired**, not fixed in place: it now answers Slack's `url_verification` challenge (so a still-configured Slack app doesn't 404) and returns `410 Gone` for every `event_callback`, with all unsafe logic (the channel-hash tenant derivation, the global bot token, the direct `searchChunks` call) deleted from the file entirely.

## Alternatives Considered

- **Option B — keep Events ingestion in AIKB, fix the tenant mapping in place**: rejected. This was less work in the short term (signature verification already existed there) but would have perpetuated AIKB as a channel-aware, provider-specific system, directly cutting against [ADR-001](ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md)'s "Relativity owns every integration" rule and requiring every future connector to make the same one-off decision.
- **A feature flag to disable the legacy AIKB route** (rather than retiring it outright): rejected — a flag keeps the unsafe channel-hash logic present and re-enablable by an accidental environment variable flip. The code was deleted, not disabled.
- **Silent deletion (404) of the legacy route**: rejected in favor of a deliberate `410 Gone` — a plain 404 looks identical to "route never existed" in logs, giving an operator no signal if a real, still-configured Slack app is pointed at the old URL. The `410` response includes a structured warning log on every hit.

## Consequences

- Slack's OAuth, event verification, tenant mapping, and reply delivery are all one codebase's responsibility, consistent with every other integration (Google Drive, Dropbox).
- AIKB remains provider-agnostic — nothing in its request handling branches on `origin === 'slack'` beyond tagging metadata, satisfying [ADR-002](ADR-002-AIKB-OWNS-KNOWLEDGE-PROCESSING.md).
- Slack traffic cannot use `/query`'s existing member-JWT authentication (Slack has no browser session), which required a new, narrowly-scoped authentication mechanism between Relativity and AIKB for this one path — see [ADR-004](ADR-004-SIGNED-SERVICE-REQUESTS.md).
- Re-porting signature-verification logic from AIKB to Relativity was accepted as a one-time cost; the new implementation is modeled directly on AIKB's already-correct `crypto.timingSafeEqual` comparison pattern.
- Any future Teams/Gmail/Outlook event ingestion follows the same shape by default, with no repeat of this decision required.

## Implementation Evidence

- Relativity: `POST /api/integrations/slack/events` (`routes/integrations/slack.js`), `services/slackSignatureService.js` (HMAC verification), `services/slackEventsService.js` (orchestration), `slack_event_log` table (migration `20260716_slack_event_log.sql`) for deduplication.
- AIKB: `aikb/routes/slack.js` returns `410 Gone` for every `event_callback` and contains no channel-hash or global-bot-token logic (verified by AIKB's retired-endpoint regression suite, passing unmodified).
- Verified end-to-end against a real, connected Slack workspace in production on 2026-07-16 (Slack mention → Relativity → AIKB → Relativity → threaded Slack reply).
- Full detail: [../history/ARCHITECTURE_REVIEW_PHASES.md](../history/ARCHITECTURE_REVIEW_PHASES.md) — Phase 4, Milestones 1 and 4.

## Related Documents

- [../architecture/CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md)
- [../architecture/SERVICE_CONTRACTS.md](../architecture/SERVICE_CONTRACTS.md)
- [ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md](ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md)
- [ADR-004-SIGNED-SERVICE-REQUESTS.md](ADR-004-SIGNED-SERVICE-REQUESTS.md)
- [../history/ARCHITECTURE_REVIEW_PHASES.md](../history/ARCHITECTURE_REVIEW_PHASES.md)
