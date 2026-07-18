# Connector Roadmap

Source repository: `relativitysystems/Relativity`. This document sequences external connectors and states, for each, what's implemented today versus planned. See [../architecture/CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md) for the detailed current architecture of every connector listed here, and [../decisions/ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md](../decisions/ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md) for why every connector lives in Relativity.

## Purpose

Answer, for each current or future connector: is it implemented, partially implemented, planned, or future — and what would building or completing it require, given the pattern Slack already established.

## Current Connector Foundation

Every connector should follow the reusable pattern extracted from Slack's implementation ([../architecture/CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md)): authenticate via `oauth_connections`/`oauth_credentials` (encrypted, provider-generic schema), resolve tenant only from a trusted server-side lookup (never a client-supplied identifier), normalize the unit of work (question or document), assign documents to a default collection at first ingest, and call AIKB's existing knowledge pipeline rather than implementing a parallel retrieval path. This pattern is **observed, not enforced** — nothing in the codebase prevents a new connector from bypassing it.

## Connector Status

| Connector | Status | Auth model | Notes |
|---|---|---|---|
| **Slack** | **Implemented** — reference implementation | Encrypted OAuth (`oauth_connections`/`oauth_credentials`), hashed server-side state, HMAC-verified events | Full Q&A path: app-mention → verified event → shared knowledge pipeline → threaded reply. See [../architecture/CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md) and [../decisions/ADR-003-SLACK-EVENTS-LIVE-IN-RELATIVITY.md](../decisions/ADR-003-SLACK-EVENTS-LIVE-IN-RELATIVITY.md). |
| **Google Drive** | **Partially implemented** | Legacy plaintext OAuth (`oauth_tokens`) for a persistent connection (no UI entry point) + a separate Picker-obtained browser-token import flow | One-shot import only (Picker UI); no recurring sync, no webhook. Not migrated to encrypted credentials. |
| **Dropbox** | **Partially implemented** | Legacy plaintext OAuth (`oauth_tokens`) | Connect/status only — `listFiles()` is orphaned code with no calling route; no working import path exists today. |
| **Microsoft Teams** | **Planned** | Would use `oauth_connections` (`provider = 'microsoft'`, already in the CHECK constraint) | No adapter exists. Would mirror Slack's shape closely — Adaptive Card formatting instead of `mrkdwn`, same OAuth/event/delivery split. |
| **Gmail** | **Planned** | Would use `oauth_connections` (`provider = 'gmail'`, already in the CHECK constraint) | Dual role possible: a Knowledge Surface (ask a question via email) and/or a Connected Source (import Drive-attached content). No adapter exists. Threading is looser than Slack (no reliable thread id in all cases) — needs thread-matching heuristics. |
| **Outlook** | **Future** | Would mirror Gmail via Microsoft Graph | No adapter exists; would follow Gmail's design once built. |
| **Meeting transcripts** (e.g. Zoom, Gong-style sources) | **Future** | Not yet designed | Would be a Connected Source (Import/Sync role), not a Knowledge Surface — no one "asks a question" through a transcript feed. |
| **CRM systems** | **Future** | Not yet designed | Would be a Connected Source; no adapter or schema work has started. |

## Shared Connector Primitives (Implemented)

- **OAuth**: `oauth_connections` (metadata) + `oauth_credentials` (AES-256-GCM ciphertext), one active connection per `(client_id, provider)` and one active organization per `(provider, external_account_id)`, both enforced by partial unique indexes. `provider` CHECK constraint already lists `slack`, `microsoft`, `gmail`, `google_drive`, `dropbox` — adding a new provider needs no schema change, only a new adapter.
- **State/CSRF protection**: `oauthStateService`'s hashed, single-use, server-side state pattern (Slack only today — not yet backported to Google Drive/Dropbox, which still use an unsigned base64 `state` parameter).
- **Event idempotency**: a per-provider event log table (`slack_event_log` today) keyed by `UNIQUE(provider, external_event_id)`, with a `received → enqueued → answered → delivered`/`failed` state machine — reusable shape for any future webhook-driven provider.
- **Collection-scoped retrieval**: a per-client, per-provider allow-list join table (`slack_collection_access` today), fail-closed by default.

## Requirements Not Yet Built (apply to any new connector)

- **Incremental sync / recurring ingestion.** Every current ingestion entry point is one-shot and user-initiated; there is no "Knowledge Sync" concept (a connected, continuously-polled source with its own cursor/watermark state) in either repository. A future continuously-synced Google Drive folder, or any recurring-ingestion connector, needs this built from scratch.
- **Deletion handling for synced sources.** Since no recurring sync exists, there is no corresponding "source deleted upstream → remove from AIKB" flow either.
- **Provenance beyond `source_provider`/`source_file_id`.** Current provenance is a flat pair of fields; a richer per-source metadata model (e.g. a Drive folder path, a Slack channel origin) is partially present (`document_import_log.source_path`) but not standardized across providers.
- **Webhook infrastructure beyond Slack's specific shape.** Slack's HMAC verification and event-log pattern is reusable in shape, but no shared/generic webhook-verification middleware exists — each new provider currently re-implements its own signature check.
- **Collection mapping defaults per channel/provider sub-scope.** Slack's `slack_collection_access` is client-wide, not per-channel; a `Channel` sub-entity with its own default-collection mapping was proposed (Phase 3) but not built.
- **Permissions beyond client-wide entitlement.** No per-user or per-employee authorization model exists for any connector (tracked as Milestone 7 for Slack specifically — see [../history/ARCHITECTURE_REVIEW_PHASES.md](../history/ARCHITECTURE_REVIEW_PHASES.md)).

## Related Documents

- [../architecture/CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md)
- [../decisions/ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md](../decisions/ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md)
- [../decisions/ADR-003-SLACK-EVENTS-LIVE-IN-RELATIVITY.md](../decisions/ADR-003-SLACK-EVENTS-LIVE-IN-RELATIVITY.md)
- [MASTER_ROADMAP.md](MASTER_ROADMAP.md)
- [FEATURE_BACKLOG.md](FEATURE_BACKLOG.md)
