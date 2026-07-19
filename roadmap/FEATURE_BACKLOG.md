# Feature Backlog

Source repositories: `relativitysystems/AIKB` and `relativitysystems/Relativity`. Every item below is derived directly from evidence found during the architecture review that produced this documentation set (dead code, explicit TODO comments, documented security gaps, and schema/code already built but not yet wired up). No item here is speculative product ideation — see `roadmap/MASTER_ROADMAP.md` for that instead.

**Exception:** the [Go-to-Market Priority](#go-to-market-priority) section below is explicitly *not* code-evidence-derived technical debt — it is a business/product deliverable, included here per direct product direction because it is currently the highest-priority item on the roadmap. Everything else in this document keeps its original code-evidence-only scope.

## Overview

This backlog groups technical work into High/Medium/Low priority based on risk and impact observed directly in the code: security gaps affecting tenant isolation or credential handling are High; inconsistencies and unfinished-but-scaffolded features are Medium; dead code and minor cleanup are Low. Each item cross-references the architecture document where the underlying evidence is documented in full.

## Current Implementation

The items below reference the current state of both codebases as documented in [AIKB.md](../architecture/AIKB.md), [INGESTION_PIPELINE.md](../architecture/INGESTION_PIPELINE.md), [CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md), [SECURITY.md](../architecture/SECURITY.md), [CLIENT_PORTAL.md](../product/CLIENT_PORTAL.md), [KNOWLEDGE_ANALYTICS.md](../product/KNOWLEDGE_ANALYTICS.md), and [KNOWLEDGE_GAP_DETECTION.md](../product/KNOWLEDGE_GAP_DETECTION.md). This document does not repeat that detail — it only lists the actionable follow-up each finding implies.

## Architecture

No new architecture is proposed by this document itself; every item below either closes a gap in the existing architecture or completes a feature the existing schema/code already anticipates (e.g., a column or table that exists but has no reader/writer yet).

---

## Go-to-Market Priority

| # | Item | Notes |
|---|---|---|
| GTM1 | **Demo Video and Sales-Ready Demo Account.** Choose the demo customer scenario; prepare realistic business documents; configure General and Slack knowledge collections; verify portal queries; verify Slack questions; prepare a short demo script; record the product walkthrough; add narration explaining pain point, value proposition, and differentiation; edit and review the video; publish it to the appropriate website/outreach location; prepare a shorter cut for direct outreach if useful. Treat this as a product and go-to-market deliverable, not merely content creation — see [../roadmap/MASTER_ROADMAP.md](MASTER_ROADMAP.md)'s dedicated Demo Video milestone for acceptance criteria and the dependency checklist, and [../go-to-market/DEMO_VIDEO_STRATEGY.md](../go-to-market/DEMO_VIDEO_STRATEGY.md) for the full narrative strategy, fictional-company storyline, feature-by-feature script framework, and production checklists. | Depends only on Slack bounded-delivery and Knowledge Collections being verified in staging (both are code-complete; see H5 below and [MASTER_ROADMAP.md](MASTER_ROADMAP.md) Track A) — does **not** wait for any other item in this backlog, including the High-priority security items below. |

---

## High Priority

Security-relevant gaps with a direct tenant-isolation or credential-handling impact.

| # | Item | Evidence |
|---|---|---|
| H2 | Migrate Google Drive and Dropbox OAuth tokens off the legacy plaintext `oauth_tokens` table onto the already-built encrypted `oauth_connections`/`oauth_credentials` model (schema already lists these providers in its CHECK constraint) | [CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md), [SECURITY.md](../architecture/SECURITY.md) |
| H3 | ✅ **Completed.** ~~Replace non-constant-time secret comparisons with `crypto.timingSafeEqual`~~ — shared API key checks now constant-time in both repos (`Relativity/middleware/apiKey.js`, `aikb/routes/knowledge.js`), along with the admin session-signature check (`Relativity/middleware/adminAuth.js`) and the admin password check (`Relativity/routes/admin.js#POST /login`). All comparisons follow the existing length-check-then-`timingSafeEqual` pattern already used by `slackSignatureService.js`/`serviceRequestAuth.js`. 266/266 Relativity and 47/47 AIKB tests passing. | [SECURITY.md](../architecture/SECURITY.md) — Current Risks |
| H4 | Add per-caller entitlement verification to AIKB's `x-api-key`-only routes (ingest/list/delete under `/api/knowledge`), which today trust the shared key plus Relativity's upstream checks without independently re-verifying caller-to-client entitlement | [AIKB.md](../architecture/AIKB.md) — Current Limitations, [SECURITY.md](../architecture/SECURITY.md) |
| H5 | ✅ **Completed.** ~~Slack Terminal Delivery Failure Handling.~~ Bounded, immediate-in-flow Slack delivery retries implemented (3 total attempts by default, configurable); on exhaustion, the `slack_event_log` row is marked with a terminal `delivery_failed` status, the stored question is redacted, and only dedup/diagnostic metadata is retained. The `GET /api/integrations/slack/sweep` scheduled-recovery path has been **removed**, not merely disabled. AIKB-side chat content is redacted via a best-effort signed callback. 266/266 Relativity and 47/47 AIKB tests passing. See [ADR-007](../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md)'s Implementation Status section for full detail, including the follow-ups tracked as M14 below. | [ADR-007](../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md), [CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md), [../history/ARCHITECTURE_REVIEW_PHASES.md](../history/ARCHITECTURE_REVIEW_PHASES.md) |

## Medium Priority

Inconsistencies between providers, and features whose schema/backend already exists but whose UI or automatic behavior does not.

| # | Item | Evidence |
|---|---|---|
| M1 | Backport the hashed, server-stored OAuth-state pattern (`oauthStateService`) from Slack to Google Drive and Dropbox, replacing their unsigned base64 `state` parameter | [SECURITY.md](../architecture/SECURITY.md), [CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md) |
| M2 | Hash team-invite tokens at rest (`team_invites.token` is currently stored in plaintext), mirroring the OAuth-state hash-only pattern | [SECURITY.md](../architecture/SECURITY.md) |
| M3 | Add key-rotation support for `INTEGRATION_CREDENTIAL_ENCRYPTION_KEY` — the `encryption_key_version` column already exists on `oauth_credentials` but is hard-coded to `1` everywhere | [SECURITY.md](../architecture/SECURITY.md) |
| M4 | Implement automatic, deduplicated knowledge-gap persistence server-side, using the `idempotency_key` column and unique index already added in migration 005 but not yet used by any write path | [KNOWLEDGE_GAP_DETECTION.md](../product/KNOWLEDGE_GAP_DETECTION.md) |
| M5 | Build an admin review workflow for knowledge gaps using the existing `status` lifecycle (`open`/`reviewed`/`resolved`/`ignored`) — the column exists but no code currently reads or writes it | [KNOWLEDGE_GAP_DETECTION.md](../product/KNOWLEDGE_GAP_DETECTION.md) |
| M6 | Add general-purpose rate limiting (portal login, admin login, AIKB's `x-api-key`-gated routes) and an explicit CORS policy — neither exists in either repository today | [SECURITY.md](../architecture/SECURITY.md) |
| M7 | Implement the cross-client question-count rollup for the admin console — explicitly marked `// TODO: totalQuestions — needs a dedicated AIKB analytics endpoint` in `Relativity/routes/admin.js` | [KNOWLEDGE_ANALYTICS.md](../product/KNOWLEDGE_ANALYTICS.md) |
| M8 | Surface Google Drive/Dropbox connection status in the portal UI — `GET /auth/me` already returns `googleDriveConnected`/`dropboxConnected`, but neither is read anywhere in `portal.js` | [CLIENT_PORTAL.md](../product/CLIENT_PORTAL.md) |
| M9 | Add a UI control for the existing chat-session-rename endpoint (`PATCH /api/knowledge/chat/sessions/:id/title`) — implemented on the backend, unreachable from the frontend | [CLIENT_PORTAL.md](../product/CLIENT_PORTAL.md) |
| M10 | Extend collection-based retrieval filtering to the portal's own chat query — collections currently scope only Slack's retrieval, not in-portal chat | [CLIENT_PORTAL.md](../product/CLIENT_PORTAL.md), [AIKB.md](../architecture/AIKB.md) |
| M11 | Add Row-Level Security as defense-in-depth on the highest-sensitivity tables (`oauth_tokens`, `oauth_credentials`, `knowledge_chunks`), without removing existing application-layer checks | [SECURITY.md](../architecture/SECURITY.md) |
| M12 | Ship Milestone 6's fuller knowledge-gap/conversation-metadata polish — a system-vs-user `reportedBy` distinction and a richer `originMetadata` shape, on top of Milestone 4's already-shipped minimal `origin`/`origin_metadata`/`idempotency_key` schema slice | [../history/ARCHITECTURE_REVIEW_PHASES.md](../history/ARCHITECTURE_REVIEW_PHASES.md), [KNOWLEDGE_GAP_DETECTION.md](../product/KNOWLEDGE_GAP_DETECTION.md) |
| M13 | Implement Milestone 7 — Slack direct-message support and per-user/employee-level authorization (Slack user → Relativity member identity mapping), explicitly deferred from the Slack MVP | [../history/ARCHITECTURE_REVIEW_PHASES.md](../history/ARCHITECTURE_REVIEW_PHASES.md), [CONNECTOR_ROADMAP.md](CONNECTOR_ROADMAP.md) |
| M14 | **Slack delivery-failure operational follow-ups (H5 completed; these were not part of its scope).** (a) Manual staging/production-like verification of the bounded-retry/`delivery_failed` flow, since automated tests cover logic but not a live Slack workspace/portal walkthrough — see [MASTER_ROADMAP.md](MASTER_ROADMAP.md) Track A. (b) Monitoring/alerting for a sustained rise in `delivery_failed` events, currently visible only in application logs. (c) Observability for AIKB redaction-callback failures — the callback is best-effort and its failures are only logged, never surfaced. (d) Implement the technical-metadata retention/cleanup TODO left in `Relativity/services/slackEventLogService.js` (ADR-007 suggested 7–30 days; no mechanism exists in either repository yet). None of these block the demo video (GTM1) or the Go-to-Market track. | [ADR-007](../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md) — Known Gaps, [CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md) — Current Limitations |

## Low Priority

Dead code removal and minor consistency cleanup with no functional or security impact.

| # | Item | Evidence |
|---|---|---|
| L1 | Remove or wire up `aikb/middleware/resolveContext.js`'s permissive `resolveContext` function — exported but not used by any current route | [AIKB.md](../architecture/AIKB.md) |
| L2 | Remove `Relativity/services/dropboxService.js#listFiles()` or wire it into a real route — currently orphaned, with no calling route in this repository | [CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md) |
| L3 | Remove the "chat welcome chips" dead code in the portal — the HTML element is commented out, but `portal.js` still wires a click handler to it | [CLIENT_PORTAL.md](../product/CLIENT_PORTAL.md) |
| L4 | Remove leftover "TEMP DEBUG" `console.log`/`console.error` statements in the voice-transcription code path (`routes/api.js`, `portal.js`) | [CLIENT_PORTAL.md](../product/CLIENT_PORTAL.md) |
| L5 | Consolidate AIKB's two overlapping analytics endpoints (`/summary`, `/analytics`), which independently compute overlapping aggregates on every call with no shared computation or caching | [KNOWLEDGE_ANALYTICS.md](../product/KNOWLEDGE_ANALYTICS.md) |
| L6 | Clean up `deleteLegacyDocumentsForClient`'s reference to an untracked legacy `documents` table with no migration in this repository | [AIKB.md](../architecture/AIKB.md) |
| L7 | Delete AIKB's now-fully-inert legacy `routes/slack.js` (`410` stub) and its unused `SLACK_BOT_TOKEN`/`SLACK_SIGNING_SECRET` config reads, now that Relativity's real Slack Events endpoint is built and verified in production | [CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md), [../history/ARCHITECTURE_REVIEW_PHASES.md](../history/ARCHITECTURE_REVIEW_PHASES.md) |
| L8 | Document `SERVICE_REQUEST_SIGNING_SECRET`, `RELATIVITY_API_BASE_URL`, and `RELATIVITY_DELIVER_TIMEOUT_MS` in AIKB's `.env.example` — read by `aikb/config/index.js` since Milestone 4 but never added to the example file | [../history/ARCHITECTURE_REVIEW_PHASES.md](../history/ARCHITECTURE_REVIEW_PHASES.md) |
| L9 | **Reclassified from H1.** Remove (or properly wire up) `GET /api/google-drive/files/:clientId` and `/file/:clientId/:fileId` in `Relativity/routes/api.js`. These are gated by the `apiKey` middleware, whose `API_KEY` env var is configured nowhere (`.env`, `.env.example`, or `config/index.js`) — so as written they 401 unconditionally regardless of caller. More importantly, nothing calls them: the portal's real Google Drive import (`portal.js` → `GET /picker-config` + `POST /import`, both `clientAuth`-gated) uses a browser-obtained Picker OAuth token and a different service (`googleDriveImportService`), not `googleDriveService.getValidAccessToken(clientId)`/`listFiles()` as these two routes do. Originally flagged as a High-priority tenant-isolation gap (unchecked `clientId`, shared static key), but since no live caller reaches them, there is no active exploit path today — it's dead code, not a live entitlement gap. Decide: delete outright, or wire up `API_KEY` and add the entitlement check if a future server-to-server ingestion caller is still planned for this path. | [SECURITY.md](../architecture/SECURITY.md) — Client Isolation Gaps |

---

## Current Limitations

This backlog (excluding the [Go-to-Market Priority](#go-to-market-priority) section, which is deliberately business-priority-driven, not evidence-derived) is scoped to what is directly evidenced in the two source codebases as of this review — it excludes further product ideation, new-integration proposals (Gmail/Outlook/Teams/CRM — see [CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md) for the pattern those would follow, not a scheduled item here), and anything requiring information outside the codebase (e.g., customer feedback) to prioritize correctly. Priority levels for the H/M/L sections reflect risk/impact as inferred from code, not business priority, which may reorder these items — the Go-to-Market item is the one deliberate exception, and it currently outranks every H-priority item on business grounds. See [MASTER_ROADMAP.md](MASTER_ROADMAP.md) for the full rationale.

## Future Extension Points

As new architecture documents in this repository are written or updated, new evidence-based items should be added here in the same format (item, one-line description, cross-reference to the document containing the full evidence) rather than as a separate, disconnected backlog — keeping this document the single place technical debt and scaffolded-but-incomplete features are tracked.
