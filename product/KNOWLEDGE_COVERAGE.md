# Knowledge Coverage

**Status of this document: Proposed. Nothing described here is implemented.** No code exists in `relativitysystems/Relativity` or `relativitysystems/aikb` for a coverage score, source inventory, or gap-recommendation engine. This document is planning only, cross-referenced from [MASTER_ROADMAP.md](../roadmap/MASTER_ROADMAP.md) (Version 1 Priority 2) and [PRODUCT_MATURITY.md](../architecture/PRODUCT_MATURITY.md) (Stage 3, Intelligence pillar).

Source repositories referenced for what already exists to build on: `relativitysystems/AIKB` (`services/supabaseService.js`, `services/runKnowledgeQuery.js`), `relativitysystems/Relativity` (`services/aikbService.js`, `routes/admin.js`). Cross-reference [KNOWLEDGE_GAP_DETECTION.md](KNOWLEDGE_GAP_DETECTION.md) (the closest existing capability) and [KNOWLEDGE_ANALYTICS.md](KNOWLEDGE_ANALYTICS.md) (the sibling Intelligence-pillar initiative — Coverage answers "is our knowledge complete and healthy," Analytics answers "how is our knowledge being used").

## Why this is a strategic initiative, not a feature list

Individually, "coverage score," "stale document detection," and "recommended next integrations" look like a handful of small, unrelated dashboard widgets. Treated that way, they'd be built inconsistently, on different schedules, answering different questions with different data models. **Knowledge Coverage is one initiative because it answers one question a client actually asks: "how complete and healthy is what Relativity knows about my business, and what should I do next?"** Every capability below is a different lens on that same underlying question, and should share one data model and one surface, not be scattered across the portal as unrelated widgets.

This is also why Knowledge Coverage is a Version 1 priority, not a Stage 4 nice-to-have: a platform that only ever answers from whatever happens to be ingested, with no way to tell a client what's *missing*, undermines the "AI operating memory" positioning the moment a client asks a question the system can't answer and has no way to explain why.

## Relationship to Knowledge Gap Detection

[KNOWLEDGE_GAP_DETECTION.md](KNOWLEDGE_GAP_DETECTION.md) is implemented today (detection, persistence, dedup, admin review) and is the single most direct input to Knowledge Coverage: every detected gap is evidence of a coverage hole. Knowledge Coverage does not replace or duplicate gap detection — it is the aggregation, scoring, and recommendation layer built on top of it, combined with signals gap detection alone doesn't capture (which sources are connected at all, how stale connected sources are, how complete a connector's ingestion is).

## Proposed Capabilities

| Capability | Description | Primary data source |
|---|---|---|
| **Coverage score** | A single, per-client score (or small set of sub-scores) summarizing how complete the client's connected knowledge is — the headline number the rest of this initiative supports | Aggregates every row below |
| **Connected source inventory** | A list of every source type the client could connect (documents, Drive, Slack, Gmail, future CRM/meetings/phone) and whether each is connected, and if so, since when | `oauth_connections`, upload/import history |
| **Missing knowledge sources** | Which plausible sources for this client are *not* connected at all — the inverse of the inventory above | Inventory + a per-client-type expected-source heuristic (e.g., a client with Slack connected but no documents uploaded) |
| **Recommended next integrations** | A ranked suggestion of which unconnected source would most improve coverage next, given what's already connected and what's been asked | Inventory + knowledge-gap patterns + connector roadmap availability |
| **Connector completeness** | For each *connected* source, how much of what's plausibly available has actually been ingested (e.g., a Gmail connection with automatic sync off, or a Drive import that hasn't been refreshed since initial setup) | Per-connector sync state (`email_sync_state`, `document_import_log`, Slack collection allow-lists) |
| **Ingestion health** | Failed/stuck ingestion jobs, by client and by source, surfaced as a coverage concern rather than only a technical metric | `knowledge_ingestion_jobs` |
| **Stale knowledge detection** | Documents/sources that haven't been updated or re-verified in a long time, flagged as a freshness risk | `knowledge_documents.updated_at`/`created_at`, sync-run timestamps |
| **Knowledge freshness** | The inverse, positive framing of staleness — a per-collection or per-client "how current is this" signal, likely the same underlying computation as staleness presented differently | Same as above |
| **Unanswered question detection** | Questions that repeatedly hit a knowledge gap (already detected and persisted, see gap detection) rolled up by frequency/recency rather than viewed one row at a time | `knowledge_gaps` |
| **Documentation recommendations** | Given a cluster of unanswered questions or a stale/missing source, suggest what a client should write or upload next — the actionable output the rest of this initiative exists to produce | Unanswered-question rollup + missing-source inventory |

## Design Considerations (not yet decided)

- **Score computation is not designed here.** Whether coverage is one number or a small set of sub-scores (e.g., per-source-type completeness), and the exact weighting, is implementation-level design deferred to whoever scopes the first milestone — this document establishes the initiative's shape and inputs, not the scoring formula.
- **Where this is computed.** Following this platform's existing division of labor ([ADR-002](../decisions/ADR-002-AIKB-OWNS-KNOWLEDGE-PROCESSING.md)), coverage computation over knowledge content (staleness, unanswered questions, ingestion health) is a natural AIKB responsibility; connector/source inventory (what's connected at all) is Relativity's, since Relativity owns every integration ([ADR-001](../decisions/ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md)). A coverage score that combines both will need a cross-repository aggregation step, similar in shape to the existing admin-console per-client fan-out (`GET /admin/clients/:clientId/aikb-health`).
- **Reuse, don't duplicate, Knowledge Analytics' aggregation helpers.** `getClientKnowledgeStats`/`getClientSummaryData` (see [KNOWLEDGE_ANALYTICS.md](KNOWLEDGE_ANALYTICS.md)) already compute several of the raw counts this initiative needs (document counts by status, ingestion job failure counts, gap counts). Coverage should extend those shared query helpers rather than re-querying the same tables a second time — the same duplicate-computation problem backlog item L5 already fixed once for Analytics should not be reintroduced here.
- **Client-facing, not admin-only.** Unlike Knowledge Gap Detection's admin-only review workflow today, Coverage is intended primarily as a client-facing signal (a business owner should see their own coverage score and recommendations) — this is a deliberate design difference from how gap review currently works, worth confirming explicitly before implementation rather than defaulting to admin-only because that's the existing pattern.

## Relationship to Automatic Sync Polish

Connector completeness and staleness detection depend on knowing, per connector, what "fully synced" even means — which is exactly what [MASTER_ROADMAP.md](../roadmap/MASTER_ROADMAP.md)'s Automatic Sync Polish priority is building (background synchronization, version tracking, change detection for every connector, not just Gmail). Knowledge Coverage's connector-completeness and freshness capabilities are more meaningful once every connector reports sync state consistently — sequenced accordingly in the roadmap.

## Not Yet Designed

- The exact UI surface (a dedicated Coverage tab vs. an Overview-tab widget vs. folded into a broader Analytics dashboard).
- Whether coverage recommendations should ever be proactively pushed to a client (email/notification) versus pull-only (viewed when the client opens the portal).
- Any specific threshold that triggers a "your knowledge base needs attention" flag.

## Related Documents

- [../roadmap/MASTER_ROADMAP.md](../roadmap/MASTER_ROADMAP.md) — Version 1 Priority 2
- [../architecture/PRODUCT_MATURITY.md](../architecture/PRODUCT_MATURITY.md) — Stage 3, Intelligence pillar
- [KNOWLEDGE_GAP_DETECTION.md](KNOWLEDGE_GAP_DETECTION.md) — the implemented capability this initiative builds on
- [KNOWLEDGE_ANALYTICS.md](KNOWLEDGE_ANALYTICS.md) — the sibling Intelligence-pillar initiative and its shared aggregation helpers
- [../roadmap/CONNECTOR_ROADMAP.md](../roadmap/CONNECTOR_ROADMAP.md) — automatic-sync convergence, a prerequisite for meaningful connector-completeness signals
