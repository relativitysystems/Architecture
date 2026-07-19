# Architecture Review History

Source: the original `architecture_review_report.md` (Phases 1–4, produced across an architecture discovery, platform design, domain modeling, and Slack MVP implementation effort). This document is a **historical record**. Nothing in it describes the current system by default — where a finding was later resolved, that resolution is noted inline and the finding is not repeated as current fact elsewhere in this documentation set. For the current architecture, see [../architecture/SYSTEM_OVERVIEW.md](../architecture/SYSTEM_OVERVIEW.md).

## Purpose

This document preserves the reasoning, findings, and milestone-by-milestone progression that produced the platform's current Slack integration and connector architecture — so a future reader can understand *why* things are shaped the way they are, not just what they look like today. It is organized the way the original review was conducted: four phases, followed by the implementation notes for each shipped milestone.

---

## Phase 1: Initial Discovery

**Status at the time:** both repositories (`Relativity`, `AIKB`) inspected read-only; no code, migrations, or packages changed.

### Architecture at the time

The intended split — Relativity as surface/identity gateway, AIKB as knowledge engine — was found to be *mostly* real and cleaner than most two-repository splits get. Relativity owned client/team identity (Supabase Auth + `client_members`/`clients` in a "Global" Supabase project), portal/admin UIs, upload intake, and OAuth credential capture for Slack/Google Drive/Dropbox, proxying almost everything document/chat/gap-related to AIKB over HTTP. AIKB owned document parsing/chunking/embedding, pgvector retrieval, LLM answer generation, citations, chat session/message persistence, and knowledge-gap records, with no identity system of its own — it re-validated the caller's Supabase JWT against Relativity's Global Supabase project.

### Critical findings

1. **Two disconnected Slack subsystems.** AIKB had a live, undocumented Slack Events handler (`aikb/routes/slack.js`) completely disconnected from Relativity's Slack OAuth flow. It used a single static global `SLACK_BOT_TOKEN`, derived a fake `clientId` by hashing the Slack channel ID (not a real tenant lookup), and bypassed AIKB's own session/member/knowledge-gap logic. Relativity's OAuth flow captured a real per-client bot token into `oauth_tokens`, but nothing ever read it.
2. **Tenant isolation for AIKB's mutation/listing routes rested entirely on a single shared `x-api-key` secret**, not a per-caller identity check — any holder of that key could read/ingest/delete any client's documents.
3. **No collections/department/per-document access model existed in either repository.** Retrieval was scoped only to `client_id`.

### Other findings (High/Medium/Low)

- No DB-level tenant isolation (RLS) in either repository — confirmed zero `CREATE POLICY`/`ENABLE ROW LEVEL SECURITY` statements anywhere.
- No knowledge-gap idempotency — `createKnowledgeGap` was a plain `INSERT` with no dedup.
- `searchChunksWithTitleBoost`'s second query was only transitively client-scoped, not directly filtered in its own `WHERE` clause — a defense-in-depth gap, not an active leak at the time.
- Dead/orphaned code: `aikbService.clearChatHistory` (unused), `aikb/middleware/resolveContext.js`'s permissive variant (never wired to a route), `aikb/services/googleDriveService.js` (would throw if invoked), `Relativity/services/dropboxService.js#listFiles` (dead).
- Duplicate/contradictory Slack env vars in Relativity (`SLACK_APP_ID`/`SLACK_APP_SECRET` used, `SLACK_CLIENT_ID`/`SLACK_CLIENT_SECRET` unused and dead).
- Admin token comparison was not constant-time (`sig !== expected`).
- Two overlapping AIKB analytics endpoints computed overlapping aggregates independently.

### Recommendations at the time

- Identity/tenancy, integrations (OAuth/webhooks/provider calls), and user-facing surfaces → Relativity.
- Knowledge collections, ingestion, retrieval, answer generation, citations, knowledge gaps, conversation state, background jobs, analytics → AIKB.
- Proposed a cross-repository contract (`POST /query` with `origin`/`originMetadata`/`idempotencyKey`, signed `allowedCollections`) — see [../architecture/SERVICE_CONTRACTS.md](../architecture/SERVICE_CONTRACTS.md) for what was actually built.
- Four decisions were flagged as requiring approval: (1) which repo owns Slack Events ingestion, (2) per-client vs. shared API key between the repos, (3) where the collections/ACL model should live, (4) what to do with the existing dead-end Slack OAuth token. All four were resolved during Phase 4 — see [../decisions/](../decisions/) for the outcomes (ADR-003, ADR-004, ADR-005, and Milestone 2 below, respectively).

### Historical finding — status update (recorded after Phase 4 Milestones 1–3 shipped)

The first and third Critical findings above were resolved — AIKB's Events handler was retired to a `410 Gone` stub (Milestone 1), and Relativity's bot token is now stored AES-256-GCM-encrypted, consumed for the first time by Milestone 4. `SLACK_SIGNING_SECRET` is no longer vestigial. The second Critical finding (shared-`x-api-key`-only routes on AIKB's non-Slack management routes) **remains open by deliberate scope decision** — see [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md) item H4.

---

## Phase 2: Platform Architecture (Design Specification)

**Status:** design-only; no code, migrations, or repository changes were made in producing this phase.

Phase 2 proposed a much larger target architecture than what Phase 4 ultimately shipped: a "Knowledge Surface" abstraction (every interface — Portal, Slack, Teams, Gmail, Outlook, Public API — implementing the same `authenticate`/`identifyUser`/`authorize`/`queryKnowledge`/`formatAnswer`/`deliverAnswer`/`recordAudit` contract), a fully split Collection model (Relativity owns entitlement, AIKB owns assignment and enforcement), a unified signed `ServiceRequest` envelope replacing the shared API key everywhere, a Relativity-side durable outbox for event handoff, and a structured `audit_log` table.

**What actually shipped is narrower**, by deliberate MVP scope decisions made in Phase 4: only Slack was built (not the general Knowledge Surface framework); collections are owned by AIKB end-to-end rather than split (see [ADR-005](../decisions/ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md)); only a narrow `/ask`/`/deliver` HMAC envelope was built, not the full signed `ServiceRequest` platform (see [ADR-004](../decisions/ADR-004-SIGNED-SERVICE-REQUESTS.md)); no durable outbox or structured `audit_log` table exists. These remain **design intent for future work**, not current architecture — see [../roadmap/MASTER_ROADMAP.md](../roadmap/MASTER_ROADMAP.md).

Design principles from this phase that *did* carry through into what was built: every retrieval must be authorized inside the same SQL statement that performs the vector search (implemented — [ADR-005](../decisions/ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md)); no interface bypasses the shared knowledge pipeline (implemented — Slack's `/ask` calls the same `runKnowledgeQuery` function `/query` does); tokens are always encrypted at rest (implemented for Slack — [ADR-006](../decisions/ADR-006-OAUTH-CREDENTIAL-ENCRYPTION.md)); every inbound webhook is signature-verified before any other processing (implemented for Slack).

---

## Phase 3: Domain Model (Specification)

**Status:** specification only; no code, schema, or repository changes were made.

Phase 3 produced a canonical vocabulary intended to replace ambiguous terms platform-wide: **Organization** (replacing Client/Company/Tenant), **Conversation** (replacing Chat Session), **Provider Account** (replacing Workspace), **Knowledge Collection** (replacing Document Group), and a formal **Principal** concept (TeamMember/Guest/ServiceAccount) with default-deny resolution rules.

**This renaming was not adopted in the codebase.** The database and code continue to use `clients`/`client_members`/`client_id` today — "Organization" remains a documentation-only proposal, not an implemented rename. Where this documentation set uses `client_id`/`clients`, that is the actual, current schema; where the original Phase 3 material is referenced, it describes a proposed future vocabulary only.

Of the domain model Phase 3 specified, the parts that materialized in the actual Milestone 4/5 implementation are: `oauth_connections`/`oauth_credentials` (matching Phase 3's OAuth Connection/Token split), `knowledge_collections` (a flatter, AIKB-owned version of Phase 3's Collection concept, with no inheritance), and `origin`/`originMetadata`/`idempotencyKey` fields on conversations and gaps. The full Principal/Guest/Group/ServiceAccount identity model, the Integration/Knowledge-Surface type registry, `audit_log`, and `Notification` entities were **not implemented** — they remain a documented target for future platform growth, not current architecture.

---

## Phase 4: Slack MVP Implementation Roadmap

**Status:** implementation planning, followed by actual implementation across Milestones 1–4 (all shipped and verified in production as of 2026-07-16). Milestones 5–7 were scoped but not started as of the most recent review.

### Scope and non-goals

In scope: one Slack workspace per organization, `@RelativityBot` mentions answered from AIKB's shared, organization-scoped knowledge pipeline, encrypted credentials, deduplicated delivery, Slack-tagged knowledge gaps. Explicitly out of scope for Phase 4: direct messages, fine-grained per-employee authorization, the full Relativity Group/Collection/entitlement model, migrating Google Drive/Dropbox onto encrypted credentials, KMS-managed encryption/key rotation, Teams/Gmail/Outlook adapters, refactoring the shared `x-api-key` gate on AIKB's non-Slack routes, RLS, and a structured `audit_log`.

### Milestone 1 — Neutralize the legacy AIKB Slack route

**Status: Done.** AIKB's `routes/slack.js` was rewritten to answer Slack's `url_verification` challenge (so a still-configured Slack app doesn't 404) and return `410 Gone` for every `event_callback`, logging a structured warning. The unsafe logic (`slackChannelToClientId` hash-derivation, the global-bot-token reply path, the direct `searchChunks` call) was **deleted**, not feature-flagged. Deployed to AIKB (Railway) first, alone, with no dependency on any later milestone.

### Milestone 2 — Encrypted credential storage (Relativity, `feat/encrypted-integration-credentials`)

**Status: Done — committed to `main`, migrated, deployed.**

- New tables: `oauth_connections` (metadata) + `oauth_credentials` (AES-256-GCM ciphertext), migration `supabase/migrations/20260714_oauth_connections.sql`. Two partial unique indexes: one active connection per `(client_id, provider)`, one active organization per `(provider, external_account_id)`.
- New `services/tokenEncryption.js`/`integrationCredentialEncryption.js` — AES-256-GCM, random 12-byte IV per call, 16-byte auth tag, no plaintext fallback.
- Migration's final statement destructively deleted existing plaintext Slack rows: `DELETE FROM oauth_tokens WHERE provider = 'slack'` — executing the already-approved decision to discard the old Slack OAuth flow outright (Phase 1 Decision 4). Google Drive/Dropbox rows untouched.
- Encryption-key variable renamed in a follow-up revision to `INTEGRATION_CREDENTIAL_ENCRYPTION_KEY` (provider-neutral), with `SLACK_TOKEN_ENCRYPTION_KEY` retained only as a deprecated temporary fallback.
- **Tests:** 25 new (initial), growing to 46 across the two Milestone 2 test files after a pre-merge revision; full suite reached **69/69 passing** (up from 51/51 initial, up from 26 pre-existing).
- **Files:** new `supabase/migrations/20260714_oauth_connections.sql`, `services/integrationCredentialEncryption.js`, `services/oauthConnectionsService.js`, two new test files; modified `config/index.js`, `services/supabaseService.js` (deprecation comments only), `.env.example`.
- See [ADR-006](../decisions/ADR-006-OAUTH-CREDENTIAL-ENCRYPTION.md) for the decision this milestone implements.

### Milestone 3 — Slack OAuth connection flow (Relativity, `feat/slack-oauth`)

**Status: Done — committed to `main`, migrated, deployed.**

- New route module `routes/integrations/slack.js` mounted at `/api/integrations/slack`: `GET /start` (owner/admin only), `GET /callback` (public, resolves identity only from server-side state), `GET /status` (any active member), `POST /disconnect` (owner/admin only).
- New `oauth_states` table (migration `20260715_oauth_states.sql`) — only a SHA-256 hash of a 32-byte random state token is persisted; the raw token is never stored. Single-use, atomic consumption (`UPDATE ... WHERE state_hash = $1 AND consumed_at IS NULL AND expires_at > now()`), 10-minute TTL.
- Old flow (`routes/auth.js`'s `/slack/start`/`/slack/callback`) retired to `410 Gone`, all unsafe logic deleted (unsigned `base64(JSON)` state, `incoming-webhook` scope, `upsertToken` calls).
- Portal UI: one card in a new Integrations panel on the Overview tab.
- **Tests:** 79 new across four new test files; full suite reached **144/144 passing** (up from 69/69).
- **Files:** new `supabase/migrations/20260715_oauth_states.sql`, `services/oauthStateService.js`, `services/slackIntegrationService.js`, `routes/integrations/slack.js`, four test files; modified `services/slackService.js` (full rewrite), `services/supabaseService.js`, `routes/auth.js`, `app.js`, `config/index.js`, `.env.example`, portal HTML/CSS/JS.

### Milestone 4 — Slack app mentions and grounded AIKB answers (Relativity `feat/slack-events`; AIKB `feat/slack-ask-pipeline`)

**Status: Done — deployed to production, verified end-to-end against a real Slack workspace on 2026-07-16.** This section originally replaced two earlier, unimplemented milestone drafts ("service-request contract mechanism" and "durability shape") that were superseded before any code was written — those drafts are not preserved separately, since nothing in them was ever built.

**Design:** `/query`'s 293-line handler body was extracted into a shared `runKnowledgeQuery()` function called by both `/query` (`origin: 'portal'`) and a new `POST /api/knowledge/ask` (`origin: 'slack'`) — no second retrieval implementation for Slack. A minimal, additive HMAC-signed envelope (not the full future `ServiceRequest` platform) gates `/ask` and the reversed `/deliver` callback — see [ADR-004](../decisions/ADR-004-SIGNED-SERVICE-REQUESTS.md). Retrieval scope for Slack in this milestone is full-organization, identical to the portal — the "General"-only restriction was deferred to Milestone 5 as optional hardening, not a hard blocker.

**Deployment sequence actually followed:** AIKB's ask-pipeline branch deployed first (inert until called); Relativity's events branch deployed last, only after the Slack app's Event Subscriptions URL was manually pointed at the new endpoint and passed Slack's `url_verification` challenge.

**Production deployment (2026-07-16) — deviations from plan:**
- A `vercel.json` `crons` entry for the delivery-retry sweep was deployed, then **rejected by the Vercel Hobby plan** (any schedule finer than daily is rejected outright, causing deployment failure, not silent degradation). Fixed by removing the `crons` entry entirely (commit `fix: remove unsupported Vercel cron schedule`), not downgrading to daily. **Net effect: the automated retry sweep is currently unscheduled** — recovery from a stuck event depends on AIKB's own Inngest `onFailure` callback, not the independent sweep backstop. Tracked as an open item at the time — see [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md). *(Later superseded: the plan to restore this scheduler was replaced by a bounded-retry/terminal-`delivery_failed` design instead of ever scheduling the sweep — see [ADR-007](../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md).)*
- The first real `app_mention` event was accepted by Relativity and forwarded to AIKB's `/ask` route, but the resulting Inngest event was not processed — **Inngest had not yet synchronized against the newly deployed AIKB instance** and didn't know the new function existed. This was a deployment/registration gap, not an application defect; a manual Inngest sync resolved it.
- Following a **separate security fix** (branch `fix/secure-slack-sweep`, not committed/pushed at time of writing), the sweep endpoint (`GET /api/integrations/slack/sweep`) was found to skip authentication entirely once `CRON_SECRET` stopped being auto-provisioned by Vercel (a consequence of removing the `crons` entry). Root cause: the handler only checked the `Authorization` header `if (config.cron.secret)` was truthy — an unset secret meant the whole check was skipped, not enforced. Fixed via a new `services/cronSweepAuthService.js`: the endpoint is now fail-closed, returning `503` when unconfigured rather than skipping authentication, and using `crypto.timingSafeEqual` for the comparison (previously a plain `!==`). Full Relativity suite reached **278/278 passing** after this fix (up from 247/247).

**Test results:** Relativity reached **247/247 passing** (103 new tests across 10 new files) after the main Milestone 4 branches, then **278/278** after the sweep security fix. AIKB reached **39/39 passing** (31 new tests across 5 new files, up from 8 after Milestone 1).

**Migrations:** Relativity `supabase/migrations/20260716_slack_event_log.sql` (new `slack_event_log` table, `UNIQUE(provider, external_event_id)` dedup); AIKB `migrations/005_slack_origin_tracking.sql` (nullable `origin`/`origin_metadata`/`idempotency_key` on `knowledge_chat_sessions` and `knowledge_gaps`, partial unique index on `idempotency_key`).

**Known limitations recorded at the time (see current-status table below for resolution):** automated retry sweeping disabled (Vercel plan tier); a malformed non-JSON request body to `/events` gets Express's default error response rather than a 200 ack (theoretical, since Slack never sends malformed JSON in practice); no key rotation; the full future signed `ServiceRequest` platform remains unbuilt; AIKB's `.env.example` was found not to document three new environment variables (`SERVICE_REQUEST_SIGNING_SECRET`, `RELATIVITY_API_BASE_URL`, `RELATIVITY_DELIVER_TIMEOUT_MS`) that `config/index.js` reads.

### Milestone 5 — Company-wide knowledge scope ("General"-only restriction)

**Status at last review: not started; superseded by later, larger work.** This milestone as originally scoped was a deliberately narrow stand-in (a single hardcoded `'general'` value on a new `collection` text column, `CHECK`-constrained, no registry table) — explicitly described as a **smaller** and different approach than Phase 3's fuller Collection/Group model, chosen only to avoid over-building before the first bot test.

**What actually shipped instead is larger than this milestone's own scope:** the current `architecture/AIKB.md` documents a fully implemented, general-purpose `knowledge_collections` table (migration `006_knowledge_collections.sql`) — full CRUD, multiple named collections per client (not just a hardcoded `'general'` literal), a `slack_collection_access` join table for per-collection Slack allow-listing, and enforcement inside `match_knowledge_chunks` via a `match_collection_ids UUID[]` parameter. This is a more complete implementation than Milestone 5 as originally scoped, produced by later work not captured in the milestone notes above. See [../architecture/AIKB.md](../architecture/AIKB.md) and [ADR-005](../decisions/ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md) for the current, superseding design.

### Milestone 6 — Fuller knowledge-gap/conversation metadata polish

**Status: not started as of the last review.** Milestone 4's minimal schema slice (nullable `origin`/`origin_metadata`/`idempotency_key` on `knowledge_gaps`) shipped as part of Milestone 4 itself, since `/ask` could not be written correctly without it. The fuller scope — a system-vs-user `reportedBy` distinction, richer `originMetadata` shape, and any conversation-level polish beyond the minimal slice — remains unbuilt. See [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md).

### Milestone 7 — Direct messages + employee-level authorization

**Status: explicitly deferred, no work done or expected imminently.** Covers Slack direct messages, per-employee/Identity-Link-based authorization, and the full Guest/Principal resolution model from Phase 3. See [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md) and [../roadmap/CONNECTOR_ROADMAP.md](../roadmap/CONNECTOR_ROADMAP.md).

---

## Current Status Summary

| Original finding / proposal | Current status | Resolution |
|---|---|---|
| Two disconnected Slack subsystems (AIKB Events handler vs. Relativity OAuth) | **Resolved** | AIKB's handler retired to `410 Gone` (Milestone 1); Slack Events now owned entirely by Relativity (Milestone 4). See [ADR-003](../decisions/ADR-003-SLACK-EVENTS-LIVE-IN-RELATIVITY.md). |
| Slack tenant mapping via a channel-ID hash | **Resolved** | Replaced by a trusted `oauth_connections` lookup keyed on Slack `team_id` (Milestone 2/4). |
| Slack OAuth tokens stored in plaintext | **Resolved** | AES-256-GCM encryption via `oauth_connections`/`oauth_credentials` (Milestone 2). See [ADR-006](../decisions/ADR-006-OAUTH-CREDENTIAL-ENCRYPTION.md). |
| Relativity captures Slack tokens but never consumes them | **Resolved** | Consumed by the Slack delivery path (Milestone 4). |
| `SLACK_SIGNING_SECRET` unused/vestigial | **Resolved** | Wired through and used by Milestone 4's signature verification. |
| No collections/department access model exists | **Resolved, exceeding original scope** | `knowledge_collections` fully implemented in AIKB (beyond Milestone 5's originally-scoped hardcoded stand-in). See [ADR-005](../decisions/ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md). |
| Shared `x-api-key`-only tenant authorization on AIKB's non-Slack routes | **Open — deliberate scope decision** | Unaffected by Milestones 1–4; tracked in [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md) (H4). |
| No DB-level tenant isolation (RLS) | **Open** | Tracked in [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md) (M11). |
| No knowledge-gap idempotency | **Partially resolved** | Schema supports it (`idempotency_key` on `knowledge_gaps`); no write path uses it yet — gaps remain explicit-user-action-only. Tracked (M4). |
| Full signed `ServiceRequest` platform (entitledCollectionIds, principal registry, contract versioning) | **Open — proposed, not built** | Only a narrow `/ask`/`/deliver` HMAC envelope exists. See [ADR-004](../decisions/ADR-004-SIGNED-SERVICE-REQUESTS.md). |
| Google Drive / Dropbox on legacy plaintext `oauth_tokens` | **Open** | Not migrated onto the encrypted model; tracked (H2). |
| Automated Slack delivery-retry sweep | **Superseded — will not be restored** | Implemented but unscheduled due to Vercel Hobby-plan cron limits. The earlier plan to restore its scheduler was superseded by a deliberate product decision: bounded, immediate delivery retries followed by a terminal `delivery_failed` status, with no scheduled sweep of any kind. See [ADR-007](../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md), tracked in [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md). |
| Non-constant-time secret comparisons | **Partially resolved** | Fixed for the sweep endpoint (`fix/secure-slack-sweep`); shared `x-api-key` and admin comparisons elsewhere remain non-constant-time. Tracked (H3). |
| Slack direct messages / employee-level authorization | **Open — deferred (Milestone 7)** | No work done or scheduled. |

## Related Documents

- [../architecture/SYSTEM_OVERVIEW.md](../architecture/SYSTEM_OVERVIEW.md)
- [../decisions/](../decisions/)
- [../roadmap/MASTER_ROADMAP.md](../roadmap/MASTER_ROADMAP.md)
- [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md)
