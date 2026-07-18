# Feature Backlog

Source repositories: `relativitysystems/AIKB` and `relativitysystems/Relativity`. Every item below is derived directly from evidence found during the architecture review that produced this documentation set (dead code, explicit TODO comments, documented security gaps, and schema/code already built but not yet wired up). No item here is speculative product ideation — see `roadmap/MASTER_ROADMAP.md` for that instead once populated.

## Overview

This backlog groups technical work into High/Medium/Low priority based on risk and impact observed directly in the code: security gaps affecting tenant isolation or credential handling are High; inconsistencies and unfinished-but-scaffolded features are Medium; dead code and minor cleanup are Low. Each item cross-references the architecture document where the underlying evidence is documented in full.

## Current Implementation

The items below reference the current state of both codebases as documented in [AIKB.md](../architecture/AIKB.md), [INGESTION_PIPELINE.md](../architecture/INGESTION_PIPELINE.md), [CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md), [SECURITY.md](../architecture/SECURITY.md), [CLIENT_PORTAL.md](../product/CLIENT_PORTAL.md), [KNOWLEDGE_ANALYTICS.md](../product/KNOWLEDGE_ANALYTICS.md), and [KNOWLEDGE_GAP_DETECTION.md](../product/KNOWLEDGE_GAP_DETECTION.md). This document does not repeat that detail — it only lists the actionable follow-up each finding implies.

## Architecture

No new architecture is proposed by this document itself; every item below either closes a gap in the existing architecture or completes a feature the existing schema/code already anticipates (e.g., a column or table that exists but has no reader/writer yet).

---

## High Priority

Security-relevant gaps with a direct tenant-isolation or credential-handling impact.

| # | Item | Evidence |
|---|---|---|
| H1 | Add per-client entitlement checks to `GET /api/google-drive/files/:clientId` and `/file/:clientId/:fileId` — currently gated only by the shared static API key, with `clientId` taken unchecked from the URL | [SECURITY.md](../architecture/SECURITY.md) — Client Isolation Gaps |
| H2 | Migrate Google Drive and Dropbox OAuth tokens off the legacy plaintext `oauth_tokens` table onto the already-built encrypted `oauth_connections`/`oauth_credentials` model (schema already lists these providers in its CHECK constraint) | [CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md), [SECURITY.md](../architecture/SECURITY.md) |
| H3 | Replace non-constant-time secret comparisons with `crypto.timingSafeEqual` — shared API key checks in both repos, and the admin password/session-signature checks in `Relativity/middleware/adminAuth.js`/`routes/admin.js` | [SECURITY.md](../architecture/SECURITY.md) — Current Risks |
| H4 | Add per-caller entitlement verification to AIKB's `x-api-key`-only routes (ingest/list/delete under `/api/knowledge`), which today trust the shared key plus Relativity's upstream checks without independently re-verifying caller-to-client entitlement | [AIKB.md](../architecture/AIKB.md) — Current Limitations, [SECURITY.md](../architecture/SECURITY.md) |
| H5 | Re-enable scheduled execution of the Slack retry sweep (`GET /api/integrations/slack/sweep`) — the durable-redelivery-recovery code exists and works but has no active scheduler after its Vercel Cron entry was removed due to a hosting-plan limitation | [CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md) |

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

---

## Current Limitations

This backlog is scoped to what is directly evidenced in the two source codebases as of this review — it excludes product ideation, new-integration proposals (Gmail/Outlook/Teams/CRM — see [CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md) for the pattern those would follow, not a scheduled item here), and anything requiring information outside the codebase (e.g., customer feedback) to prioritize correctly. Priority levels reflect risk/impact as inferred from code, not business priority, which may reorder these items.

## Future Extension Points

As new architecture documents in this repository are written or updated, new evidence-based items should be added here in the same format (item, one-line description, cross-reference to the document containing the full evidence) rather than as a separate, disconnected backlog — keeping this document the single place technical debt and scaffolded-but-incomplete features are tracked.
