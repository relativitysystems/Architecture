# Code Ownership Matrix

Cross-reference: [RELATIVITY_REPO.md](RELATIVITY_REPO.md), [AIKB_REPO.md](AIKB_REPO.md), [../../decisions/ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md](../../decisions/ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md), [ADR-002](../../decisions/ADR-002-AIKB-OWNS-KNOWLEDGE-PROCESSING.md).

The platform boundary confirmed by this audit (schemas, functions, and code cross-referenced directly): **Relativity owns identity, tenancy, and every provider integration; AIKB owns ingestion, retrieval, reasoning, conversations, and knowledge gaps.**

## Table ownership — Database Verified

| Table | Project | Owning repo | R/W from other repo? |
|---|---|---|---|
| `clients`, `client_members`, `client_users`, `team_invites`, `client_member_sessions` | Global | Relativity | AIKB: read-only (`globalSupabase` client in `supabaseService.js`, entitlement checks only) |
| `client_portal_issues`, `document_import_log`, `leads` | Global | Relativity | None |
| `oauth_tokens` (legacy), `oauth_connections`, `oauth_credentials`, `oauth_states` | Global | Relativity | None |
| `slack_event_log` | Global | Relativity | None |
| `slack_collection_access` | Global | Relativity | None — but referenced conceptually by AIKB's collection model via the client's Slack allow-list, enforced entirely in Relativity |
| `folder_states`, `automation_logs` | Global | Relativity (dead feature) | None — see [../audits/SCHEMA_DRIFT.md](../audits/SCHEMA_DRIFT.md) |
| `knowledge_documents`, `knowledge_chunks`, `knowledge_ingestion_jobs` | AIKB | AIKB | Relativity: read via HTTP API only (`services/aikbService.js`), never direct DB |
| `knowledge_chat_sessions`, `knowledge_chat_messages` | AIKB | AIKB | Relativity: read via HTTP API for display/ownership filtering |
| `knowledge_gaps` | AIKB | AIKB | Relativity: write via `POST /gaps` proxy only |
| `knowledge_collections` | AIKB | AIKB | Relativity: read/write **directly** via a second Supabase client (`slackCollectionAccessService.js`, `aikbAskClient.js`) — the one confirmed direct cross-project table access in either direction |
| `documents` | AIKB | AIKB (legacy, pre-AIKB-schema) | None — see [../audits/SCHEMA_DRIFT.md](../audits/SCHEMA_DRIFT.md) |

## Function/RPC ownership — Database Verified

| Function | Project | Called by |
|---|---|---|
| `replace_active_oauth_connection` | Global | Relativity (`oauthConnectionsService.js`, via `.rpc(...)`) |
| `set_updated_at()` | Global | Postgres trigger only (`clients`, `oauth_tokens`, `folder_states`) — no direct code caller |
| `match_knowledge_chunks` (5-arg) | AIKB | AIKB (`supabaseService.js:searchChunks`, used by both portal and Slack query paths) |
| `match_knowledge_chunks` (4-arg, legacy) | AIKB | No live caller in either repo |
| `match_documents` | AIKB | Dropped (backlog L6, `007_drop_legacy_documents.sql`, applied and verified in production) |

## API ownership

| API surface | Owner | Notes |
|---|---|---|
| `/auth/*`, `/api/*` (Relativity), `/admin/*`, `/api/integrations/slack/*` | Relativity | Customer-facing gateway |
| `/api/knowledge/*` (AIKB) | AIKB | Management routes (`x-api-key`), member routes (JWT), `/ask` (signed envelope, Slack only) |
| `/api/inngest` | AIKB | Inngest webhook callback |

## Integration ownership — Historical (per existing ADR-001), Structure Verified for file presence

| Integration | Owner | Provider auth model |
|---|---|---|
| Slack | Relativity | OAuth v2 + Events signature + encrypted `oauth_connections`/`oauth_credentials` |
| Google Drive | Relativity | OAuth, legacy plaintext `oauth_tokens` |
| Dropbox | Relativity | OAuth, legacy plaintext `oauth_tokens` |

## Open questions — Unknown

- Whether AIKB's `services/googleDriveService.js` is fully dead code (README states Google Drive sync was "intentionally removed") or retains a partial live code path — not verified in this pass; flag for Phase 10 legacy audit follow-up.
- Full route-by-route ownership detail for every path under `routes/api.js` (Relativity) and `routes/knowledge.js` (AIKB) — file structure confirmed, full contents not yet read line-by-line in this audit.

## Related documents

[RELATIVITY_REPO.md](RELATIVITY_REPO.md) · [AIKB_REPO.md](AIKB_REPO.md) · [../supabase/global/](../supabase/global/) · [../supabase/aikb/](../supabase/aikb/) · [../audits/SCHEMA_DRIFT.md](../audits/SCHEMA_DRIFT.md)
