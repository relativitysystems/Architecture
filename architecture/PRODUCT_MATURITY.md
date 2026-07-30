# Product Maturity

Source repositories: `relativitysystems/Relativity` and `relativitysystems/AIKB`. This document is the **single source of truth** for how Relativity Systems evolves from prototype to commercial platform. Every roadmap document — [MASTER_ROADMAP.md](../roadmap/MASTER_ROADMAP.md), [FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md), [CONNECTOR_ROADMAP.md](../roadmap/CONNECTOR_ROADMAP.md) — should describe work in terms of which stage it advances, rather than inventing its own sequencing language. Where another document's status claim and this one disagree, treat this document as authoritative and fix the other one.

## Purpose

Answer, at a glance: **what stage is the platform in today, what does the next stage require, and why is the sequence ordered this way?** This document does not restate implementation detail already covered elsewhere — it links to the architecture/product document that owns each capability and states only the maturity judgment: implemented, partial, or not started.

## Product Vision (context for every stage below)

Relativity Systems is evolving into a **Company Knowledge Infrastructure** — an AI operating memory that combines uploaded documents, Google Drive, Slack, Gmail, and (later) CRM systems, meetings, and phone transcripts. The AI answers using both **durable organizational knowledge** (what's been ingested and indexed) and **live organizational context** (what a connected system says right now). Every stage below is a step toward that combination; a feature that doesn't serve either "make durable knowledge more complete/trustworthy" or "make live context safely reachable" doesn't belong on this roadmap.

## Stage Overview

| Stage | Name | Status | One-line definition |
|---|---|---|---|
| 1 | Prototype | **Complete** | Core RAG works, one client, one portal |
| 2 | Beta | **Complete** | Multiple connectors, collections, real onboarding |
| 3 | Version 1 | **In progress — the current target** | Demo-ready and first-customer-ready: durable knowledge + live email + coverage/analytics intelligence + operational trust |
| 4 | Expansion | Not started | More connectors, workflow automation, agent execution |
| 5 | Company Operating Memory | Long-term vision | The full product vision realized across every organizational system |

A stage is not "done" until every item in it is implemented and verified — not merely designed. Items marked **proposed** or **planned** elsewhere in this repository are not stage-complete regardless of how detailed their design document is.

---

## Stage 1 — Prototype (Complete)

The platform's original, single-tenant-feeling core. Every item below is implemented and in production.

| Capability | Status | Reference |
|---|---|---|
| Core RAG (retrieval + generation pipeline) | Implemented | [AI_AGENTS.md](../product/AI_AGENTS.md), [AIKB.md](AIKB.md) |
| Document uploads (single/multi-file, ZIP, folder picker) | Implemented | [INGESTION_PIPELINE.md](INGESTION_PIPELINE.md), [CLIENT_PORTAL.md](../product/CLIENT_PORTAL.md) |
| Collections | Implemented (full CRUD, fail-closed retrieval enforcement) | [ADR-005](../decisions/ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md) |
| Citations | Implemented (structured `sources[]`, never model-authored) | [AIKB.md](AIKB.md) |
| Authentication | Implemented (Supabase Auth JWT for members; shared-password admin auth, [flagged as a scaling gap](../roadmap/FEATURE_BACKLOG.md) — H7) | [SECURITY.md](SECURITY.md) |
| Tenant isolation | Implemented at the application layer only — **no database-level RLS backstop today**; carried forward as a Stage 3 Trust gap, not resolved by Stage 1/2 | [SECURITY.md](SECURITY.md) |
| Basic portal | Implemented (upload, chat, chat history, team management) | [CLIENT_PORTAL.md](../product/CLIENT_PORTAL.md) |

---

## Stage 2 — Beta (Complete)

Multiple real connectors and organizational structure on top of the Stage 1 core.

| Capability | Status | Reference |
|---|---|---|
| Google Drive | Implemented — **one-shot Picker import only, not continuous sync** | [CONNECTOR_FRAMEWORK.md](CONNECTOR_FRAMEWORK.md) |
| Slack | Implemented — reference connector, bounded delivery retries, terminal `delivery_failed` state | [CONNECTOR_FRAMEWORK.md](CONNECTOR_FRAMEWORK.md), [ADR-007](../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md) |
| Knowledge collections | Implemented (see Stage 1) | — |
| Automatic synchronization | **Implemented for Gmail only** (EM1–EM10, cron-tick scheduler per [ADR-009](../decisions/ADR-009-EMAIL-AUTOMATIC-SYNC-SYSTEM-CLOCK.md)); Slack and Drive remain one-shot/event-driven with no recurring sync — this gap is exactly what Stage 3's **Automatic Sync Polish** priority closes | [EMAIL_INGESTION.md](EMAIL_INGESTION.md), [CONNECTOR_ROADMAP.md](../roadmap/CONNECTOR_ROADMAP.md) |
| Connector management | Implemented (Slack + Gmail connect/status/disconnect UI; Drive/Dropbox persistent-connection UI was built then removed as unused scaffolding — backlog M15) | [CLIENT_PORTAL.md](../product/CLIENT_PORTAL.md) |
| Improved onboarding | **Partial** — an onboarding-progress checklist exists on the portal Overview tab; there is still no self-serve signup, no bulk/API-driven migration path, and no migration-specific progress UI | [CLIENT_ONBOARDING.md](../product/CLIENT_ONBOARDING.md) |

---

## Stage 3 — Version 1 (Public Demo / First Customers)

**This is the most important stage and the current focus of the entire roadmap.** Version 1 is the minimum product worthy of a public demo and early customer onboarding — not the platform's final form, and not merely "whatever is built today." It is organized into three pillars. All three must be real (implemented and verified) before Version 1 is called demo-ready; see [MASTER_ROADMAP.md](../roadmap/MASTER_ROADMAP.md)'s Version 1 Priorities for the sequencing and dependency chain across pillars.

### Pillar 1: Knowledge Foundation

The durable-knowledge half of the product vision. Mostly complete; the one open item is what Stage 3's fourth priority (Automatic Sync Polish) closes.

| Capability | Status |
|---|---|
| Uploads | Implemented (Stage 1) |
| Google Drive | Implemented, one-shot (Stage 2) |
| Slack | Implemented (Stage 2) |
| Collections | Implemented (Stage 1) |
| Citations | Implemented (Stage 1) |
| Automatic sync | Implemented for Gmail; **not yet for Slack/Drive** — closing this gap, and validating Gmail's automatic sync against a real account (EM10.5), is Version 1 Priority 4 |

### Pillar 2: Intelligence

The half of the product vision that isn't built yet. This is the primary gap between "Beta" and a real Version 1 — a platform that only ever answers from a synchronized copy of a document, with no visibility into what's missing or how well it's working, is not yet the "AI operating memory" the product vision describes.

| Capability | Status | Reference |
|---|---|---|
| Live Email Lookup | **Proposed, not implemented.** Full design (EL1–EL12) exists, ADR-010 shape confirmed 2026-07-30, zero milestones built | [LIVE_EMAIL_LOOKUP.md](LIVE_EMAIL_LOOKUP.md), [ADR-010](../decisions/ADR-010-LIVE-TOOL-CALLS-ORCHESTRATED-BY-AIKB.md) |
| Knowledge Coverage | **New strategic initiative, not implemented.** No coverage score, source inventory, or gap-recommendation system exists today | [KNOWLEDGE_COVERAGE.md](../product/KNOWLEDGE_COVERAGE.md) |
| Knowledge Gap Detection | Implemented (detection, persistence, dedup, admin review) — the one piece of Intelligence already real, and a direct input to Knowledge Coverage | [KNOWLEDGE_GAP_DETECTION.md](../product/KNOWLEDGE_GAP_DETECTION.md) |
| Knowledge Analytics | **Partial.** Backend aggregation endpoints exist (`/summary`, `/analytics`, `/stats`); no client-facing dashboard, no time-series, no cross-client rollup UI | [KNOWLEDGE_ANALYTICS.md](../product/KNOWLEDGE_ANALYTICS.md) |

### Pillar 3: Trust

The operational and security maturity a paying customer should expect before relying on the platform daily. Mixed status — some pieces exist per-connector but nothing is unified.

| Capability | Status |
|---|---|
| Connector health | Partial — Slack has a `delivery_failed` terminal state, Gmail has a sync-run history view; no unified cross-connector health surface, no monitoring/alerting for either |
| Audit logs | Partial — `slack_event_log`, `email_ingestion_events` exist per-connector; no unified audit surface, no retention policy decided for either |
| Sync monitoring | Partial — exists for Gmail (EM7 sync-run history); does not exist for Slack or Drive |
| Security | Documented (`SECURITY.md`); known open gaps tracked in [FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md) (H6: rate-limit stores don't survive multi-replica; H7: shared admin password doesn't scale to multiple admins) |
| Tenant isolation | Application-layer only — RLS is enabled on every table but has zero policies, and every DB client uses the service-role key, so it provides no practical backstop today |
| Source transparency | Implemented (citations, Stage 1) |

### What "demo-ready" requires

A prospect-facing demo does not require every Trust item fully solved — see [MASTER_ROADMAP.md](../roadmap/MASTER_ROADMAP.md) for the exact bar. It does require Knowledge Foundation complete, at least Live Email Lookup and Knowledge Coverage real (not merely designed), Knowledge Analytics visible to a client, and the Trust gaps that are cheap to close (connector health, sync monitoring) closed before customers rely on the product daily rather than watch a demo of it.

---

## Stage 4 — Expansion (Not Started)

Everything below is future work, sequenced after Version 1 proves itself with real customer feedback — consistent with this platform's existing "don't pre-commit to the next connector before feedback exists" discipline (see [MASTER_ROADMAP.md](../roadmap/MASTER_ROADMAP.md)).

| Capability | Status | Reference |
|---|---|---|
| CRM integrations | ADR-010 shape decided (live-tool-call pattern); no connector built | [ADR-010](../decisions/ADR-010-LIVE-TOOL-CALLS-ORCHESTRATED-BY-AIKB.md) |
| Meeting platforms | Future, undesigned | [CONNECTOR_ROADMAP.md](../roadmap/CONNECTOR_ROADMAP.md) |
| Phone transcription | Future, undesigned | — |
| Outlook | Planned (EM12), gated on EM10.5 (Gmail real-account validation) and, for its live-lookup half, EL10/EL12 | [EMAIL_INGESTION.md](EMAIL_INGESTION.md), [LIVE_EMAIL_LOOKUP.md](LIVE_EMAIL_LOOKUP.md) |
| Microsoft Teams | Planned, no adapter built | [CONNECTOR_ROADMAP.md](../roadmap/CONNECTOR_ROADMAP.md) |
| Dropbox improvements | Removed entirely (backlog M15) — a real connector would be a rebuild, not a resumption | [CONNECTOR_ROADMAP.md](../roadmap/CONNECTOR_ROADMAP.md) |
| Workflow automation | Not designed anywhere in this repository yet | — |
| Agent execution | Future — [AI_AGENTS.md](../product/AI_AGENTS.md)'s roadmap (multi-step retrieval loops, cross-session memory, autonomous actions across connectors), building on the tool-calling infrastructure Live Email Lookup (Stage 3) proves out first | [AI_AGENTS.md](../product/AI_AGENTS.md) |

---

## Stage 5 — Company Operating Memory (Long-Term Vision)

The full product vision realized: every organizational system a business relies on — documents, Drive, Slack, Gmail, CRM, meetings, phone calls — feeding one AI operating memory that answers from durable knowledge and live context together, for the entire business, not just one connected surface at a time. This stage is intentionally not broken into milestones here; it is the destination Stage 4's connectors and Stage 3's intelligence/trust infrastructure are built toward. See [vision/PRODUCT_VISION.md](../vision/PRODUCT_VISION.md) for the company-level narrative once that document is populated (currently a stub — no non-codebase source material has been in scope to write it from).

---

## Related Documents

- [../roadmap/MASTER_ROADMAP.md](../roadmap/MASTER_ROADMAP.md) — Version 1 priority sequencing and dependency chain
- [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md) — item-level backlog
- [../roadmap/CONNECTOR_ROADMAP.md](../roadmap/CONNECTOR_ROADMAP.md) — connector-by-connector sequencing and the automatic-sync convergence target
- [LIVE_EMAIL_LOOKUP.md](LIVE_EMAIL_LOOKUP.md) — Intelligence pillar, Live Email Lookup
- [../product/KNOWLEDGE_COVERAGE.md](../product/KNOWLEDGE_COVERAGE.md) — Intelligence pillar, Knowledge Coverage
- [../product/KNOWLEDGE_ANALYTICS.md](../product/KNOWLEDGE_ANALYTICS.md) — Intelligence pillar, Knowledge Analytics
- [../product/AI_AGENTS.md](../product/AI_AGENTS.md) — Stage 4/5 agentic roadmap
