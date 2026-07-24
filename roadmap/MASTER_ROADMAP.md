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
- Three external connectors at different maturity levels: Slack (fully modernized reference implementation, including bounded delivery-failure handling), Google Drive (working but split across two auth paths), Dropbox (connect/status only, no working import path). See [CONNECTOR_ROADMAP.md](CONNECTOR_ROADMAP.md).
- Bounded, immediate Slack delivery retries with a terminal `delivery_failed` state and cross-repository redaction — no scheduled sweep of any kind. See [ADR-007](../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md).

## Completed Architecture Phases

1. **Phase 1 — Architecture discovery.** Mapped both repositories, found the platform boundary mostly sound but violated by a disconnected, unsafe Slack Events handler in AIKB. See [../history/ARCHITECTURE_REVIEW_PHASES.md](../history/ARCHITECTURE_REVIEW_PHASES.md).
2. **Phase 2/3 — Platform architecture and domain model (design specifications).** Proposed a larger target architecture (Knowledge Surface abstraction, full signed `ServiceRequest` platform, split Collection ownership). Only part of this was built — see the phase summaries in the history document for what carried through versus what remains proposed.
3. **Phase 4 Milestones 1–4 — Slack MVP, shipped to production.** Legacy AIKB Slack handler retired; encrypted OAuth credential storage built; a real Slack OAuth connection flow shipped; `@RelativityBot` mentions answered end-to-end via AIKB's shared knowledge pipeline, verified live against a real Slack workspace on 2026-07-16. See [../history/ARCHITECTURE_REVIEW_PHASES.md](../history/ARCHITECTURE_REVIEW_PHASES.md) for full milestone detail.
4. **Knowledge collections**, implemented in AIKB beyond the original Milestone 5 scope — a full CRUD model with fail-closed Slack enforcement, not just the originally-planned hardcoded stand-in. See [../decisions/ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md](../decisions/ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md).
5. **Slack bounded delivery retries and terminal `delivery_failed` state (ADR-007, item H5), shipped to both repositories.** The never-restored delivery-retry sweep and its `CRON_SECRET` auth have been removed outright (not merely disabled). In their place: up to 3 total delivery attempts with short backoff, a terminal `delivery_failed` status on exhaustion, Relativity-side question redaction, and a best-effort cross-repository callback that redacts the corresponding AIKB chat session/message content. Distinct handling for AIKB-generation failure is preserved unchanged. 266/266 Relativity tests and 47/47 AIKB tests passing. See [ADR-007](../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md)'s Implementation Status section for full file-referenced detail.

## Current Phase

Slack delivery reliability and Knowledge Collections are both implemented and test-covered. **The platform is now sufficiently developed to demonstrate to prospects and customers**, and the current phase is split across two parallel tracks that should run concurrently, not sequentially — see [Two Parallel Tracks](#two-parallel-tracks) below. This is a deliberate shift: the platform should not continue accumulating engineering milestones indefinitely before any customer validation happens. See [FEATURE_BACKLOG.md](FEATURE_BACKLOG.md) for full item-level technical detail.

## Two Parallel Tracks

### Track A — Product Reliability

- ✅ Slack bounded delivery retries and `delivery_failed` — **completed** (see above).
- ✅ Knowledge Collections — **completed** (Phase 4 above).
- [ ] Manual verification of both in a staging/production-like environment before relying on them for prospect-facing demos (automated tests pass; a live Slack-workspace/portal walkthrough has not yet been separately confirmed post-implementation). See [../architecture/CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md)'s verification checklist.
- [ ] Remaining operational follow-ups that don't block a demo but should not be forgotten: technical-metadata retention cleanup, monitoring/alerting for `delivery_failed`/redaction-callback failures. See [FEATURE_BACKLOG.md](FEATURE_BACKLOG.md).
- [ ] Security/reliability items unrelated to Slack (shared-`x-api-key` gap, encrypted-credential migration for Drive/Dropbox) — continue in parallel, not blocking the track below.

### Track B — Company Validation

- [ ] **Demo video and sales-ready demo environment** — see the dedicated section below. This is a major near-term priority, not a minor marketing task.
- [ ] Prospect outreach and founder-led sales conversations using the demo video.
- [ ] Customer feedback collection from those conversations.
- [ ] Use that feedback to inform the next major product milestone — **before** committing to another large connector or infrastructure build.

Track A's staging verification is a prerequisite for a credible Track B demo (item 1 in the sequence below); everything else in the two tracks can proceed concurrently.

## Immediate Next Priorities

In sequence:

1. **Verify the completed Slack delivery-failure handling and Knowledge Collections implementation** in staging/production-like conditions — run both automated suites, then walk through the manual checklist in [../architecture/CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md). This is the prerequisite for a credible demo, not new engineering work.
2. **Finalize the demo environment and demo data** — a stable demo client/account, representative documents, General and Slack knowledge collections configured. See the demo-video milestone below for the full checklist.
3. **Prepare and record the Relativity Systems demo video** — see [Demo Video and Sales-Ready Demo Environment](#demo-video-and-sales-ready-demo-environment) below and [FEATURE_BACKLOG.md](FEATURE_BACKLOG.md)'s corresponding item.
4. **Use the demo video for early prospect outreach and customer discovery** — founder-led sales conversations, website placement, direct outreach material.
5. **Use feedback from those conversations to inform the next major product milestone** — do not pre-commit to the next connector before this feedback exists.
6. **Then proceed with the next major connector or Knowledge Sync work** (see [Later Product Expansion](#later-product-expansion)), and continue the remaining Track A engineering items below in parallel throughout:
   - ✅ **Completed (backlog H4).** The shared-`x-api-key`-only tenant-authorization gap on AIKB's non-Slack management routes is closed for all 14 routes (`/ingest`, `/reindex`, document/client delete, documents/collections listing, collections CRUD, and the `/jobs`/`/summary`/`/analytics`/`/stats` reporting routes) via the same signed HMAC envelope `/ask` already used. See [../decisions/ADR-004-SIGNED-SERVICE-REQUESTS.md](../decisions/ADR-004-SIGNED-SERVICE-REQUESTS.md).
   - ✅ **Completed (backlog H2).** Google Drive and Dropbox migrated onto the encrypted `oauth_connections`/`oauth_credentials` model. A direct query confirmed `oauth_tokens` held zero rows for any provider before this change, so it was a code-path migration with no live credentials at risk, not a data migration. `oauth_tokens` itself has not been dropped (kept as a defensive landing spot).
   - Decide and implement Milestone 6 (fuller knowledge-gap/conversation-metadata polish) — schema-ready, code not started.
   - ✅ Housekeeping (L7): the unused `SLACK_BOT_TOKEN` config read has been removed from `aikb/config/index.js` and `.env.example`. `routes/slack.js` itself was **not** deleted — it still performs live Slack signature verification and answers the `url_verification` handshake, and `SLACK_SIGNING_SECRET` remains required for that. Address the Slack delivery-failure operational follow-ups tracked in [FEATURE_BACKLOG.md](FEATURE_BACKLOG.md).

Full item-level detail, grouped by priority and area: [FEATURE_BACKLOG.md](FEATURE_BACKLOG.md).

## Demo Video and Sales-Ready Demo Environment

The demo video is a **major company milestone, not a minor marketing task**. It enables clear communication of the product, prospect outreach, founder-led sales, customer feedback, validation of the value proposition, website and outreach material, and early client acquisition. The current platform — document ingestion, business knowledge retrieval with source-backed answers, the team/client portal, knowledge collections, Slack integration (now including bounded delivery-failure handling), secure multi-tenant behavior — is sufficiently developed to demonstrate today. It should not wait until every future connector or integration is complete.

This section tracks status, acceptance criteria, and dependencies only. The full narrative strategy — demo objective, fictional company storyline, feature-by-feature script framework, financial-value guidance, video structure, script/account-prep checklists, and trust/honesty requirements — lives in the dedicated document: [../go-to-market/DEMO_VIDEO_STRATEGY.md](../go-to-market/DEMO_VIDEO_STRATEGY.md).

**Acceptance criteria:**
- A stable demo client/account exists.
- Demo login and access flow are confirmed working.
- Representative documents are loaded.
- General and Slack knowledge collections are both demonstrated.
- At least one successful portal query is shown.
- At least one successful Slack question is shown.
- Source citations are visible in at least one answer.
- The demo explains the customer pain point.
- The demo explains the value proposition.
- The demo shows the current product honestly — without overstating future/unbuilt features.
- The demo ends with a clear next step for prospects.
- No private production customer data appears anywhere in the recording.
- No API keys, tokens, internal IDs, logs, developer tools, or other secrets appear on screen.
- The final video is suitable for the website, direct outreach, and live sales conversations.

**Dependency checklist:**
- [ ] Slack bounded-delivery implementation verified in a staging/production-like environment (Track A, item 1 above).
- [ ] Knowledge Collections verified in the same environment.
- [ ] Clean, stable, realistic demo data prepared (see [FEATURE_BACKLOG.md](FEATURE_BACKLOG.md) for the detailed task list).
- [ ] Polished demo account (no placeholder/test-looking content, no broken UI states) — see [../go-to-market/DEMO_VIDEO_STRATEGY.md](../go-to-market/DEMO_VIDEO_STRATEGY.md)'s account preparation checklist.
- [x] Founder strategy / business positioning for the narration — finalized in [../go-to-market/DEMO_VIDEO_STRATEGY.md](../go-to-market/DEMO_VIDEO_STRATEGY.md).
- [ ] Full narration and screen-action script written and reviewed, following [../go-to-market/DEMO_VIDEO_STRATEGY.md](../go-to-market/DEMO_VIDEO_STRATEGY.md)'s script requirements.
- [ ] Recording environment ready (screen capture, audio, browser/window setup free of unrelated tabs/notifications).
- [ ] Final quality review before publishing (content accuracy, no leaked secrets/customer data, audio/video quality).

No completion date is set here; none exists elsewhere in this roadmap to anchor it to. See [FEATURE_BACKLOG.md](FEATURE_BACKLOG.md) for the corresponding work-breakdown backlog item.

## Dependencies

- Migrating Google Drive/Dropbox onto encrypted credentials has no dependency on any other item and can proceed independently.
- The demo video's only hard dependency is Track A's staging verification step (item 1 above) — it does not depend on, and should not wait for, the shared-`x-api-key` gap, the Drive/Dropbox migration, Milestone 6, or any future connector.
- Any future connector (Teams, Gmail, Outlook) should wait until the shared-`x-api-key` gap is closed or explicitly accepted as a known risk for that connector too, since each new connector otherwise inherits the same weak service-to-service trust model Slack's `/ask` path deliberately avoided. See [../decisions/ADR-004-SIGNED-SERVICE-REQUESTS.md](../decisions/ADR-004-SIGNED-SERVICE-REQUESTS.md).
- Milestone 7 (direct messages, employee-level authorization) depends on the full Identity Link / Guest / Principal model from the original domain-model proposal, which does not exist today — this is a substantial build, not an incremental one.

## Later Product Expansion

Not scheduled, but a plausible next step given the current architecture (see [CONNECTOR_ROADMAP.md](CONNECTOR_ROADMAP.md) for connector-specific sequencing and [../product/AI_AGENTS.md](../product/AI_AGENTS.md) for the RAG-to-agentic roadmap):

- Additional connectors: Microsoft Teams, Gmail, Outlook, meeting-transcript sources, CRM systems — all following the pattern Slack established ([../architecture/CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md)). Gmail/Outlook now have a full design ([../architecture/EMAIL_INGESTION.md](../architecture/EMAIL_INGESTION.md)), with its first milestone (EM1 — multi-member schema foundation) implemented 2026-07-23; the connector itself (OAuth, sync, ingestion, UI) remains unbuilt — EM2 onward, still not scheduled ahead of Track B validation.
- ~~Extending collection-based retrieval filtering to the portal's own chat (currently Slack-only)~~ — done (backlog M10).
- Client-facing analytics (question volume, document counts, gap trends) — the backend already computes most of the needed data; only rendering is missing. See [../product/KNOWLEDGE_ANALYTICS.md](../product/KNOWLEDGE_ANALYTICS.md).
- ~~An admin review workflow for knowledge gaps, using the already-defined `status` lifecycle~~ — done (backlog M5). See [../product/KNOWLEDGE_GAP_DETECTION.md](../product/KNOWLEDGE_GAP_DETECTION.md).
- Tool/function calling and multi-step retrieval loops, moving from single-shot RAG toward more agentic behavior — no work started, no schema exists. See [../product/AI_AGENTS.md](../product/AI_AGENTS.md).

## Related Documents

- [FEATURE_BACKLOG.md](FEATURE_BACKLOG.md) — item-level technical backlog
- [CONNECTOR_ROADMAP.md](CONNECTOR_ROADMAP.md) — connector-by-connector sequencing
- [../go-to-market/DEMO_VIDEO_STRATEGY.md](../go-to-market/DEMO_VIDEO_STRATEGY.md) — full demo narrative strategy, storyline, and script framework
- [../architecture/SYSTEM_OVERVIEW.md](../architecture/SYSTEM_OVERVIEW.md)
- [../decisions/](../decisions/)
- [../history/ARCHITECTURE_REVIEW_PHASES.md](../history/ARCHITECTURE_REVIEW_PHASES.md)
