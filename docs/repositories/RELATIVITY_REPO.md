# Relativity Repository Reference

**Path:** `RelativitySystems/Relativity` · **Remote:** `github.com/relativitysystems/Relativity` · **Branches:** `main`, `feat/slack-events` · **Runtime:** Node/Express, deployed as a single Vercel serverless function.

Verification key used throughout this document: **Code Verified** (directly read in this audit), **Structure Verified** (confirmed via repository file listing, not opened), **Historical** (carried from the pre-existing `architecture/` docs, not independently re-verified line-by-line in this pass), **Inference**.

---

## Entry Points — Structure Verified

| File | Role |
|---|---|
| `app.js` | The Express app itself |
| `server.js` | Local dev entry point |
| `api/index.js` | Vercel serverless wrapper around `app.js` |
| `vercel.json` | Route configuration (`/api/*`, `/auth/*`, `/admin/*`) |

## Folder Layout — Structure Verified

```
Relativity/
├── api/index.js                 # Vercel entry
├── app.js, server.js            # Express app + local entry
├── config/index.js              # Central env-var config (Code Verified — read in full)
├── middleware/                  # clientAuth, adminAuth, apiKey, requireServiceRequest
├── routes/                      # admin, api, auth, leads, team, collections, integrations/slack
├── services/                    # ~25 service modules, see table below
├── supabase/migrations/         # 8 tracked .sql files (earliest: 2026-06-18)
├── public/                      # marketing, portal, admin static frontends
├── test/                        # 24 test files, node:test based (266/266 passing as of the ADR-007 implementation)
└── vercel.json
```

## Routes — Structure Verified (contents not fully read this pass)

| Route file | Covers |
|---|---|
| `routes/auth.js` | Session, OAuth (Google, Dropbox), invites, password reset |
| `routes/api.js` | Uploads, chat/query proxy to AIKB, gaps proxy, Google Drive file routes, analytics |
| `routes/admin.js` | Admin console (clients, leads management) |
| `routes/team.js` | Team member management |
| `routes/leads.js` | Public lead-capture form endpoint |
| `routes/collections.js` | Knowledge-collection CRUD/assignment proxy to AIKB |
| `routes/integrations/slack.js` | Slack OAuth, Events, delivery callback, collections allow-list. The cron sweep route has been removed ([ADR-007](../../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md), implemented) |

## Services — verification status per row

| Service | Purpose | Status |
|---|---|---|
| `supabaseService.js` | Primary Global-DB data-access layer: `clients`, `client_members`, `team_invites`, `client_member_sessions`, `client_portal_issues`, `client_users`, `oauth_tokens` (legacy — `upsertToken`/`getToken` unused by any provider as of backlog H2), plus `deleteClientFull()` client-offboarding cascade | **Code Verified** (read extensively — see audit history in this conversation) |
| `oauthConnectionsService.js` | Encrypted OAuth connection/credential CRUD (`oauth_connections`/`oauth_credentials`), calls `replace_active_oauth_connection` RPC | **Code Verified** (partial) |
| `integrationCredentialEncryption.js` | AES-256-GCM envelope encryption for OAuth secrets | **Historical** (per existing `SECURITY.md`, not re-read this pass) |
| `oauthStateService.js` | CSRF state for Slack OAuth (`oauth_states`, hash-only) | **Structure Verified** |
| `slackCollectionAccessService.js` | Reads/writes `slack_collection_access` — which AIKB collections Slack may search per client | **Code Verified** (read in full) |
| `slackEventLogService.js` | Writes `slack_event_log` (Slack Events dedup/audit); `STATUS.DELIVERY_FAILED` and `markDeliveryFailed()` (redacts `question` in the same UPDATE) implement [ADR-007](../../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md)'s terminal state | **Code Verified** |
| `slackEventsService.js`, `slackIntegrationService.js`, `slackSignatureService.js`, `slackAnswerFormatter.js`, `slackDeliveryService.js`, `slackQuestionService.js` | Slack pipeline: signature verification, event handling, question forwarding to AIKB `/ask`, answer delivery back to Slack | **Historical** |
| `slackDeliverService.js` | Handles `POST /deliver`; bounded, immediate retries (via `retryWithBackoff.js`) on the real-answer delivery leg, terminal `delivery_failed` on exhaustion; AIKB-generation-failure path unchanged (single attempt, generic `failed`) | **Code Verified** ([ADR-007](../../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md)) |
| `retryWithBackoff.js` | Generic bounded-retry helper (configurable attempt count/backoff, injectable sleep) — not Slack-specific | **Code Verified** ([ADR-007](../../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md)) |
| `slackDeliveryFailureService.js` | Shared "mark `delivery_failed` + best-effort redact via AIKB" logic, used by both `slackDeliverService.js` and `slackEventsService.js` | **Code Verified** ([ADR-007](../../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md)) |
| `aikbRedactClient.js` | Signed-envelope client for AIKB's `POST /api/knowledge/chat/redact`; best-effort, single-attempt, not itself retried | **Code Verified** ([ADR-007](../../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md)) |
| `aikbService.js` | HTTP client for AIKB's management API (documents, jobs, analytics) | **Code Verified** (partial — confirms Relativity never queries AIKB tables directly except via `slackCollectionAccessService`) |
| `aikbAskClient.js` | Signed HMAC envelope client for AIKB `/api/knowledge/ask` (Slack path) | **Structure Verified** |
| `serviceRequestAuth.js` | HMAC-signed service-request envelope (shared design with AIKB's copy) | **Historical** |
| `dropboxService.js`, `googleDriveService.js`, `googleDriveImportService.js` | Provider integrations — token storage now encrypted `oauth_connections`/`oauth_credentials` (backlog H2), no longer plaintext `oauth_tokens` | **Structure Verified** |
| `emailService.js` | Transactional email (Resend/SMTP) | **Structure Verified** |
| `importMetadata.js` | Source-label formatting for `document_import_log` | **Structure Verified** |
| `openaiService.js` | Used for audio transcription only (Relativity performs no embedding/chat) | **Historical** |

## Database — Database Verified

Global Supabase project. Full reference: [../supabase/global/DATABASE.md](../supabase/global/DATABASE.md), [TABLES.md](../supabase/global/TABLES.md).

16 `public` tables, all owned by this repo: `clients`, `client_members`, `client_users` (legacy pre-team-member model), `team_invites`, `client_member_sessions`, `client_portal_issues`, `oauth_tokens` (legacy plaintext), `oauth_connections`/`oauth_credentials` (current, encrypted), `oauth_states`, `slack_event_log`, `slack_collection_access`, `document_import_log`, `leads`, plus two confirmed-dead tables `folder_states`/`automation_logs` (see [../audits/SCHEMA_DRIFT.md](../audits/SCHEMA_DRIFT.md)).

Also holds a **second** Supabase client (`config.aikb.supabaseUrl`/`supabaseServiceRoleKey`) for direct, read/write cross-project access into AIKB's `knowledge_collections` and `slack_collection_access`-adjacent flows — confirmed in `services/slackCollectionAccessService.js`.

## Migrations — Database & Code Verified

| File | Adds |
|---|---|
| `20260618_team_members.sql` | `client_members`, `team_invites`, `client_member_sessions`; backfill from `client_users` |
| `20260619_client_members_identity.sql` | Identity linkage adjustments |
| `20260705_document_import_log.sql` | `document_import_log` |
| `20260712_document_import_log_widen.sql` | Widens `document_import_log` |
| `20260714_oauth_connections.sql` | `oauth_connections`/`oauth_credentials`, `replace_active_oauth_connection()` RPC |
| `20260715_oauth_states.sql` | `oauth_states` |
| `20260716_slack_event_log.sql` | `slack_event_log` |
| `20260717_slack_collection_access.sql` | `slack_collection_access` |

**Confirmed schema drift:** `clients`, `oauth_tokens`, `client_users`, `folder_states`, `automation_logs`, and the trigger function `set_updated_at()` all exist live but predate this migration history entirely — no tracked migration creates any of them. See [../audits/SCHEMA_DRIFT.md](../audits/SCHEMA_DRIFT.md).

## Git History Notes — Historical/Code Verified

- `folder_states`/`automation_logs` back an Inngest-based Dropbox folder-watcher feature (`services/stateService.js`) added and fully removed on the same day, 2026-06-07 (commits `b7a7859` → `f7cd4e5`). The DB tables were never dropped.
- `deleteClientFull()` in `supabaseService.js` treats `folder_states`/`automation_logs` as "defensive" deletes, commenting that they "may not exist live" — the live audit shows they do (empty).

## Related documents

[AIKB_REPO.md](AIKB_REPO.md) · [CODE_OWNERSHIP.md](CODE_OWNERSHIP.md) · [../supabase/global/](../supabase/global/) · [../../architecture/SYSTEM_OVERVIEW.md](../../architecture/SYSTEM_OVERVIEW.md) (pre-existing narrative doc, treat as **Historical** pending full re-verification) · [../../decisions/](../../decisions/) (ADRs)
