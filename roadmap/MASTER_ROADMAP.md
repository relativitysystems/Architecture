# Master Roadmap

Source repositories: `relativitysystems/Relativity` and `relativitysystems/AIKB`. This is the high-level architecture and product sequence — it summarizes where the platform has been and where it's headed next, and links out to [FEATURE_BACKLOG.md](FEATURE_BACKLOG.md) and [CONNECTOR_ROADMAP.md](CONNECTOR_ROADMAP.md) for item-level detail rather than duplicating it here.

## Purpose

A new contributor should be able to read this document and answer: what foundation is already built, what phase is the platform in right now, and what should be worked on next. Detailed technical debt lives in the backlog; connector-by-connector sequencing lives in the connector roadmap; this document is the summary that ties them together.

## Current Platform Foundation

The platform's core split is implemented and stable: Relativity owns identity, tenancy, and every external integration; AIKB owns ingestion, retrieval, reasoning, conversations, and knowledge gaps. See [../architecture/SYSTEM_OVERVIEW.md](../architecture/SYSTEM_OVERVIEW.md) and [../decisions/ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md](../decisions/ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md)/[ADR-002](../decisions/ADR-002-AIKB-OWNS-KNOWLEDGE-PROCESSING.md).

What's built:
- Multi-tenant document ingestion, chunking, embedding, and vector retrieval (AIKB).
- A working RAG chat pipeline with intent classification, query rewriting, citation generation, and heuristic knowledge-gap detection (AIKB).
- Knowledge collections with SQL-level, fail-closed retrieval enforcement (AIKB).
- A full client portal: authentication, document upload, chat, chat history, collections management, team management (Relativity).
- Three external connectors at different maturity levels: Slack (fully modernized reference implementation), Google Drive (working but split across two auth paths), Dropbox (connect/status only, no working import path). See [CONNECTOR_ROADMAP.md](CONNECTOR_ROADMAP.md).

## Completed Architecture Phases

1. **Phase 1 — Architecture discovery.** Mapped both repositories, found the platform boundary mostly sound but violated by a disconnected, unsafe Slack Events handler in AIKB. See [../history/ARCHITECTURE_REVIEW_PHASES.md](../history/ARCHITECTURE_REVIEW_PHASES.md).
2. **Phase 2/3 — Platform architecture and domain model (design specifications).** Proposed a larger target architecture (Knowledge Surface abstraction, full signed `ServiceRequest` platform, split Collection ownership). Only part of this was built — see the phase summaries in the history document for what carried through versus what remains proposed.
3. **Phase 4 Milestones 1–4 — Slack MVP, shipped to production.** Legacy AIKB Slack handler retired; encrypted OAuth credential storage built; a real Slack OAuth connection flow shipped; `@RelativityBot` mentions answered end-to-end via AIKB's shared knowledge pipeline, verified live against a real Slack workspace on 2026-07-16. See [../history/ARCHITECTURE_REVIEW_PHASES.md](../history/ARCHITECTURE_REVIEW_PHASES.md) for full milestone detail.
4. **Knowledge collections**, implemented in AIKB beyond the original Milestone 5 scope — a full CRUD model with fail-closed Slack enforcement, not just the originally-planned hardcoded stand-in. See [../decisions/ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md](../decisions/ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md).

## Current Phase

The platform is between Milestone 4 (shipped) and Milestones 5–7 (not started). Milestone 4's production rollout left the Slack delivery-retry sweep unscheduled (a Vercel plan-tier limitation); rather than restoring that scheduler, the product decision is to **not** run a scheduled sweep at all — see [ADR-007](../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md). The highest-priority open item is implementing bounded, immediate Slack delivery retries with a terminal `delivery_failed` state in their place. See [FEATURE_BACKLOG.md](FEATURE_BACKLOG.md).

## Immediate Next Priorities

In rough dependency/impact order:

1. **Implement bounded Slack delivery retries and a terminal `delivery_failed` state**, replacing the never-restored sweep scheduler — a small number of immediate, in-flow retry attempts on Slack delivery failure, then a terminal status with the question/answer/context redacted and only dedup/audit metadata retained. No recurring sweep or cron scheduler is planned. See [ADR-007](../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md) and [FEATURE_BACKLOG.md](FEATURE_BACKLOG.md).
2. **Close the shared-`x-api-key`-only tenant-authorization gap** on AIKB's non-Slack management routes (`/ingest`, `/reindex`, `/documents/:clientId`, etc.) — the longest-standing open Critical/High finding from the original review. See [../decisions/ADR-004-SIGNED-SERVICE-REQUESTS.md](../decisions/ADR-004-SIGNED-SERVICE-REQUESTS.md).
3. **Migrate Google Drive and Dropbox onto the encrypted `oauth_connections`/`oauth_credentials` model**, retiring the legacy plaintext `oauth_tokens` table entirely.
4. **Decide and implement Milestone 6** (fuller knowledge-gap/conversation-metadata polish — system-vs-user `reportedBy`, richer origin metadata) — schema-ready, code not started.
5. **Housekeeping**: delete AIKB's now-fully-inert legacy `routes/slack.js` file and its unused `SLACK_BOT_TOKEN`/`SLACK_SIGNING_SECRET` config reads.

Full item-level detail, grouped by priority and area: [FEATURE_BACKLOG.md](FEATURE_BACKLOG.md).

## Dependencies

- Migrating Google Drive/Dropbox onto encrypted credentials has no dependency on any other item and can proceed independently.
- Bounded Slack delivery retries and the `delivery_failed` terminal state are independent of every other item — it's a self-contained change to the Slack delivery path, not a code change dependent on infrastructure decisions (no scheduler/cron infrastructure is needed at all under this design).
- Any future connector (Teams, Gmail, Outlook) should wait until the shared-`x-api-key` gap (item 2 above) is closed or explicitly accepted as a known risk for that connector too, since each new connector otherwise inherits the same weak service-to-service trust model Slack's `/ask` path deliberately avoided. See [../decisions/ADR-004-SIGNED-SERVICE-REQUESTS.md](../decisions/ADR-004-SIGNED-SERVICE-REQUESTS.md).
- Milestone 7 (direct messages, employee-level authorization) depends on the full Identity Link / Guest / Principal model from the original domain-model proposal, which does not exist today — this is a substantial build, not an incremental one.

## Later Product Expansion

Not scheduled, but a plausible next step given the current architecture (see [CONNECTOR_ROADMAP.md](CONNECTOR_ROADMAP.md) for connector-specific sequencing and [../product/AI_AGENTS.md](../product/AI_AGENTS.md) for the RAG-to-agentic roadmap):

- Additional connectors: Microsoft Teams, Gmail, Outlook, meeting-transcript sources, CRM systems — all following the pattern Slack established ([../architecture/CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md)).
- Extending collection-based retrieval filtering to the portal's own chat (currently Slack-only).
- Client-facing analytics (question volume, document counts, gap trends) — the backend already computes most of the needed data; only rendering is missing. See [../product/KNOWLEDGE_ANALYTICS.md](../product/KNOWLEDGE_ANALYTICS.md).
- An admin review workflow for knowledge gaps, using the already-defined `status` lifecycle. See [../product/KNOWLEDGE_GAP_DETECTION.md](../product/KNOWLEDGE_GAP_DETECTION.md).
- Tool/function calling and multi-step retrieval loops, moving from single-shot RAG toward more agentic behavior — no work started, no schema exists. See [../product/AI_AGENTS.md](../product/AI_AGENTS.md).

## Related Documents

- [FEATURE_BACKLOG.md](FEATURE_BACKLOG.md) — item-level technical backlog
- [CONNECTOR_ROADMAP.md](CONNECTOR_ROADMAP.md) — connector-by-connector sequencing
- [../architecture/SYSTEM_OVERVIEW.md](../architecture/SYSTEM_OVERVIEW.md)
- [../decisions/](../decisions/)
- [../history/ARCHITECTURE_REVIEW_PHASES.md](../history/ARCHITECTURE_REVIEW_PHASES.md)
