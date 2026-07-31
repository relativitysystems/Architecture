# Master Roadmap

Source repositories: `relativitysystems/Relativity` and `relativitysystems/AIKB`. This is the high-level architecture and product sequence — it summarizes where the platform has been and where it's headed next, and links out to [FEATURE_BACKLOG.md](FEATURE_BACKLOG.md) and [CONNECTOR_ROADMAP.md](CONNECTOR_ROADMAP.md) for item-level detail rather than duplicating it here. **This document's stage/priority framing is owned by [../architecture/PRODUCT_MATURITY.md](../architecture/PRODUCT_MATURITY.md) — read that document first; this one sequences the work that gets the platform from one stage to the next.**

## Purpose

A new contributor should be able to read this document and answer: what foundation is already built, what stage of product maturity is the platform in right now, and what should be worked on next, in what order. Detailed technical debt lives in the backlog; connector-by-connector sequencing lives in the connector roadmap; stage definitions live in [PRODUCT_MATURITY.md](../architecture/PRODUCT_MATURITY.md); this document is the sequence that ties them together into one story: **MVP → Beta → Version 1 → Platform.**

## Current Platform Foundation

The platform's core split is implemented and stable: Relativity owns identity, tenancy, and every external integration; AIKB owns ingestion, retrieval, reasoning, conversations, and knowledge gaps. See [../architecture/SYSTEM_OVERVIEW.md](../architecture/SYSTEM_OVERVIEW.md) and [../decisions/ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md](../decisions/ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md)/[ADR-002](../decisions/ADR-002-AIKB-OWNS-KNOWLEDGE-PROCESSING.md).

What's built (this is [PRODUCT_MATURITY.md](../architecture/PRODUCT_MATURITY.md)'s Stage 1 + Stage 2 in full — see that document for the itemized status table):
- Multi-tenant document ingestion, chunking, embedding, and vector retrieval (AIKB).
- A working RAG chat pipeline with intent classification, query rewriting, citation generation, and heuristic knowledge-gap detection, including persistence, dedup, and admin review (AIKB). See [../product/KNOWLEDGE_GAP_DETECTION.md](../product/KNOWLEDGE_GAP_DETECTION.md).
- Knowledge collections with SQL-level, fail-closed retrieval enforcement (AIKB).
- A full client portal: authentication, document upload, chat, chat history, collections management, team management (Relativity).
- Three connectors at different maturity levels: Slack (fully modernized reference implementation, including bounded delivery-failure handling), Gmail (per-member ingestion implemented EM1–EM10, real-account validation pending), Google Drive (one-shot Picker import only, no persistent connection or recurring sync). Dropbox was built, then removed as unused scaffolding (backlog M15). See [CONNECTOR_ROADMAP.md](CONNECTOR_ROADMAP.md).
- Bounded, immediate Slack delivery retries with a terminal `delivery_failed` state and cross-repository redaction — no scheduled sweep of any kind. See [ADR-007](../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md).

## Completed Architecture Phases

1. **Phase 1 — Architecture discovery.** Mapped both repositories, found the platform boundary mostly sound but violated by a disconnected, unsafe Slack Events handler in AIKB. See [../history/ARCHITECTURE_REVIEW_PHASES.md](../history/ARCHITECTURE_REVIEW_PHASES.md).
2. **Phase 2/3 — Platform architecture and domain model (design specifications).** Proposed a larger target architecture (Knowledge Surface abstraction, full signed `ServiceRequest` platform, split Collection ownership). Only part of this was built — see the phase summaries in the history document for what carried through versus what remains proposed.
3. **Phase 4 Milestones 1–4 — Slack MVP, shipped to production.** Legacy AIKB Slack handler retired; encrypted OAuth credential storage built; a real Slack OAuth connection flow shipped; `@RelativityBot` mentions answered end-to-end via AIKB's shared knowledge pipeline, verified live against a real Slack workspace on 2026-07-16. See [../history/ARCHITECTURE_REVIEW_PHASES.md](../history/ARCHITECTURE_REVIEW_PHASES.md) for full milestone detail.
4. **Knowledge collections**, implemented in AIKB beyond the original Milestone 5 scope — a full CRUD model with fail-closed Slack enforcement, not just the originally-planned hardcoded stand-in. See [../decisions/ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md](../decisions/ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md).
5. **Slack bounded delivery retries and terminal `delivery_failed` state (ADR-007), shipped to both repositories.** Up to 3 total delivery attempts with short backoff, a terminal `delivery_failed` status on exhaustion, Relativity-side question redaction, and a best-effort cross-repository callback that redacts the corresponding AIKB chat session/message content. See [ADR-007](../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md)'s Implementation Status section for full file-referenced detail.
6. **Gmail ingestion, EM1–EM10 (2026-07-23 through 2026-07-25).** Per-member OAuth connect, organization policy engine, Gmail-label consent workflow, historical + incremental sync into the shared AIKB pipeline, an automatic-sync scheduler (the platform's first recurring-sync mechanism, [ADR-009](../decisions/ADR-009-EMAIL-AUTOMATIC-SYNC-SYSTEM-CLOCK.md)), member offboarding/policy-change reconciliation, and real citations. See [../architecture/EMAIL_INGESTION.md](../architecture/EMAIL_INGESTION.md).

## Where the Platform Is Now

**Stage 1 (Prototype) and Stage 2 (Beta) are complete** — see [PRODUCT_MATURITY.md](../architecture/PRODUCT_MATURITY.md) for the itemized status of every capability in both stages. **The platform is now working toward Stage 3 (Version 1): the minimum product worthy of a public demo and early customer onboarding.** This is a deliberate shift in framing from this document's prior revision, which treated the Beta-stage platform as already fully demo-ready and prioritized go-to-market work above all remaining engineering. That was correct for what it was: the Beta-stage platform genuinely is sufficiently developed to *demonstrate* (see [Demo Video and Sales-Ready Demo Environment](#demo-video-and-sales-ready-demo-environment) below, and [../go-to-market/DEMO_VIDEO_STRATEGY.md](../go-to-market/DEMO_VIDEO_STRATEGY.md), which remains accurate to what exists today). It is a narrower bar than what "Version 1" now means: a product ready not just to be shown, but to be trusted with daily use by early paying customers, with the Intelligence and Trust capabilities the product vision — a company knowledge infrastructure combining durable and live organizational context — actually depends on.

## Version 1 Priorities

**These eight priorities, in this order, are the platform's current roadmap.** The first four complete Version 1's Intelligence and Trust pillars ([PRODUCT_MATURITY.md](../architecture/PRODUCT_MATURITY.md), Stage 3) and are what should make Version 1 provably more than "the Beta platform with a longer feature list." Priority 5 is when that completed product actually reaches prospects and early customers. Priorities 6–8 are deliberately sequenced after 1–5, not before — this platform's own history (Google Drive/Dropbox connection infrastructure built well ahead of the sync engine that would use it, then deleted outright as unused scaffolding, backlog M15) is the direct argument against building further connectors, workflow automation, or agent capability ahead of the intelligence/trust work the product actually needs first.

### 1. Live Email Lookup

**Status: implementation started 2026-07-30.** Full design exists — [../architecture/LIVE_EMAIL_LOOKUP.md](../architecture/LIVE_EMAIL_LOOKUP.md), milestones EL1–EL12, the [ADR-010](../decisions/ADR-010-LIVE-TOOL-CALLS-ORCHESTRATED-BY-AIKB.md) live-tool-call shape confirmed, and every open product decision resolved as of 2026-07-30 (see that document's Decision Log). EL1–EL7A are implemented as of 2026-07-31: architecture/contracts, the read-only tool registry, the signed `POST /api/tools/execute` endpoint, real Gmail search/content tools behind a 9-gate authorization chain (`Relativity/services/emailLiveLookupService.js`, including member consent), AIKB's bounded tool-calling orchestration (`aikb/services/runKnowledgeQuery.js`/`openaiService.js`), the portal's mode selector/consent modal/live-source citations, and Slack identity linking (`Relativity/services/slackUserLinkService.js` — the mapping only, no live-lookup behavior consumes it yet) — a live email question now produces a real answer end-to-end, from the portal UI through to a fixture Gmail mailbox. EL7B onward (Slack live-email access itself, and beyond) are not. Real-account validation (originally sequenced as part of EL8–EL10) remains blocked on EM10.5, unchanged from before this update — EL4–EL7A themselves were built and tested against DI-faked fixtures, not blocked by it.

**Why first**: this is the platform's first real step toward "live organizational context," the second half of the product vision (durable knowledge alone is Stage 1/2). It is also the most fully-specified piece of unbuilt work in this entire roadmap — every other priority below has more open design questions than this one does — so it can start immediately without a design phase first.

**Dependency chain**: EL1–EL5 (architecture, tool registry, signed execution endpoint, Gmail tools, AIKB orchestration) → EL8–EL10 (citations, audit/observability, real-account staging validation) → EL6 (portal UI) → EL7A–EL7B (Slack identity linking, then Slack access) → EL11 (hybrid ingestion recommendations) → EL12 (Outlook, deferred, reassess after real usage). See that document's own Recommended Implementation Order for the full rationale.

**Depends on**: [EM10.5](../architecture/EMAIL_INGESTION.md#em105--gmail-staging-validation) (Gmail ingestion's own real-account validation) having run first, since Live Email Lookup reuses the same OAuth connections and normalization code.

### 2. Knowledge Coverage

**Status: proposed, not implemented.** New strategic initiative — see [../product/KNOWLEDGE_COVERAGE.md](../product/KNOWLEDGE_COVERAGE.md). Coverage score, connected-source inventory, missing-source detection, recommended next integrations, connector completeness, ingestion health, staleness/freshness, unanswered-question rollup, and documentation recommendations, treated as one initiative with one data model rather than scattered feature requests.

**Why second**: a platform that can't tell a client what's missing from its own knowledge undermines the demo narrative the moment a prospect asks a question the system can't answer. This is also the most direct product differentiator versus a generic RAG chatbot — see [../go-to-market/DEMO_VIDEO_STRATEGY.md](../go-to-market/DEMO_VIDEO_STRATEGY.md)'s existing "knowledge gap detection" narrative, which this initiative is a strategic expansion of.

**Depends on**: [Knowledge Gap Detection](../product/KNOWLEDGE_GAP_DETECTION.md) (implemented today, the most direct input this initiative aggregates). Its connector-completeness and freshness capabilities are more meaningful once Priority 4 (Automatic Sync Polish) gives every connector a consistent sync-state signal to read — sequenced after Coverage's own design work can start, but before those specific sub-capabilities are complete.

### 3. Knowledge Analytics

**Status: partial.** Backend aggregation exists (`/summary`, `/analytics`, `/stats`); no client-facing dashboard. See [../product/KNOWLEDGE_ANALYTICS.md](../product/KNOWLEDGE_ANALYTICS.md)'s new Strategic Initiative section — most searched topics, search trends, low-confidence answers, stale/duplicate content, connector health, sync/ingestion statistics, unused knowledge, citation frequency, knowledge growth over time, framed around helping a customer improve their organizational knowledge, not just displaying charts.

**Why third**: this is the initiative with the most existing backend to build on (least new engineering, per capability) and the clearest complement to Knowledge Coverage — Coverage answers "is our knowledge complete and healthy," Analytics answers "how is our knowledge being used, and what does that suggest we should do."

**Depends on**: shares aggregation helpers and, for connector health, sync-state data with Priorities 2 and 4 — sequence these three together, not as fully independent workstreams.

### 4. Automatic Sync Polish

**Status: partial — implemented for Gmail only.** See [CONNECTOR_ROADMAP.md](CONNECTOR_ROADMAP.md)'s new Automatic Sync Convergence section for the full per-connector gap table. Every connector should converge toward: background synchronization, version tracking, change detection, retry handling, health monitoring, and user notifications — "connect once, stay updated automatically."

**Why fourth**: Gmail already proved this mechanism works (EM8, [ADR-009](../decisions/ADR-009-EMAIL-AUTOMATIC-SYNC-SYSTEM-CLOCK.md)); this priority is about generalizing a proven pattern, not inventing a new one, and it directly unblocks the connector-completeness half of Knowledge Coverage and the connector-health half of Knowledge Analytics.

**Recommended near-term scope** (see [CONNECTOR_ROADMAP.md](CONNECTOR_ROADMAP.md)): a unified connector-health surface generalizing Gmail's existing sync-run view; alerting/notification on sync failure or auth expiry for Gmail and Slack; a deliberate decision on whether Google Drive gets a real recurring-sync engine in Version 1 or explicitly waits.

**Depends on**: [EM10.5](../architecture/EMAIL_INGESTION.md#em105--gmail-staging-validation) validating Gmail's existing automatic sync against a real account, since that's the pattern being generalized.

### 5. Demo + Early Customers

**Status: Beta-stage demo is ready today; Version 1 demo-readiness is gated on Priorities 1–4.** See [Demo Video and Sales-Ready Demo Environment](#demo-video-and-sales-ready-demo-environment) below.

### 6. Additional Connectors

**Status: not started beyond what's already built.** Microsoft Teams, Outlook (EM12), meeting-transcript sources, CRM systems. See [CONNECTOR_ROADMAP.md](CONNECTOR_ROADMAP.md) and [PRODUCT_MATURITY.md](../architecture/PRODUCT_MATURITY.md) Stage 4.

**Why sixth, not earlier**: this platform's own history argues against building further connector infrastructure ahead of validated demand — do not pre-commit to the next connector before Priority 5's customer feedback exists.

### 7. Workflow Automation

**Status: not designed anywhere in this repository yet.** A genuinely new capability area — likely depends on the tool-calling infrastructure Priority 1 (Live Email Lookup) proves out, and on [ADR-010](../decisions/ADR-010-LIVE-TOOL-CALLS-ORCHESTRATED-BY-AIKB.md)'s bounded-tool-call pattern extended beyond read-only lookups. No design document exists yet — the first step here is architecture, not implementation.

### 8. Advanced AI Agents

**Status: not started.** See [../product/AI_AGENTS.md](../product/AI_AGENTS.md)'s Future Roadmap — multi-step retrieval loops, cross-session memory, autonomous actions across connectors, streaming responses. Explicitly sequenced last: it is the largest, least-specified body of work in this roadmap, and it builds directly on the tool-calling infrastructure Priority 1 establishes for the first time in either codebase.

## Demo Video and Sales-Ready Demo Environment

The demo video remains a **major company milestone, not a minor marketing task**, and the existing strategy document is accurate to what exists today — see [../go-to-market/DEMO_VIDEO_STRATEGY.md](../go-to-market/DEMO_VIDEO_STRATEGY.md) for the full narrative strategy, fictional-company storyline, feature-by-feature script framework, and production checklists. Nothing about that document's honesty guardrails or product-accuracy table changes here.

**Two distinct bars, stated explicitly so they aren't conflated:**
- **Beta-stage demo (ready today)**: document ingestion, Google Drive one-shot import, Slack Q&A, knowledge collections, source citations, knowledge gap detection — sufficient to demonstrate the current product honestly, per the existing strategy document. This can and should continue to be used for outreach; it does not need to wait for Version 1.
- **Version 1 demo-ready (the new bar this document tracks)**: the Beta-stage demo, plus Live Email Lookup, Knowledge Coverage, and a client-facing Knowledge Analytics view all real and demonstrable, plus the Automatic Sync Polish health/alerting work landed for at least Gmail and Slack. This is the bar a prospect evaluating the product for **daily reliance**, not just a walkthrough, should be held to.

**Acceptance criteria** (Version 1 bar — supersedes the prior revision's Beta-stage criteria, which are now the "ready today" bullet above):
- Every Beta-stage acceptance criterion from the prior revision of this document (stable demo account, working portal/Slack queries, visible citations, honest framing of current vs. future capability, no leaked secrets/production data) still applies unchanged.
- Live Email Lookup is demonstrated: a live, uningested mailbox query answered correctly with a distinct "Live" source badge.
- Knowledge Coverage is demonstrated: a coverage score or missing-source recommendation shown for the demo account.
- Knowledge Analytics is demonstrated: a client-facing view (not just an admin-console number) showing at least one of the new v2 capabilities.
- At least one connector (Gmail or Slack) shows a working health/alerting signal, not just a "connected" status.

**Dependency checklist:**
- [ ] Priority 1 (Live Email Lookup) through at least EL10 (real-account staging validation).
- [ ] Priority 2 (Knowledge Coverage) has a working coverage score/recommendation surface.
- [ ] Priority 3 (Knowledge Analytics) has at least one client-facing view shipped.
- [ ] Priority 4 (Automatic Sync Polish) health/alerting landed for at least Gmail and Slack.
- [ ] Slack bounded-delivery implementation and Knowledge Collections verified in a staging/production-like environment (carried over from the prior revision — still not done; see [../architecture/STAGING_ENVIRONMENT.md](../architecture/STAGING_ENVIRONMENT.md), planned but not yet provisioned).
- [ ] Clean, stable, realistic demo data prepared, extended to cover the four new Version 1 acceptance criteria above (see [FEATURE_BACKLOG.md](FEATURE_BACKLOG.md)).
- [ ] Full narration and screen-action script updated to include the four new feature sections, following [../go-to-market/DEMO_VIDEO_STRATEGY.md](../go-to-market/DEMO_VIDEO_STRATEGY.md)'s existing script requirements and honesty guardrails.
- [ ] Final quality review before publishing (content accuracy, no leaked secrets/customer data, audio/video quality).

No completion date is set here; none exists elsewhere in this roadmap to anchor it to. See [FEATURE_BACKLOG.md](FEATURE_BACKLOG.md) for the corresponding work-breakdown backlog item (GTM1).

## Dependencies

- Priorities 1–4 share real technical overlap (aggregation helpers, sync-state signals, tool-calling infrastructure) and should be planned as one coordinated effort, not four fully independent workstreams run in parallel by unrelated teams.
- Priority 1 (Live Email Lookup) depends on [EM10.5](../architecture/EMAIL_INGESTION.md#em105--gmail-staging-validation) (Gmail ingestion's real-account validation) having run first — it reuses the same OAuth connections and normalization code and should not be built on an unvalidated foundation.
- Priority 4 (Automatic Sync Polish) also depends on EM10.5, since it generalizes the pattern EM10.5 validates.
- Priorities 2 and 3 (Coverage, Analytics) have no hard dependency on Priority 1 and can start in parallel with it — their dependency is on each other (shared aggregation) and, for their connector-completeness/health sub-capabilities specifically, on Priority 4.
- Priority 5 (Demo + Early Customers) depends on Priorities 1–4 reaching the bar stated in this section's Dependency checklist — it is not blocked on Priorities 6–8.
- Priorities 6–8 should not start ahead of Priority 5's customer feedback, per this platform's own Google Drive/Dropbox precedent (backlog M15) for what happens when connector/capability infrastructure is built ahead of validated need.
- Any future connector (Teams, Outlook, a second CRM) should wait until the shared-`x-api-key` gap is closed or explicitly accepted as a known risk for that connector too — see [ADR-004](../decisions/ADR-004-SIGNED-SERVICE-REQUESTS.md). Gmail was already an exception, satisfied by construction (its Relativity→AIKB calls already use ADR-004's signed-envelope pattern).
- Milestone 7 (Slack direct-message employee-level authorization, distinct from the narrower Slack identity-linking EL7A builds for live email lookup specifically) depends on the full Identity Link / Guest / Principal model from the original domain-model proposal, which does not exist today — this is a substantial build, not an incremental one, and is not on this roadmap's current eight priorities.

## Related Documents

- [../architecture/PRODUCT_MATURITY.md](../architecture/PRODUCT_MATURITY.md) — the stage definitions this roadmap sequences work against; treat as authoritative over this document for maturity-stage claims
- [FEATURE_BACKLOG.md](FEATURE_BACKLOG.md) — item-level technical backlog
- [CONNECTOR_ROADMAP.md](CONNECTOR_ROADMAP.md) — connector-by-connector sequencing and the automatic-sync convergence target
- [../architecture/LIVE_EMAIL_LOOKUP.md](../architecture/LIVE_EMAIL_LOOKUP.md) — Version 1 Priority 1
- [../product/KNOWLEDGE_COVERAGE.md](../product/KNOWLEDGE_COVERAGE.md) — Version 1 Priority 2
- [../product/KNOWLEDGE_ANALYTICS.md](../product/KNOWLEDGE_ANALYTICS.md) — Version 1 Priority 3
- [../product/AI_AGENTS.md](../product/AI_AGENTS.md) — Priority 8 and the tool-calling foundation Priority 1 establishes
- [../go-to-market/DEMO_VIDEO_STRATEGY.md](../go-to-market/DEMO_VIDEO_STRATEGY.md) — full demo narrative strategy, storyline, and script framework
- [../architecture/SYSTEM_OVERVIEW.md](../architecture/SYSTEM_OVERVIEW.md)
- [../decisions/](../decisions/)
- [../history/ARCHITECTURE_REVIEW_PHASES.md](../history/ARCHITECTURE_REVIEW_PHASES.md)
