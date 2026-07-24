# Connector Roadmap

Source repository: `relativitysystems/Relativity`. This document sequences external connectors and states, for each, what's implemented today versus planned. See [../architecture/CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md) for the detailed current architecture of every connector listed here, and [../decisions/ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md](../decisions/ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md) for why every connector lives in Relativity.

## Purpose

Answer, for each current or future connector: is it implemented, partially implemented, planned, or future — and what would building or completing it require, given the pattern Slack already established.

## Current Connector Foundation

Every connector should follow the reusable pattern extracted from Slack's implementation ([../architecture/CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md)): authenticate via `oauth_connections`/`oauth_credentials` (encrypted, provider-generic schema), resolve tenant only from a trusted server-side lookup (never a client-supplied identifier), normalize the unit of work (question or document), assign documents to a default collection at first ingest, and call AIKB's existing knowledge pipeline rather than implementing a parallel retrieval path. This pattern is **observed, not enforced** — nothing in the codebase prevents a new connector from bypassing it.

## Connector Status

| Connector | Status | Auth model | Notes |
|---|---|---|---|
| **Slack** | **Implemented** — reference implementation, including bounded delivery retries | Encrypted OAuth (`oauth_connections`/`oauth_credentials`), hashed server-side state, HMAC-verified events | Full Q&A path: app-mention -> verified event -> shared knowledge pipeline -> threaded reply, with bounded in-flow retries and a terminal `delivery_failed` state (no scheduled sweep) on delivery failure. See [../architecture/CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md), [../decisions/ADR-003-SLACK-EVENTS-LIVE-IN-RELATIVITY.md](../decisions/ADR-003-SLACK-EVENTS-LIVE-IN-RELATIVITY.md), and [../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md](../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md). |
| **Google Drive** | **Partially implemented** | None — one-shot Picker-obtained browser token only; no persistent connection | One-shot import only (Picker UI); no recurring sync, no webhook. Backlog M15 removed the separate persistent-OAuth-connection flow (`oauth_connections`/`oauth_credentials`, backlog H2) entirely — it had no UI entry point, no recurring sync to drive, and duplicated the Picker import inconsistently. If a real recurring sync is ever built, rebuild the connect flow alongside it rather than reintroducing it standalone. |
| **Dropbox** | **Removed (backlog M15)** | None | Previously had encrypted OAuth connect/status (`oauth_connections`/`oauth_credentials`, backlog H2) but no working import path was ever built — `listFiles()` was orphaned code with no calling route. Rather than build an import path, the entire connector (Connect card, routes, service) was removed as dead weight. `dropbox` remains a valid `oauth_connections.provider` CHECK value in case a real connector is built later. |
| **Microsoft Teams** | **Planned** | Would use `oauth_connections` (`provider = 'microsoft'`, already in the CHECK constraint) | No adapter exists. Would mirror Slack's shape closely — Adaptive Card formatting instead of `mrkdwn`, same OAuth/event/delivery split. |
| **Gmail** | **Planned — schema foundation implemented (EM1, 2026-07-23)** | Would use `oauth_connections` (`provider = 'gmail'`, already in the CHECK constraint), now per-member: `uq_oauth_connections_active_per_client_provider_member` permits one active Gmail connection per `(client_id, member_id)`, not just per client. See [../architecture/EMAIL_INGESTION.md](../architecture/EMAIL_INGESTION.md). | Full per-member mailbox design in EMAIL_INGESTION.md. EM1 (this repo's six new Relativity tables + two new AIKB tables + the `oauth_connections`/`replace_active_oauth_connection` changes) is implemented; no OAuth adapter, sync, ingestion, or UI exists yet (EM2 onward). Threading is looser than Slack (no reliable thread id in all cases) — needs thread-matching heuristics. |
| **Outlook** | **Future** | Would mirror Gmail via Microsoft Graph | No adapter exists; would follow Gmail's design once built. |
| **Meeting transcripts** (e.g. Zoom, Gong-style sources) | **Future** | Not yet designed | Would be a Connected Source (Import/Sync role), not a Knowledge Surface — no one "asks a question" through a transcript feed. |
| **CRM systems** | **Future** | Not yet designed | Would be a Connected Source; no adapter or schema work has started. |

## Shared Connector Primitives (Implemented)

- **OAuth**: `oauth_connections` (metadata) + `oauth_credentials` (AES-256-GCM ciphertext). One active connection per `(client_id, provider)` for Slack/Google Drive/Dropbox, and (since EM1, 2026-07-23) one active connection per `(client_id, provider, connected_by_member_id)` for `gmail`/`microsoft` specifically — two provider-partitioned partial unique indexes, not one — so multiple members of the same client can each hold their own active Gmail/Microsoft connection. One active organization per `(provider, external_account_id)` regardless of provider. `provider` CHECK constraint already lists `slack`, `microsoft`, `gmail`, `google_drive`, `dropbox` — adding a new provider needs no schema change, only a new adapter. See [../architecture/EMAIL_INGESTION.md](../architecture/EMAIL_INGESTION.md)'s Implementation Record.
- **State/CSRF protection**: `oauthStateService`'s hashed, single-use, server-side state pattern — used by Slack today (Google Drive and Dropbox used it too, as of backlog M1, until their persistent-connection flows were removed entirely by backlog M15). Remains provider-generic and available to any new connector.
- **Event idempotency**: a per-provider event log table (`slack_event_log` today) keyed by `UNIQUE(provider, external_event_id)`, with a `received -> enqueued -> answered -> delivered`/`failed`/`delivery_failed` state machine — the latter a distinct, redaction-triggering terminal state for exhausted bounded delivery retries ([ADR-007](../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md)) — reusable shape for any future webhook-driven provider.
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
