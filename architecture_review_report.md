# Phase 1 Architecture Discovery

Both repository paths confirmed readable: `C:\Users\tenzin\Relativity` and `C:\Users\tenzin\aikb`. No files were modified, no packages installed, no migrations run.

## 1. Executive summary

**Relativity** (`Relativity/app.js`, deployed on Vercel via `api/index.js`) is a thin, multi-tenant Express **gateway**: it owns client/team identity (Supabase Auth + `client_members`/`clients` in a "Global" Supabase project), portal/admin UIs, upload intake, and OAuth credential capture for Slack/Google Drive/Dropbox. It does **not** do retrieval, embeddings, LLM answering, or gap detection itself — it proxies almost everything document/chat/gap-related to AIKB over HTTP (`services/aikbService.js`).

**AIKB** (`aikb/server.js`, deployed on Railway) is the actual **knowledge engine**: it owns document parsing/chunking/embedding (`inngest/functions.js`, `services/documentParser.js`, `services/openaiService.js`), pgvector retrieval (`match_knowledge_chunks` RPC), LLM answer generation, citations, chat session/message persistence, and knowledge-gap records — all in its own Supabase project. Critically, AIKB does **not** have its own identity system; it re-validates the caller's Supabase JWT against Relativity's **same Global Supabase project** (`GLOBAL_SUPABASE_URL`) for member-authenticated routes.

**Most important architectural conclusion:** the intended split (Relativity = surface/identity, AIKB = knowledge engine) is *mostly* real and cleaner than most two-repo splits get — but there are three findings that must be resolved before any Slack/Teams/email work starts:

1. **AIKB already has a live, undocumented Slack Events handler** (`aikb/routes/slack.js`) that is completely disconnected from Relativity's Slack OAuth flow. It uses a single static global `SLACK_BOT_TOKEN`, derives a fake `clientId` by hashing the Slack channel ID (not a real tenant lookup), and bypasses AIKB's own session/member/knowledge-gap logic. This is not "no Slack integration to build on" — it's **two incompatible half-built Slack integrations in two different repos**, neither of which is a safe foundation as-is.
2. **Tenant isolation for AIKB's mutation/listing routes rests entirely on a single shared `x-api-key` secret**, not on a per-caller identity check — any holder of that key can read/ingest/delete any client's documents. Only the `requireMemberContext`-gated routes (`/query`, `/chat/*`, `/gaps`) independently verify the caller against Global `client_members`.
3. **No collections/department/per-document access model exists in either repo.** Retrieval is scoped only to `client_id`. This blocks the stated future goal (Sales/Engineering/HR/Legal/Executive scopes) and must be designed before Slack (which will need channel→scope mapping) is built.

## 2. Repository maps

### Relativity
- **Runtime**: Node/Express, single serverless function on Vercel.
- **Entry points**: `app.js` (the real app), `server.js` (local dev listener), `api/index.js` (Vercel wrapper — `module.exports = require('../app')`), `vercel.json` (routes all `/api/*`, `/auth/*`, `/admin/*` to that one function).
- **Major route groups**: `routes/auth.js` (`/auth` — session bootstrap, OAuth start/callback for Slack/Google/Dropbox, invites, password reset), `routes/api.js` (`/api` — uploads, chat/query proxy, gaps proxy, analytics), `routes/leads.js` (`/api/leads` — public marketing form), `routes/team.js` (`/api/team` — invites/roles), `routes/admin.js` (`/admin` — admin console, AI-BDR proxy).
- **Services**: `services/aikbService.js` (all AIKB HTTP calls), `services/supabaseService.js` (Global DB access), `services/slackService.js` (27 lines — OAuth token exchange only), `services/googleDriveService.js`/`googleDriveImportService.js`, `services/dropboxService.js`, `services/openaiService.js` (audio transcription only), `services/importMetadata.js`.
- **Database usage**: "Global" Supabase project — `clients`, `client_members`, `team_invites`, `client_member_sessions`, `oauth_tokens`, `leads`, `client_portal_issues`, `document_import_log` (+ deprecated `client_users`). Also writes raw files into **AIKB's** Supabase Storage bucket directly (not AIKB's Postgres).
- **External dependencies**: Supabase (Global project), AIKB REST API, OpenAI (transcription only), Slack OAuth API, Google Drive API, Dropbox API, Resend/SMTP for email.
- **Deployment model**: single Vercel serverless function; static assets served by Vercel CDN, not Express, in production.

### AIKB
- **Runtime**: Node/Express, single process on Railway (inferred — no Railway-specific config file found in repo).
- **Entry points**: `server.js` — mounts `/health`, `/api/inngest` (Inngest's `serve()` handler, same process), `/api/knowledge` (`routes/knowledge.js`), `/api/slack` (`routes/slack.js`).
- **Major route groups**: `routes/knowledge.js` (`x-api-key`-gated: ingest/reindex/delete/list/jobs/summary/analytics; `requireMemberContext`-gated: query/chat/gaps), `routes/slack.js` (Slack Events API — HMAC-signature-gated, no API key required).
- **Services**: `services/supabaseService.js` (dual Supabase clients — AIKB project + Global project), `services/documentParser.js` (txt/md/csv/docx via `mammoth`/pdf via `pdf-parse`), `services/chunkService.js`, `services/openaiService.js` (embeddings + `gpt-4.1`/`gpt-4o-mini` chat), `services/googleDriveService.js` (dead/orphaned — references a `config.googleDrive` that doesn't exist).
- **Database usage**: AIKB's own Supabase project — `knowledge_documents`, `knowledge_chunks` (pgvector), `knowledge_ingestion_jobs`, `knowledge_chat_sessions`, `knowledge_chat_messages`, `knowledge_gaps`. Reads-only from Relativity's **Global** project for `auth.getUser()`, `clients`, `client_members` (no writes).
- **External dependencies**: OpenAI (embeddings + chat), Inngest (in-process, same server), Supabase (two projects), Slack Events/Web API.
- **Deployment model**: single Railway web process; Inngest functions run in-process, invoked via webhook callback to `/api/inngest` — no separate worker service exists in this repo.

## 3. Current responsibility matrix

| Capability | Relativity today | AIKB today | Duplication/conflict |
|---|---|---|---|
| Client/org identity | Owns `clients` table (Global DB) | Reads Global `clients` (read-only, via `requireActiveClient`) | None — clean single-owner |
| User auth (portal) | Owns via Supabase Auth + `client_members` | Re-validates same JWT against same Global DB (`requireMemberContext`) | Duplicated *validation logic*, same source of truth — low risk but two auth code paths to keep in sync |
| Admin auth | Separate password+HMAC scheme (`middleware/adminAuth.js`) | N/A | — |
| Document upload/intake | Multer upload → writes to AIKB Storage → calls `/ingest` | Owns parsing/chunking/embedding | Clean handoff |
| Document metadata | Local `document_import_log` (provenance/labels only) | Owns `knowledge_documents` (source of truth) | Two metadata stores, but Relativity's is explicitly display-only per migration comment |
| Retrieval/vector search | None | Owns (`match_knowledge_chunks` RPC) | Clean |
| LLM answer generation | None (only audio transcription) | Owns (`openaiService.generateRagAnswer`) | Clean |
| Citations | Displays only | Generates + stores | Clean |
| Chat sessions/messages | Local `client_member_sessions` = ownership/visibility map only | Owns actual session/message content | Clean split, but two tables must stay reconciled (AIKB `sessionId` ↔ Relativity `aikb_session_id`) |
| Knowledge gaps | Pass-through route only (`POST /api/knowledge/gaps`) | Owns schema + persistence; **no auto-write, no dedup** | Not duplicated, but no idempotency anywhere in the chain |
| Slack OAuth (per-client) | Owns (`routes/auth.js`, `services/slackService.js`) — captures bot token to `oauth_tokens` | Not involved | **Conflict**: token captured here is never consumed anywhere |
| Slack Events/bot runtime | Not present | **Owns a live, separate implementation** (`routes/slack.js`) using a single static `SLACK_BOT_TOKEN`, not the per-client token Relativity captures | **Critical duplication/conflict** — two disconnected Slack subsystems |
| Google Drive OAuth + import | Owns, fully wired (start/callback/picker/import) | Dead/orphaned code only (`googleDriveService.js`, unreachable, would throw if called) | Not a conflict — AIKB's copy is vestigial |
| Dropbox | OAuth flow exists; `listFiles` is dead code, referenced n8n route no longer exists | N/A | Partial dead code in Relativity only |
| Background jobs | None (no Inngest) | Owns all Inngest functions, in-process on the Railway web server | Clean, but "in-process" means no independent worker scaling |
| Analytics | Displays aggregate counts only | Computes on-the-fly (no dedicated analytics table) | Clean |
| Tenant isolation enforcement | App-layer `client_id` filtering via `clientAuth`; no RLS | App-layer filtering; only the vector-search RPC enforces `client_id` in SQL itself; mutation/list routes trust the shared API key | **Gap in both** — no DB-level (RLS) backstop anywhere |
| Collections/department scoping | Not found | Not found | Neither repo has this — must be designed net-new |

## 4. Data ownership map

| Entity | Relativity | AIKB |
|---|---|---|
| `clients` (Global DB) | R/W (owner) | R only |
| `client_members` (Global DB) | R/W (owner) | R only (member/role lookup) |
| `team_invites` (Global DB) | R/W (owner) | — |
| `client_member_sessions` (Global DB) | R/W (owner — maps to AIKB session ids) | — |
| `oauth_tokens` (Global DB) | R/W (owner — Slack/Google/Dropbox tokens, **plaintext**) | — |
| `document_import_log` (Global DB) | R/W (owner — provenance/display only) | — |
| `knowledge_documents` (AIKB DB) | — (proxies via API only) | R/W (owner) |
| `knowledge_chunks` (AIKB DB, pgvector) | — | R/W (owner) |
| `knowledge_ingestion_jobs` (AIKB DB) | R (via API, display) | R/W (owner) |
| `knowledge_chat_sessions` / `knowledge_chat_messages` (AIKB DB) | R (via API, display/ownership filtering) | R/W (owner) |
| `knowledge_gaps` (AIKB DB) | W (via `POST /gaps` proxy only) | R/W (owner) |
| Background events (Inngest) | — (no Inngest in this repo) | Owner — `knowledge/document.ingest\|reindex\|delete` |
| Supabase Storage — AIKB bucket | W (uploads raw files directly) | R/W (downloads for parsing, deletes on cleanup) |

## 5. Request-flow diagrams

**Portal question:**
```
Browser (portal.js)
  --Bearer JWT--> Relativity POST /api/knowledge/query (routes/api.js)
      [clientAuth resolves client_id server-side]
      --x-api-key + forwarded JWT--> AIKB POST /api/knowledge/query (routes/knowledge.js)
          [requireMemberContext re-validates JWT against Global DB]
          -> intent classification (gpt-4o-mini)
          -> match_knowledge_chunks RPC (pgvector, client_id-scoped)
          -> generateRagAnswer (gpt-4.1)
          -> persist user+assistant messages (knowledge_chat_messages)
          -> [does NOT auto-write knowledge_gaps]
      <-- {answer, sources, sessionId, isKnowledgeGap, gapReason}
  <-- Relativity stores/maps client_member_sessions if new session
Browser <-- rendered answer
```

**Document upload:**
```
Browser --file--> Relativity POST /api/knowledge/upload (routes/api.js, multer memory storage)
  -> aikbService.uploadAndIngest()
       1. uploadToStorage() --> AIKB's Supabase Storage bucket (direct Storage client, bypasses AIKB's API)
       2. POST AIKB /api/knowledge/ingest {clientId, sourceProvider:'portal_upload', sourceFileId, storagePath}
            -> Inngest event knowledge/document.ingest queued
            -> Inngest fn: download from Storage -> parseDocument -> chunkText -> generateEmbeddings -> insertKnowledgeChunks
  -> Relativity also writes document_import_log row (local provenance/display only)
```

**Knowledge-gap creation:**
```
Browser (after receiving isKnowledgeGap:true from a query response)
  --user action--> Relativity POST /api/knowledge/gaps {sessionId, question, reason}
      --x-api-key + JWT--> AIKB POST /api/knowledge/gaps (requireMemberContext)
          -> supabaseService.createKnowledgeGap (plain INSERT, no dedup/idempotency)
```

**Existing OAuth integration (Slack, per-client capture — currently a dead end):**
```
Browser (clientAuth session) --> Relativity GET /auth/slack/start
  -> redirect to Slack authorize URL (state = base64({clientId}))
Slack --> Relativity GET /auth/slack/callback (PUBLIC route, no auth middleware)
  -> slackService.exchangeCodeForToken(code)
  -> supabaseService.upsertToken(clientId, 'slack', access_token)  [PLAINTEXT, oauth_tokens table]
  -> supabaseService.updateClientSlackChannel(clientId, channel_id)
  [nothing in Relativity ever reads this token again — no chat.postMessage call anywhere in the repo]
```

**Existing background job:**
```
Relativity POST /api/knowledge/ingest --> AIKB queues knowledge/document.ingest event
  -> Inngest (in-process on AIKB's Railway server) invokes ingestDocument()
       step: create-job -> check-existing (content-hash dedup) -> update-job-running
       -> download -> parse -> hash -> upsert document -> delete old chunks -> chunk -> embed -> insert chunks
       -> mark-indexed -> complete-job
  (Inngest cloud calls back to AIKB's own /api/inngest webhook to drive step execution)
```

**Separately — live but disconnected Slack bot flow (found in AIKB, not referenced by the task's "known context"):**
```
Slack workspace --event--> AIKB POST /api/slack/events (routes/slack.js)
  -> verifySlackSignature (AIKB's own SLACK_SIGNING_SECRET, independent of Relativity's unused one)
  -> clientId = SHA1-hash(slack_channel_id)   [NOT a real tenant lookup — no relation to Relativity's clients table]
  -> searchChunks(clientId, ...) directly (bypasses /query's intent classification, session/member context, gap logic)
  -> answer generated
  [not confirmed by evidence gathered whether/how it posts back via chat.postMessage — flagged for Phase 2 verification]
```

## 6. Coupling and duplication findings

**Critical**
- **Two disconnected Slack subsystems.** Relativity's OAuth flow (`Relativity/routes/auth.js:171-205`, `Relativity/services/slackService.js`) captures a real per-client bot token into `oauth_tokens`, but nothing ever reads it. AIKB's live Events handler (`aikb/routes/slack.js`) instead uses one static global `SLACK_BOT_TOKEN` and a hash-derived fake `clientId` (`slackChannelToClientId`, `aikb/routes/slack.js:124`) with no real tenant mapping. Building on either as-is would either (a) never actually notify Slack, or (b) leak across tenants, since a hash of a channel ID is not an authorization check tied to `clients`/`client_members`.
- **AIKB's `x-api-key`-only routes have no per-client authorization.** `POST /ingest`, `POST /reindex`, `DELETE /document/:id`, `GET /documents/:clientId`, `GET /jobs/:clientId`, `GET /summary/:clientId`, `GET /analytics/:clientId`, `DELETE /client/:clientId` (`aikb/routes/knowledge.js`) only check that `clientId` is *active* in the Global DB (`requireActiveClient`), not that the caller is entitled to that specific client. A single leaked shared `API_KEY` value grants cross-tenant read/write/delete over every client's documents.
- **Slack bot token stored in plaintext.** `Relativity/services/supabaseService.js` `upsertToken` writes `oauth_tokens.access_token` unencrypted. `SLACK_TOKEN_ENCRYPTION_KEY` is declared in `.env.example` but never referenced by any code (`Relativity` repo-wide grep, confirmed by agent).
- **No signature verification on Relativity's Slack OAuth callback route being public is fine (that's normal OAuth), but the vestigial `SLACK_SIGNING_SECRET` in Relativity's `.env.example` is unused there** — signature verification only exists in AIKB's separate, disconnected bot.

> **Historical finding — status update added after Phase 4 Milestones 1–3:** the first and third Critical findings above are resolved — AIKB's Events handler was retired to a `410 Gone` stub (Milestone 1), and Relativity's bot token is now stored AES-256-GCM-encrypted in `oauth_credentials`, consumed for the first time by Phase 4 Milestone 4 (§2.4). `SLACK_SIGNING_SECRET` is no longer vestigial — it is wired through `config/index.js` and used by Milestone 4's signature verification (§4.5). The second finding (shared-`x-api-key`-only routes on AIKB) remains open by deliberate scope decision, unaffected by Milestones 1–3 (Phase 4 §1 non-goals).

**High**
- **No DB-level tenant isolation (RLS) in either repo.** Both use the Supabase service-role key everywhere, bypassing any RLS that could exist; correctness depends entirely on every query author remembering `.eq('client_id', ...)`. Confirmed no `CREATE POLICY`/`ENABLE ROW LEVEL SECURITY` anywhere in either repo's SQL.
- **No knowledge-gap idempotency.** `aikb/services/supabaseService.js` `createKnowledgeGap` is a plain INSERT with no unique constraint or dedup check (`aikb/migrations/003_chat_history.sql`) — repeated Slack/portal reports of the same question create duplicate rows.
- **`searchChunksWithTitleBoost`'s second query is only transitively client-scoped**, not directly filtered by `client_id` in its own WHERE clause (`aikb/services/supabaseService.js:499`) — currently safe because the candidate document-id list is itself client-scoped upstream, but it's a defense-in-depth gap relative to the RPC path.

**Medium**
- **Dead/orphaned code inflating audit surface**: `Relativity/services/aikbService.js` `clearChatHistory` (unused — actual bulk-clear loops `deleteChatSession` instead), `aikb/middleware/resolveContext.js` `resolveContext` (permissive variant, never wired to a route), `aikb/services/googleDriveService.js` (would throw at runtime if invoked — references nonexistent `config.googleDrive`), `Relativity/services/dropboxService.js` `listFiles` (dead, references a route that no longer exists).
- **Duplicate/contradictory Slack env vars in Relativity**: `.env.example` lists both a used block (`SLACK_APP_ID`/`SLACK_APP_SECRET`/`SLACK_REDIRECT_URI`, read by `config/index.js:16-20`) and an unused block (`SLACK_CLIENT_ID`/`SLACK_CLIENT_SECRET`, a second `SLACK_REDIRECT_URI`) that no code reads.
- **Admin token comparison is not constant-time**: `Relativity/middleware/adminAuth.js:23` (`sig !== expected`) — timing side-channel on an HMAC comparison.

**Low**
- Two overlapping AIKB endpoints (`GET /summary/:clientId` and `GET /analytics/:clientId`) compute overlapping aggregates independently (`aikb/services/supabaseService.js:157-263`) with no shared computation or caching layer — not a correctness issue, just redundant queries.
- `TEST_BASE_URL`/`TEST_CLIENT_EMAIL`/etc. in Relativity's `.env.example` aren't referenced by any committed test file — vestigial.

## 7. Recommended platform boundary

- **Identity and tenancy** → **Relativity** (already sole owner of `clients`/`client_members`; AIKB should keep read-only access to the same Global project, not fork its own).
- **Integrations (OAuth, webhooks, provider API calls)** → **Relativity**, for consistency with the existing Google Drive/Dropbox pattern — Relativity is already the natural place for provider-specific glue, browser-facing consent screens, and per-client credential storage. AIKB should remain provider-agnostic and only receive normalized "ask a question on behalf of client X, origin Y" requests.
- **Permissions (roles, future collection ACLs)** → **Relativity** for the *role* axis (already there); the *collection/document* ACL axis should be modeled in **AIKB**, since AIKB is what actually executes retrieval and is the only place that can enforce it server-side inside the vector-search query itself (as it already does for `client_id` in `match_knowledge_chunks`).
- **Knowledge collections** → **AIKB**, for the same reason — access filtering has to happen at the SQL/RPC layer where retrieval executes, not in the calling gateway.
- **Ingestion, retrieval, answer generation, citations** → **AIKB** (already sole owner; keep it that way).
- **Knowledge gaps** → **AIKB** (already sole owner of storage/schema) — but Relativity's role as the *trigger* (deciding when to call `POST /gaps`) should be replaced by having AIKB itself decide and persist, using an idempotency key, rather than relying on a second network hop from a possibly-inconsistent caller.
- **Conversation state** → **AIKB** (already sole owner; correct, since it needs message history for retrieval-query rewriting anyway).
- **Background jobs** → **AIKB** (already sole owner; keep Inngest here — it already sits next to the data it's processing).
- **Analytics** → **AIKB** (already computes this from data it owns).
- **User-facing interfaces (portal, admin, future Slack/Teams/email UI surfaces)** → **Relativity**, as the customer-facing layer — but "interface" must mean *thin channel adapter*, not reimplementing retrieval/session logic per channel (the mistake the current AIKB Slack prototype makes).

## 8. Proposed cross-repository contract

*(Design recommendation only — not implemented.)*

- **Ask a question**: `POST /api/knowledge/query { clientId, question, sessionId?, origin: 'portal'|'slack'|'teams'|'email'|'api', originMetadata?, idempotencyKey }` → `{ answer, sources[], sessionId, isKnowledgeGap, gapReason? }`. Add `origin`/`originMetadata` now (currently missing) so gap records and audit trails can distinguish channels later.
- **Save or return a knowledge gap**: fold into `/query` itself — AIKB decides server-side whether to persist a gap (using `isKnowledgeGapAnswer`, which it already has) and returns the created `gapId`, keyed by an idempotency key derived from `(sessionId, question-hash)` so retries/Slack redeliveries don't duplicate. Retire the separate caller-triggered `POST /gaps` as the *only* path, or keep it strictly for explicit user "flag this as a gap" actions with its own idempotency key.
- **Retrieve citations**: already returned inline on `/query`; no separate endpoint needed.
- **Create/continue a conversation**: `sessionId` optional on `/query`; if omitted, AIKB creates one and returns it — this already works, keep it, but require `origin` to be set at session-creation time so a Slack thread and a portal session are distinguishable.
- **Apply authorized collection/document filters**: extend `/query` with `{ allowedCollections?: string[] }` resolved and signed by Relativity (since Relativity owns roles/identity) — AIKB enforces it server-side inside the retrieval RPC, never trusting an unsigned filter from a low-trust channel adapter like a Slack handler.
- **Idempotency keys**: add an `Idempotency-Key` header (or body field) honored on `/ingest`, `/gaps`, and any future Slack-event-triggered call — required because Slack Events API redelivers on timeout.
- **Identify source/origin**: every cross-repo call should carry `origin` + a caller-scoped credential (see §11 decision on per-channel API keys vs. one shared key) so AIKB can attribute and rate-limit by origin, not just by `clientId`.

## 9. Slack readiness assessment

| Item | Status |
|---|---|
| OAuth (per-client bot install) | **Partially ready** — flow exists and works in Relativity, but the resulting token is never consumed anywhere |
| Events endpoint | **Partially ready** — exists in AIKB with real signature verification, but bypasses proper tenant mapping |
| Durable processing | **Missing** — Slack events are handled synchronously in `routes/slack.js`, not via Inngest; no retry/redelivery-safe queueing |
| Tenant mapping (Slack team/channel → client_id) | **Blocked** — currently a hash function, not a real mapping table; no schema exists anywhere for this |
| User identity mapping (Slack user → company identity) | **Missing** — not found in either repo |
| Permissions | **Missing** — no collections/ACL model exists to enforce what a Slack channel should be allowed to retrieve |
| Retrieval filters (beyond client_id) | **Missing** |
| Thread continuity | **Missing** — AIKB's Slack handler doesn't use the session system at all |
| Direct messages | **Missing** — not handled by the current `app_mention`-only handler |
| Knowledge gaps (from Slack) | **Missing** — AIKB's Slack handler never calls `createKnowledgeGap` |
| Audit records | **Missing** — no Slack-specific audit metadata anywhere |
| Token encryption | **Missing** — Relativity's captured token is plaintext; AIKB's bot token is a plain env var (acceptable for a single static token, but irrelevant since it's disconnected from the OAuth flow anyway) |
| Tests | **Missing** — no automated tests touch Slack in either repo |

> **Historical finding — status update added after Phase 4 Milestones 1–3 (see Phase 4 below for full detail; this table is preserved as-written, not edited, per the report's own historicity rule):** "OAuth (per-client bot install)" is resolved — Milestone 3 shipped a working, encrypted, org-bound OAuth flow whose token is consumed by Milestone 4. "Events endpoint" and "Tenant mapping" are resolved differently than this table anticipated — AIKB's Events endpoint was retired (410 Gone, Milestone 1) rather than fixed in place, and tenant mapping now lives in Relativity's `oauth_connections` table (Milestone 2), not as a fix to AIKB's hash function. "Token encryption" is resolved — Milestone 2 added AES-256-GCM envelope encryption. "Durable processing," "Permissions," "Retrieval filters," "Thread continuity," "Direct messages," "Knowledge gaps (from Slack)," "Audit records," and "Tests" remain open and are addressed (or explicitly deferred) by Phase 4 Milestone 4 and its adjacent Milestones 5–7.

## 10. Prerequisites before Slack coding

**Blocking prerequisites**
1. Decide and implement one real `slack_team_id`/`slack_channel_id` → `client_id` mapping table (owner: Relativity, per §7), replacing the hash-derivation in `aikb/routes/slack.js`.
2. Decide which repo owns Slack Events ingestion long-term and delete/rebuild the other half — the current split-brain (OAuth in Relativity, Events+bot token in AIKB) cannot ship safely as-is.
3. Move Slack signature verification and event receipt to a durable, idempotent path (Inngest event, not synchronous handling) to survive Slack's redelivery-on-timeout behavior.
4. Encrypt stored OAuth tokens (`oauth_tokens.access_token`) before they're used for anything beyond the current dead-end.
5. Close the shared-`API_KEY`-only tenant gap on AIKB's ingest/list/delete routes (§6 Critical) before any Slack-triggered ingestion or deletion path is added.
6. Design and implement at least a minimal collections/scope model in AIKB (even if just a `collection` column + retrieval filter) before exposing any channel-based surface, since channel-based surfaces are exactly where over-broad retrieval is most damaging.
7. Add an idempotency key to knowledge-gap creation and to any future Slack-event-triggered ingest/query calls.

> **Historical finding — status update added after Phase 4 Milestones 1–3:** items 1 (tenant mapping table), 2 (Events split-brain), and 4 (token encryption) are resolved — Milestone 1 retired AIKB's Events handler, Milestone 2 built the encrypted `oauth_connections`/`oauth_credentials` tables, and Milestone 3 wired the real OAuth flow into them. Item 3 (durable, idempotent event receipt) and item 7 (idempotency keys) are addressed by Phase 4 Milestone 4 (§2.4 §4.7–§4.8). Items 5 (shared-`x-api-key` tenant gap on AIKB's non-Slack routes) and 6 (collections/scope model) remain open by deliberate scope decision — see Phase 4 §1 non-goals and §2.4 §4.9/§2.5.

**Can wait until Phase 2**
- User-identity mapping (Slack user → company identity) — needed for per-user auditing/permissions but not for a minimal read-only Q&A bot scoped at the channel level.
- Direct-message support.
- Slack-specific audit metadata beyond basic logging.

**Optional future improvements**
- Reranking layer.
- Materialized/cached analytics instead of on-the-fly aggregation.
- Removing dead code identified in §6 Medium/Low.

## 11. Architecture decisions requiring my approval

1. **Which repo owns Slack Events ingestion going forward?**
   - Option A: Move Events webhook + signature verification into Relativity (consistent with it owning all other integrations/OAuth), have Relativity call AIKB's existing `/query` contract.
   - Option B: Keep Events ingestion in AIKB (already built, already signature-verified) but fix the tenant-mapping and bring it in line with `/query`'s existing session/gap logic instead of bypassing it.
   - **Recommendation**: Option A — for architectural consistency (Relativity = integration layer everywhere else) and because AIKB should stay a provider-agnostic knowledge engine.
   - **Tradeoff**: Option A requires re-porting the already-working signature-verification code from AIKB to Relativity; Option B is less work now but perpetuates AIKB as a channel-aware system, which cuts against the stated long-term platform boundary.

2. **Per-client vs. shared API key between Relativity and AIKB.**
   - Option A: Keep one shared `API_KEY` (current state), and instead add a signed, per-request client-authorization token from Relativity (e.g., short-lived JWT asserting `{clientId, origin}`).
   - Option B: Issue AIKB a proper service-identity check on every route (mirroring what `requireMemberContext` already does), removing the current blanket trust in the shared secret.
   - **Recommendation**: Option B, since AIKB already has the machinery (`requireMemberContext`) — extend it (or an origin-aware equivalent) to the currently-unprotected ingest/list/delete routes.
   - **Tradeoff**: Option B requires every current caller (Relativity's server-to-server routes) to carry forward a real caller identity, which today they don't always have (e.g., internal cleanup calls during client deletion) — needs a service-identity concept, not just user JWTs.

3. **Where should the collections/ACL model live?**
   - Option A: Relativity owns collection definitions/membership; AIKB enforces them at retrieval time via a signed filter passed on `/query`.
   - Option B: AIKB owns collections end-to-end (its own table, its own admin API), Relativity only displays them.
   - **Recommendation**: Option A — collection *definition and assignment* is a tenancy/permissions concern (Relativity's domain), but *enforcement* must happen in AIKB's retrieval SQL, which is why the filter needs to be signed rather than freely passed.
   - **Tradeoff**: Option A needs a signing/trust mechanism between the two repos; Option B is simpler to build but pulls a tenancy concept into the knowledge engine, muddying the boundary described in §7.

4. **What happens to the existing dead-end Slack OAuth token?**
   - Option A: Discard and rebuild the whole Slack OAuth+bot flow cleanly under whichever repo Decision 1 selects.
   - Option B: Wire the existing captured token into a real `chat.postMessage` call and keep the current OAuth flow as-is, layering fixes on top.
   - **Recommendation**: Option A, given how small `slackService.js` is (27 lines) and how much else needs to change (signing, encryption, mapping) — not worth preserving.
   - **Tradeoff**: Option A throws away working OAuth-exchange code that could be reused nearly as-is if Option B were chosen instead.

## 12. Suggested Phase 2

Once the four decisions above are made, Phase 2 should be a **design-only** pass (still no implementation) that: (a) specifies the exact `slack_channel_mapping`/collections schema and where it lives, (b) specifies the signed-filter/service-identity contract between Relativity and AIKB in full request/response detail, and (c) produces a concrete, ordered implementation plan for the blocking prerequisites in §10 — at which point a Phase 3 implementation plan (with your approval) can begin actual coding.

---

# Phase 2: Relativity Platform Architecture — Technical Blueprint

**Status:** Design specification only. No code, migrations, or repository changes were made in producing this document.
**Source of truth:** Phase 1 Architecture Discovery report (above).
**Scope:** Long-term platform architecture for Relativity + AIKB, designed interface-first (Slack is the first consumer, not the design driver).

## 0. Design philosophy

Phase 1 found a boundary that is *mostly* right but violated in one place that matters most: AIKB's `routes/slack.js` is a second, parallel "surface" implementation that duplicates identity resolution, retrieval, and answer generation outside the platform's actual knowledge pipeline. Phase 2 treats that as the central lesson: **every interface must be a thin adapter over one shared pipeline.** Nothing about Slack is special enough to justify its own retrieval or session logic — if it needs its own logic today, that's a gap in the shared abstraction, not a reason to special-case Slack.

Two assumptions from Phase 1 are revised here (flagged inline where they occur):
- **Collections**: Phase 1 recommended Relativity own collection *definition*, AIKB enforce via a signed filter. This document splits it further — collection *entitlement* (who can access what) stays in Relativity; collection *assignment* (which documents belong to which collection) moves to AIKB, because that data is inseparable from the documents it describes.
- **Background execution ownership**: Phase 1 framed "who owns Slack event processing" as a binary choice between repos. This document reframes it as a split between *provider-specific edge* (always Relativity) and *provider-agnostic core* (always AIKB) — see §9.

## 1. Platform architecture

```
                                   PUBLIC INTERNET
   ┌──────────┬──────────┬───────────┬───────────┬───────────┬─────────────┐
   │  Browser │  Slack   │  Teams    │  Gmail     │  Outlook  │  Public API │
   │ (Portal) │ Events/  │  Events/  │  Push/     │  Webhook/ │  Clients    │
   │          │ OAuth    │  OAuth    │  OAuth     │  OAuth    │             │
   └────┬─────┴────┬─────┴─────┬─────┴─────┬─────┴─────┬─────┴──────┬──────┘
        │          │           │           │           │            │
════════│══════════│═══════════│═══════════│═══════════│════════════│═══════  TB1
        ▼          ▼           ▼           ▼           ▼            ▼
   ┌────────────────────────────────────────────────────────────────────┐
   │                    RELATIVITY  (Vercel, Node/Express)               │
   │                                                                      │
   │   ┌────────────────────────────────────────────────────────────┐   │
   │   │  SURFACE ADAPTERS  (one per interface, implement the        │   │
   │   │  Knowledge Surface contract — §4)                            │   │
   │   │  PortalSurface │ SlackSurface │ TeamsSurface │ GmailSurface  │   │
   │   │  OutlookSurface │ ApiSurface                                 │   │
   │   └───────────────────────────┬────────────────────────────────┘   │
   │ ═══════════════════════════════│═══════════════════════════════ TB2│
   │   ┌───────────────────────────▼────────────────────────────────┐   │
   │   │  RELATIVITY CORE                                             │   │
   │   │  Identity & Org Service │ Authorization Service │            │   │
   │   │  OAuth Platform (§6)    │ Integration Orchestrator │         │   │
   │   │  Delivery Router (provider-specific reply formatting/send)  │   │
   │   └───────────────────────────┬────────────────────────────────┘   │
   │                               │                                     │
   │   Global Supabase (Relativity-owned DB): orgs, users, team_members, │
   │   roles, groups, identity_links, oauth_connections, surface_       │
   │   conversation_links, audit_log                                    │
   └───────────────────────────────┼─────────────────────────────────────┘
                                    │
   ═════════════════════════════════│═══════════════════════════════ TB3
                                    ▼  signed ServiceRequest{org, principal, collections[]}
   ┌────────────────────────────────────────────────────────────────────┐
   │                       AIKB  (Railway, Node/Express)                 │
   │                                                                      │
   │   ┌────────────────────────────────────────────────────────────┐   │
   │   │  KNOWLEDGE API: ask, continueConversation, resolveGaps,     │   │
   │   │  retrieveSources, resolveAuthorizedCollections, health       │   │
   │   └───────────────────────────┬────────────────────────────────┘   │
   │   ┌───────────────────────────▼────────────────────────────────┐   │
   │   │  RETRIEVAL & REASONING: collection-filtered vector search,   │   │
   │   │  prompt construction, LLM call, citation builder              │   │
   │   └───────────────────────────┬────────────────────────────────┘   │
   │   ┌───────────────────────────▼────────────────────────────────┐   │
   │   │  DURABLE EXECUTION (Inngest, in-process): ingestion,          │   │
   │   │  reindex, delete, async query fan-out, delivery callback      │   │
   │   └───────────────────────────┬────────────────────────────────┘   │
   │                               │                                     │
   │   AIKB Supabase (AIKB-owned DB): knowledge_documents, chunks,       │
   │   collections, document_collections, conversations, messages,       │
   │   knowledge_gaps                                                    │
   └───────────────────────────────┼─────────────────────────────────────┘
   ═════════════════════════════════│═══════════════════════════════ TB4
                                    ▼
                         ┌────────────────────┐
                         │ OpenAI (or future   │
                         │ LLM/embedding       │
                         │ provider)            │
                         └────────────────────┘
```

### Trust boundaries

| # | Boundary | Crossing rule |
|---|---|---|
| TB1 | Public internet → Surface Adapter | Every inbound request must be authenticated at the protocol level appropriate to that surface (Slack signature, Teams bot token, Gmail/Outlook OAuth push validation, Portal Supabase JWT, Public API key) before touching any shared code. |
| TB2 | Surface Adapter → Relativity Core | Adapters must have fully normalized the caller into an `AuthorizationContext` (org, principal, roles) before calling Core. Core never parses provider-specific payloads. |
| TB3 | Relativity Core → AIKB | Every call carries a signed **ServiceRequest** (§5, §6 of Phase 1 decision 2 resolved as Option B): org id, resolved principal, resolved collection ids, origin, idempotency key. AIKB trusts this signature, never a raw client-supplied `clientId` or a static shared key alone. |
| TB4 | AIKB → LLM provider | Only chunks already filtered by the collection/tenant boundary may enter a prompt. This is the platform's actual data-leak boundary — enforced once, in one place (§3, §10). |
| Internal | Relativity ↔ Global Supabase, AIKB ↔ AIKB Supabase | Service-role key, same as today; no change recommended (RLS as defense-in-depth is a **Should Have**, §10). |
| Internal | Relativity ↔ OAuth Providers (token exchange) | External, secrets involved. |
| Internal | AIKB Inngest → Relativity Delivery Router | Async answer delivery calls back into Relativity so that provider-specific "send" logic (chat.postMessage, Graph API, SMTP reply) stays out of AIKB. Authenticated the same way as TB3, reversed. |

### Request flow (generic, applies to every surface)

```
1. External event arrives at Surface Adapter (TB1)
2. Adapter: authenticate()      -> verify protocol-level signature/token
3. Adapter: identifyUser()      -> map external identity to a Principal (§2)
4. Core:    authorize()         -> resolve org, role, entitled collections (§3)
5. Core:    queryKnowledge()    -> signed ServiceRequest to AIKB (TB3)
6. AIKB:    retrieve + reason   -> collection-filtered search, LLM, citations
7. AIKB:    persist             -> conversation, message, gap-if-any
8. Core:    formatAnswer()      -> surface-specific rendering
9. Adapter: deliverAnswer()     -> provider API call (sync or async via callback)
10. Core:   recordAudit()       -> audit_log entry, every step
```

## 2. Identity model

**Revised ownership:** all identity lives in Relativity. AIKB receives a resolved, signed context per request and persists nothing about *who* a person is beyond an opaque `principalId` for scoping conversations/gaps.

```
Organization
   │
   ├── User (has Supabase Auth account)
   │      └── TeamMember (org-scoped: role, group memberships, status)
   │
   ├── Group  (e.g. "Sales", "Engineering" — maps to Collections, §3)
   │
   ├── ServiceAccount (machine identity: Public API callers, internal jobs)
   │
   └── IdentityLink (external identity → Principal)
          ├── SlackIdentity   (team_id + user_id → Principal | Guest)
          ├── TeamsIdentity   (tenant_id + aad_object_id → Principal | Guest)
          ├── EmailIdentity   (verified sender address → Principal | Guest)
          └── ExternalUser / GuestUser (unlinked external identity, default-deny)
```

| Entity | Owner | Definition |
|---|---|---|
| **Organization** | Relativity | The tenant. Replaces/renames today's `clients` table 1:1 — no schema change in spirit, just the platform-wide term. |
| **User** | Relativity | A Supabase Auth account. Can belong to multiple Organizations in the future (not required now — Must Have: single-org; Should Have: multi-org). |
| **Team Member** | Relativity | User × Organization join, carrying `role` and `status` — same as today's `client_members`. |
| **Role** | Relativity | Coarse permission tier (owner/admin/member/viewer) — unchanged from Phase 1, kept as-is. |
| **Permission** | Relativity | New: a named capability (`upload_documents`, `manage_integrations`, `view_restricted_collections`) — Should Have. Roles are bundles of default Permissions; Permissions let an org override a single capability without inventing a new role. |
| **Group** | Relativity | New: a named set of TeamMembers used purely for **collection entitlement** (§3) — e.g. "Sales", "Legal". Independent of Role (a viewer can be in the Legal group and see Legal-restricted answers; an admin isn't automatically in every group). |
| **Slack User / Teams User / External User** | Relativity, via `IdentityLink` | An external identity is *never* itself a Principal — it must resolve to a `TeamMember` (via IdentityLink) or fall through to Guest. |
| **Guest User** | Relativity | A resolvable-but-unlinked external identity (e.g. a Slack user in the workspace who hasn't been matched to a TeamMember, or a Slack Connect external participant). Guests get an org-configurable default entitlement — Must Have default: **no collections beyond "General"**, no knowledge-gap visibility, always audited distinctly from linked users. |
| **Service Account** | Relativity | Machine identity for Public API callers and internal jobs. Has its own scoped Role/Permission set, never a "role" like a human — Must Have for a real Public API (§11). |

**Why this stays entirely in Relativity:** AIKB's job is to answer questions inside a boundary someone else has already drawn. Giving AIKB its own identity table (even partial) is exactly the drift that produced the disconnected Slack bot in Phase 1 — a second place where "who is allowed to see what" gets decided.

## 3. Knowledge authorization

**Revised split (see §0):**

| Concept | Owner | Why |
|---|---|---|
| Collection **definition** (name, description, restriction level) | Relativity | It's an authorization construct, configured by org admins alongside roles/groups. |
| Collection **entitlement** (which Groups/Roles can access which Collections) | Relativity | Same reasoning — this is who-can-see-what, Relativity's domain end to end. |
| Collection **assignment** (which documents/chunks belong to which Collection) | AIKB | Documents and their chunks live in AIKB; tagging them is inseparable from ingestion, which AIKB already owns. |
| Collection **enforcement** (filtering retrieval) | AIKB | Must happen at the SQL/RPC layer where retrieval executes — the only place a filter can't be bypassed by a compromised or buggy caller. |

```
Organization
  └── Collection (e.g. "General", "Sales", "Engineering", "HR",
                   "Finance", "Legal", "Executive", custom)
        ├── restrictionLevel: open | department | restricted
        ├── inheritsFrom: Collection?   (e.g. "Sales-EMEA" inherits "Sales")
        └── entitledGroups: Group[]     (Relativity-side mapping)

  Document (AIKB) ──belongs to──▶ one or more Collections (AIKB-side tags)
```

- **Inheritance**: a Collection may declare a parent; entitlement to the parent implies entitlement to the child unless explicitly revoked (deny-override, not allow-override — Must Have, since silent widening of access is the dangerous failure mode). Depth capped at 2 levels to avoid unreviewable inheritance chains — Should Have guardrail.
- **Restricted collections**: `restrictionLevel: restricted` collections are never included in a Guest's or unlinked-identity's entitlement set, full stop, regardless of Group mapping — Must Have.
- **Department collections**: modeled as ordinary Collections whose entitled Group matches a department Group; no special-cased "department" type needed.
- **Custom/future collections**: since Collection is just `{id, org, name, restrictionLevel, parent?}` + entitlement mapping, no schema change is needed to add new ones — org admins create them at runtime. This satisfies design objective 7 (every integration reuses the same abstraction) directly.

**How authorization reaches AIKB:** Relativity resolves `entitledCollectionIds` for the current Principal at request time (`resolveAuthorizedCollections`, §5) and includes it, **signed**, in the `ServiceRequest` sent to AIKB. AIKB's retrieval RPC filters on `client_id AND collection_id IN (:entitledCollectionIds)` in the same query that already enforces `client_id` today (`match_knowledge_chunks`) — extending a mechanism that's proven to work, not inventing a parallel one. AIKB never trusts an unsigned collection list from a caller (this is what makes the AIKB Slack prototype's channel-hash approach unsafe — it had no signed authorization at all).

This directly satisfies design objective 4: **AIKB never receives more knowledge than the user is allowed to retrieve**, because the filter is applied inside the same SQL statement that does the vector search — there is no code path that returns chunks before the filter runs.

## 4. Knowledge Surface abstraction

Every interface (Portal, Slack, Teams, Gmail, Outlook, Public API, and anything future) implements the same contract. This is the direct fix for the Phase 1 critical finding — no surface is allowed to run its own retrieval or identity resolution.

| Method | Responsibility | Executes in | Notes |
|---|---|---|---|
| `authenticate(rawRequest)` | Verify the request really came from the provider (signature, OAuth bearer, bot token) | Surface Adapter | Protocol-specific; this is the *only* method allowed to know provider wire-format details. |
| `identifyUser(rawRequest)` | Resolve external identity → `Principal` via `IdentityLink`, or `Guest` | Surface Adapter, backed by Relativity Core's Identity Service | Never invents an identity; always looks up or falls through to Guest. |
| `receiveQuestion(rawRequest)` | Extract the natural-language question + any thread/reply context | Surface Adapter | Normalizes into `{text, externalThreadId?}`. |
| `authorize(principal, org)` | Resolve entitled collections | Relativity Core (shared, not per-surface) | Identical call for every surface — this is what makes objective 1 (new interface = little work) true. |
| `queryKnowledge(question, principal, collections, conversationRef)` | Call AIKB's `ask`/`continueConversation` contract (§5) | Relativity Core (shared) | Every surface calls the exact same function. |
| `formatAnswer(answer, citations)` | Render for the target medium (Slack blocks, HTML email, Adaptive Card, plain JSON) | Surface Adapter | Only place formatting logic lives. |
| `deliverAnswer(formatted, destination)` | Send the reply via the provider's API | Surface Adapter, using OAuth Platform (§6) for credentials | May be synchronous (Portal, Public API) or async via Delivery Router callback (Slack/Teams/email, §9). |
| `recordAudit(event)` | Write a structured audit entry | Relativity Core (shared) | Every method above emits one; not something each surface reimplements. |

**Design rule (Must Have):** steps `authorize` and `queryKnowledge` are **never** implemented per-surface. If a surface's SDK makes this hard (e.g., needs a synchronous ack), that's solved in the delivery/background layer (§9), not by duplicating the pipeline — this is precisely the mistake found in AIKB's `routes/slack.js`.

**New surface checklist** (this is what "should require very little work" cashes out to): implement `authenticate`, `identifyUser`, `receiveQuestion`, `formatAnswer`, `deliverAnswer` for the new provider; register an OAuth provider config if needed (§6); everything else is inherited.

## 5. Cross-repository contract

All requests from Relativity to AIKB carry a common envelope:

```
ServiceRequest envelope (all endpoints):
  organizationId
  principal: { id, type: user|guest|service_account, displayName? }
  origin: portal | slack | teams | gmail | outlook | api
  originMetadata: { externalThreadId?, externalMessageId? }
  entitledCollectionIds: [ ... ]        // signed by Relativity, §3
  idempotencyKey
  signature                              // service-identity signature, TB3
```

| Endpoint | Purpose | Key request fields | Key response fields |
|---|---|---|---|
| `POST /ask` | New question, possibly starting a conversation | `question`, `conversationId?` | `answer`, `citations[]`, `conversationId`, `isKnowledgeGap`, `gapId?` |
| `POST /conversations/{id}/continue` | Follow-up within existing thread | `question` | same shape as `/ask` |
| `POST /gaps` | Explicit user-flagged gap (distinct from auto-detected) | `conversationId?`, `question`, `reason` | `gapId` |
| `GET /sources/{answerId}` | Re-fetch full citation detail (e.g., for a "show sources" UI action after the fact) | — | `citations[]` with document/page detail |
| `POST /resolve-collections` | Relativity asks AIKB to validate a collection-id list still exists/is active before signing it (defense-in-depth cache-busting) | `collectionIds[]` | `validCollectionIds[]` |
| `GET /health` | Liveness/readiness | — | `status`, `dependencies: {db, llm, embeddings}` |

**Idempotency (Must Have):** `idempotencyKey` on `/ask` and `/gaps` is required, not optional (Phase 1 found none exists today). Derived by the Surface Adapter from the provider's own delivery-dedup primitive where one exists (Slack `event_id`, Gmail `historyId`+`messageId`), or a UUID for Portal.

**Auto gap detection (revised from Phase 1):** AIKB decides and persists gaps server-side inside `/ask`/`/continue` using its existing `isKnowledgeGapAnswer` logic — no second network hop required for the common case. `/gaps` remains only for explicit "flag this as wrong/missing" user actions, which is semantically different (user disagreement vs. no-context detection) and should stay a distinct signal in the gap record (`reportedBy: system | user`).

## 6. OAuth platform

One framework, one set of tables, provider differences pushed into config + a small adapter, not duplicated flow code.

```
oauth_providers            (registry, one row per provider type)
  id, name, authUrl, tokenUrl, scopes[], authStyle (basic|body), metadataSchema

oauth_connections           (one row per org × provider, replaces oauth_tokens)
  id, organizationId, providerId, status (connected|expired|revoked),
  externalAccountId, scopesGranted[], connectedByPrincipalId, createdAt

oauth_tokens                (envelope-encrypted, 1:1 or 1:N with connection
                              for refresh-token rotation history)
  id, connectionId, accessTokenEncrypted, refreshTokenEncrypted,
  expiresAt, encryptionKeyVersion
```

| Concern | Design |
|---|---|
| **Token storage** | Split from `oauth_connections` metadata into `oauth_tokens` so access patterns differ (metadata read often, tokens read only at send-time) — Must Have, also closes Phase 1's plaintext-token finding by construction. |
| **Encryption** | Envelope encryption: a KMS-managed root key encrypts per-org data keys, which encrypt tokens (`encryptionKeyVersion` supports rotation without re-touching every row at once). Must Have before any token is used for anything beyond capture (Phase 1 blocking prerequisite #4). |
| **Refresh** | A single scheduled Inngest function (AIKB or Relativity — recommend **Relativity**, since it owns OAuth end-to-end) sweeps `oauth_connections` nearing `expiresAt` and refreshes via the provider's `tokenUrl`. One implementation for all providers, not one per integration. |
| **Revocation** | `deleteClientFull`-style cascading delete (already exists in spirit, Phase 1 §6/§9) extended to call each connected provider's revoke endpoint before deleting local rows — Should Have; Must Have is at minimum deleting local tokens on disconnect. |
| **Provider metadata** | `oauth_providers.metadataSchema` lets provider-specific fields (Slack's `incoming_webhook`, Teams' tenant id) live in a typed JSONB column on `oauth_connections` rather than one-off columns like today's `clients.slack_channel_id`. |
| **Secret management** | Provider `clientId`/`clientSecret` move out of flat env vars into a secrets manager reference (Should Have — Must Have is at minimum consolidating the current duplicate/dead env vars found in Phase 1 §6). |
| **Callback pattern** | Uniform route shape for every provider: `GET /oauth/:providerId/start`, `GET /oauth/:providerId/callback` — replaces the current one-off route per provider (`/auth/slack/start`, `/auth/google/start`, ...). |

## 7. Conversation model

AIKB remains the system of record for conversation *content* (it needs history for retrieval-context rewriting, per Phase 1 §11); Relativity keeps only the *external-thread mapping*, generalized from today's `client_member_sessions`.

```
AIKB:
  conversation
    id, organizationId, principalId, origin, originThreadId,
    title, createdAt, lastActivityAt, retentionPolicy, deletedAt

  message
    id, conversationId, role, content, sources[], metadata,
    createdAt, deletedAt

Relativity:
  surface_conversation_link
    id, organizationId, surface, externalThreadId, conversationId,
    createdAt
```

| Conversation type | `origin` | `originThreadId` | Notes |
|---|---|---|---|
| Portal | `portal` | null | One conversation per browser session start, as today. |
| Slack thread | `slack` | Slack `thread_ts` | New message in same thread → `continue`, resolved via `surface_conversation_link`. |
| Slack DM | `slack` | Slack DM channel id | Same mechanism, no special case. |
| Teams thread | `teams` | Teams conversation id | Same mechanism. |
| Email | `gmail` / `outlook` | RFC `Message-ID` / thread id | Same mechanism; a reply-chain is a thread. |

**Required metadata on every conversation (Must Have):** `organizationId`, `principalId` (or `guestId`), `origin`, `originThreadId` (nullable for Portal/API), `entitledCollectionIdsAtCreation` (snapshot — so later collection changes don't retroactively alter what a historical answer *implies* it saw), `retentionPolicy` (org-configurable; email/Slack often need shorter retention than a Portal knowledge base).

## 8. Knowledge gap architecture

One `knowledge_gaps` table (AIKB-owned, per Phase 1 recommendation — kept), extended:

| Field | Purpose |
|---|---|
| `organizationId`, `conversationId`, `messageId` | unchanged in spirit from today |
| `origin` | **new** — portal / slack / teams / gmail / outlook / api |
| `originMetadata` | **new** — JSONB, e.g. Slack channel/thread for triage routing |
| `idempotencyKey` | **new**, unique constraint — `hash(organizationId, conversationId, normalizedQuestion)`; closes Phase 1's no-dedup finding |
| `reportedBy` | **new** — `system` (auto-detected no-context) vs `user` (explicit flag) |
| `status` | open / reviewed / resolved / ignored — unchanged |
| `createdAt`, `resolvedAt`, `resolvedByPrincipalId` | audit trail |

- **Every source feeds it through the same path**: gap creation happens exclusively inside AIKB's `/ask`/`/continue` handling (system-reported) or `/gaps` (user-reported) — never inside a Surface Adapter. This is what "one universal gap model" requires: no surface is allowed to write directly to `knowledge_gaps`.
- **Idempotency**: unique constraint on `idempotencyKey` turns duplicate detection into a DB-level guarantee instead of app-level discipline.
- **Analytics**: rollups (`totalGaps`, `gapsByOrigin`, `gapsByCollection`) computed from this one table — no per-surface gap counters to reconcile.

## 9. Background execution

**Revised framing (see §0):** rather than asking "does event processing live in Relativity or AIKB," split by whether the code is provider-specific.

```
Surface Adapter (Relativity)              AIKB
──────────────────────────────            ─────────────────────────────
1. authenticate + verify signature
2. immediate ack to provider (<3s)
3. emit SurfaceEvent{idempotencyKey, ...} ──▶ 4. Inngest enqueues
                                               knowledge.query.requested
                                           5. authorize() already resolved
                                              by step 3's payload (signed)
                                           6. retrieval + LLM + persist
                                           7. Inngest emits
                                              knowledge.answer.ready
8. Delivery Router receives callback     ◀── (webhook back to Relativity)
9. formatAnswer + deliverAnswer
   (provider API call, uses OAuth token)
```

- **Should the `Relativity → AIKB → Inngest` shape remain?** Yes, with one addition: Relativity needs a **durable outbox** for step 3, because Vercel serverless functions can't guarantee a background retry if the call to AIKB fails mid-request. Recommend a minimal `outbound_events` table in Relativity's Global DB, written synchronously in the same request that acks the provider, drained by a scheduled sweep — this is a small, generic mechanism, not a second job queue (Must Have; without it, a transient AIKB outage silently drops a Slack question with no retry).
- **Retries**: Inngest's existing per-function `retries: 3` (Phase 1 §7) stays the mechanism inside AIKB. The new outbox handles the Relativity→AIKB leg specifically.
- **Event naming (Must Have convention):** `domain.entity.action`, past tense for completed facts, present-imperative for requests: `surface.event.received`, `knowledge.query.requested`, `knowledge.answer.ready`, `knowledge.gap.created`, `surface.delivery.requested`, `surface.delivery.confirmed`, `oauth.token.refreshed`. Consistent naming is what lets a future surface reuse the same event vocabulary instead of inventing `slack.message.answered` one-offs.
- **Why not move Inngest into Relativity?** Vercel's serverless model can't host a long-running durable-execution engine the way Railway's always-on process can; keeping Inngest where the data and retrieval logic already live avoids a third deployment target.

## 10. Security architecture

| Concern | Design | Priority |
|---|---|---|
| Tenant isolation | Every AIKB query filters `organizationId` inside the SQL/RPC itself (already true for vector search today — Phase 1 §11); extend the same pattern to the currently-shared-key-only routes (list/ingest/delete) by requiring the signed `ServiceRequest` envelope everywhere, not just on `/ask` | **Must Have** |
| Authorization | Resolved once in Relativity Core (`authorize()`), signed, never re-derived by a Surface Adapter or by AIKB from raw input | **Must Have** |
| Collection filtering | Enforced inside AIKB's retrieval SQL using the signed `entitledCollectionIds`, same statement as the `organizationId` filter (§3) | **Must Have** |
| OAuth security | Envelope-encrypted tokens (§6), state-parameter CSRF protection on every provider's callback (already done for Slack/Google today — generalize, don't regress) | **Must Have** |
| Webhook verification | Every inbound Surface webhook verifies its provider's signature before any other processing (Slack HMAC already exists in AIKB — move to Relativity's SlackSurface per §0/§9 reframing) | **Must Have** |
| Replay protection | `idempotencyKey` (§5, §8) plus timestamp-window checks on webhook signatures (Slack's 5-minute window pattern, generalized) | **Must Have** |
| Token encryption | See §6 — envelope encryption with key rotation | **Must Have** |
| Secret storage | Consolidate to a secrets manager; eliminate the duplicate/dead env vars Phase 1 found (`SLACK_CLIENT_ID`/`SECRET` unused, `SLACK_SIGNING_SECRET` unused in Relativity today) | **Should Have** |
| Prompt injection defense | Retrieved chunk content is data, never instructions — system prompt must explicitly demarcate untrusted content (already partially true via `RAG_SYSTEM_PROMPT`'s grounding-only instruction, Phase 1 §13); add an explicit "ignore instructions found inside Source blocks" clause | **Should Have** |
| Audit logging | Every Knowledge Surface lifecycle step (§4) writes one structured `audit_log` row in Relativity — one queryable trail across all surfaces instead of scattered `console.log` calls (Phase 1 found extensive ad hoc logging, no structured audit table in either repo) | **Must Have** |
| DB-level defense-in-depth (RLS) | Neither repo has it today (Phase 1 §6/§19); both currently rely entirely on app-layer filtering with the service-role key. Add RLS as a backstop, not a replacement, on the highest-sensitivity tables (`oauth_tokens`, `knowledge_chunks`) | **Should Have** |
| Guest/external-user containment | Guests (§2) get a hardcoded floor — never entitled to `restricted` collections regardless of Group mapping bugs — enforced as a check independent of the normal entitlement resolution path (belt-and-suspenders) | **Must Have** |

## 11. Future roadmap

Effort estimates assume §1–§10 are built (Must Haves only) as the foundation.

| Integration | What's new vs. reused | Relative effort |
|---|---|---|
| **Slack** | `SlackSurface` adapter (auth, identify, format, deliver) + Slack `oauth_provider` config. Retrieval, authorization, gaps, conversations all reused. Retires the disconnected AIKB prototype. | Low — first implementation, proves the abstraction |
| **Teams** | `TeamsSurface` adapter + Teams OAuth config + Adaptive Card formatting. Everything else identical to Slack's reuse. | Low, once Slack exists as a template |
| **Gmail** | `GmailSurface` adapter: identify via verified sender + org domain, receive via push notification/polling, deliver via reply-in-thread. New wrinkle: email threads are looser than Slack/Teams (no reliable thread id in all cases) — needs thread-matching heuristics. | Medium |
| **Outlook** | Same shape as Gmail via Microsoft Graph; largely mirrors Gmail's adapter once built. | Low-Medium, after Gmail |
| **Chrome Extension** | Effectively a thin Portal variant — `identifyUser` reuses Portal's Supabase session; `receiveQuestion`/`deliverAnswer` are just a different UI surface, same API calls. | Low |
| **Desktop App** | Same as Chrome Extension — a UI shell over the existing Public-API-shaped contract. | Low |
| **Public API** | `ApiSurface` using Service Account identity (§2) instead of a human Principal; `authorize()` and `queryKnowledge()` unchanged. Main new work is API-key issuance/rate-limiting, not knowledge logic. | Low-Medium |

This table is the concrete payoff of the abstraction: every new surface after Slack is bounded to "write an adapter," never "extend the knowledge pipeline."

## 12. Phase 2 summary: Must Have / Should Have / Future

| Priority | Items |
|---|---|
| **Must Have** (blocking, before Slack) | Signed `ServiceRequest` envelope replacing the shared API key (§1 TB3, §10); Collection entitlement + enforcement split (§3); Knowledge Surface contract with shared `authorize`/`queryKnowledge` (§4); Idempotency keys on `/ask` and `/gaps` (§5, §8); OAuth token envelope encryption (§6); Relativity outbox for durable event handoff (§9); Structured audit logging (§10); Guest containment floor (§10) |
| **Should Have** (soon after) | Permission entity beyond coarse Roles (§2); OAuth revocation calling provider APIs (§6); Secrets-manager consolidation (§10); RLS as defense-in-depth (§10); Prompt-injection demarcation hardening (§10) |
| **Future** | Multi-org Users (§2); custom collection inheritance depth review as usage grows (§3); Chrome Extension / Desktop App (§11) |

This document is a design specification. No implementation plan, migration, or code follows from it automatically — Phase 3 (implementation planning) requires explicit approval to begin, per the Phase 1 decisions this document builds on.

---

# Phase 3: Relativity Systems Domain Model

**Status:** Specification only. No code, schema, or repository changes were made. Builds directly on the Phase 1 Architecture Discovery and Phase 2 Platform Architecture above.

**Purpose:** This document is the canonical vocabulary and data-ownership contract for the Relativity platform. Every future PR, API, migration, and integration should be checked against it before being written. Where this document and existing code disagree, this document wins for *new* work; reconciling old code is a Phase 4+ implementation concern, not something resolved here.

**Revision history:** approved pending 13 targeted revisions, incorporated throughout this section — the precise retrieval-authorization guarantee (§2.2, §5), per-turn reauthorization and audit-only status of `entitledCollectionIdsAtCreation` (§2.3, §5), a default-deny Principal-resolution policy with explicit categories (§2.1), an explicit canonical **Principal** concept distinct from User (§2.1), a distributed Collection lifecycle with an AIKB-side registry and sync events (§2.2), a flat/no-inheritance initial Collection model (§2.2), staged token encryption (§2.4), execution-mechanism-agnostic OAuth refresh ownership (§2.4), corrected Background Event vs. Audit Event retention semantics (§2.5), metadata-only Audit Events by default (§2.5), an idempotent multi-state Organization deletion workflow (§2.1), contract versioning/replay fields on Service Request/Response (§2.5), and retention of the Folder/source-path concept alongside Collection (§1).

## 1. Language standardization

Pick-one decisions, made once, binding platform-wide.

| Ambiguous term(s) seen in the wild | Canonical term | Ruling |
|---|---|---|
| Client / Company / Tenant / Organization | **Organization** | "Tenant" survives only as an adjective (*tenant isolation*, *tenant-scoped query*), never as a noun for the entity. "Client" is retired from all new code, even though it remains the physical table name in Relativity today (Phase 1 §4) until a future migration renames it. |
| Chat Session / Portal Session / Conversation | **Conversation** (dialogue) vs. **Session** (auth) | "Session" is reserved exclusively for an authenticated login session (Supabase Auth JWT lifetime). Anything about a Q&A dialogue — Slack thread, portal chat, email thread — is a **Conversation**. Phase 1 found `client_member_sessions` conflating these two meanings; the successor concept (`surface_conversation_link`, §2.4) is deliberately not named "session." |
| Workspace / Provider Account | **Provider Account** | A single entity. "Workspace" is a *display label* used in UI copy when `ProviderAccount.accountType = collaborative` (Slack, Teams) — it is not a separate table or code path. |
| External Identity / Identity Link | **Identity Link** (persisted) | "External Identity" describes the *unresolved*, transient identity data a Surface Adapter reads off the wire before it's matched to an Identity Link row or falls through to Guest. It has no table of its own. |
| Embedding / Knowledge Chunk | **Knowledge Chunk** (entity) | "Embedding" is the vector value stored in `chunk.embedding` — an attribute, not an entity. |
| Vector Index / Knowledge Chunk | Chunk is the entity; **Vector Index** is infrastructure | The ivfflat/pgvector index over `chunk.embedding` is a database-operations concern (rebuilt on scale, not created/updated/deleted by application code) — documented in §2.2 for completeness, not modeled as an app-level entity. |
| Knowledge Source / Knowledge Document / Knowledge Import | **Knowledge Source** is an attribute, not a table | `source_provider` + `source_file_id` are fields recorded on both `Document` and `Import` — there is one provenance value per document, not a separate Source row to keep in sync. |
| Document Group / Knowledge Collection | **Knowledge Collection** | "Document Group" never shipped in either repo (Phase 1 §20) and is retired as a term before it does. |
| Folder / source path / Knowledge Collection | **Both kept, deliberately separate** | **Folder is not retired.** A Folder (or source path) is provider-side hierarchy — where a file physically sat in Google Drive, Dropbox, or a ZIP/folder upload (Phase 1's `document_import_log.source_path`). A **Knowledge Collection** is an authorization and semantic grouping, defined by an org admin, independent of where a file came from. The two may coexist on the same Document — a folder path is provenance metadata (§2.2 Knowledge Source); Collection membership is what gates retrieval. Neither implies the other. |
| Integration / Knowledge Surface / OAuth Connection | Three distinct, deliberately layered concepts | **Knowledge Surface** = the code contract a channel implements (§4 of Phase 2). **OAuth Connection** = a credential grant to one external account. **Integration** = the org-facing, configured *instance* that ties one Knowledge Surface to one or more OAuth Connections plus settings (default collections, enabled/disabled). These are commonly conflated in conversation ("the Slack integration") but must stay separate rows/types — see §2.4. |
| Thread / Conversation | **Conversation** is the entity; **Thread** is provider vocabulary | A provider's own thread identifier (Slack `thread_ts`, an email `Message-ID`) is captured as `Conversation.originThreadId` metadata. There is no standalone `Thread` table. |
| Citation / Knowledge Source | Both kept, deliberately distinct | **Source** answers "where did this *document* come from." **Citation** answers "which *document/chunk* informed *this specific answer*." Different question, different scope (document-scoped vs. answer-scoped) — not a duplication. |

**Retired terms (do not use in new code):** Client, Company, Tenant (as noun), Chat Session, Document Group, Slack Integration / Teams Integration (use "Slack Knowledge Surface" / the org's "Slack Integration" record distinctly). **Folder is not retired** — see the row above; it names a different concept than Collection and both remain in use.

## 2. Domain entities

Each entity: **Purpose · Owner · Source of truth · Relationships · Lifecycle (created → updated → deleted) · Security · Future extensibility.**

### 2.1 Identity & Access (owned entirely by Relativity — AIKB never owns identity)

| Entity | Purpose | Source of truth / Relationships | Lifecycle | Security | Future extensibility |
|---|---|---|---|---|---|
| **Principal** | The organization-scoped actor that can be authorized: exactly one of `team_member`, `guest`, or `service_account`. Not a physical table of its own — a role every authorization/audit/conversation-ownership field points to. | Realized via a `team_member_id \| guest_id \| service_account_id` reference — **never** a bare `user_id`. A `User` is not itself a Principal; only its `TeamMember` row (the User × Organization membership) is. | Resolved fresh on every request from the caller's verified identity — never cached across requests as a standing grant. | Using a raw `user_id` anywhere a Principal is expected is a modeling error — it bypasses the per-organization status/role checks that only `TeamMember` carries. | Future Principal-resolution paths (e.g., a device-bound Voice Assistant identity) are additional resolution *paths* into the same three subtypes, not new Principal subtypes. |
| **Organization** | The tenant boundary. Every other org-scoped entity hangs off this. | Relativity Global DB. 1—* User (via TeamMember), Group, Guest, ServiceAccount, OAuthConnection, Collection, Integration. | Created on signup/admin provisioning. Updated by org admins (name, settings). Deletion is an **idempotent, multi-system workflow**, tracked through explicit states: `active` → `deletion_requested` → `deleting` → `deleted`, with `deletion_failed` reachable from `deleting` (and retryable back into `deleting`). Relativity must receive confirmation that AIKB's data (documents, chunks, conversations, gaps) and every Integration/OAuth Connection have been cleaned up before advancing to `deleted`; reissuing the same deletion request at any state is safe (idempotent) rather than compounding side effects. | The root of every isolation check; every downstream query must trace back to exactly one Organization. | Multi-org-per-User is additive (join table), doesn't change Organization itself. |
| **User** | A real person with a login (Supabase Auth account). | Relativity Global DB (Supabase Auth + `users` profile). 1—* TeamMember (one per Organization it belongs to). | Created on signup/invite acceptance. Updated on profile change. Deleted on account deletion (cascades TeamMember rows, not Organization). | Never trust a client-supplied user id; always resolve from the verified JWT. | Multi-org membership (Should Have per Phase 2 §12) is just multiple TeamMember rows for one User — no new entity needed. |
| **Team Member** | A User's membership in one Organization: role, status, group memberships. **This is the organization-scoped Principal for a User** — a User is never directly authorized; only its Team Member row is (see Principal, above). | Relativity Global DB. *—1 User, *—1 Organization, *—1 Role, *—* Group. | Created on invite acceptance. Updated on role/status change (disable/re-enable), last-active timestamp. Deleted (or soft-deleted) on removal from org; must guard against removing the last owner (Phase 1 §5 rule, kept). | The unit tenant-isolation queries actually filter on (via `client_id`/`organizationId`). | None needed — deliberately minimal. |
| **Role** | Coarse permission tier: owner / admin / member / viewer. | Relativity Global DB (currently a CHECK constraint enum; recommend a reference table if custom roles become a Should Have). | Created at platform design time (fixed set today). Updated only if custom roles are added later. Never deleted while any TeamMember references it. | Drives which UI/admin actions are permitted — must never be inferred from anything but this field. | Org-defined custom roles = Should Have; requires moving from CHECK-constraint enum to a real table + default Permission bundle. |
| **Permission** | A single named capability (`upload_documents`, `manage_integrations`, `view_restricted_collections`). | Relativity Global DB — **does not exist as a table today**; Phase 1 found only coarse Role gating. | Created when a new capability needs finer control than Role provides. Updated rarely. Deleted only when a capability is retired entirely. | Lets an org grant/revoke one capability without inventing a new Role — reduces role sprawl. | Should Have, not Must Have — introduce only when a real product need for per-capability overrides appears. |
| **Group** | A named set of TeamMembers used **only** for Collection entitlement (§2.2) — e.g. "Sales," "Legal." Independent of Role. | Relativity Global DB — new entity, does not exist today. *—* TeamMember, *—* Collection (entitlement). | Created by org admins. Updated (membership changes) by admins or self-service if allowed. Deleted by admin; deletion must cascade entitlement removal, never silently widen access. | A viewer can be in a restricted Group; an admin is not automatically in every Group — entitlement and role authority never conflate. | Nested groups (Group-of-Groups) are Future, not Must/Should — add only if a real org structure needs it. |
| **Guest** | A resolvable-but-unlinked external identity (e.g., a Slack user not yet matched to a TeamMember, a Slack Connect external participant). The default resolution outcome for every Principal-resolution category except a mapped active Team Member (see the categories table below). | Relativity Global DB — new entity. *—1 Organization, *—1 IdentityLink (the link that identified them as a Guest rather than a TeamMember). | Created automatically the first time an unlinked external identity is seen. Updated if later linked to a real TeamMember (Guest record retained for audit history, not deleted). Deleted only on data-retention expiry. | **Default-deny, non-bypassable:** a Guest receives no Collection access by default — never entitled to `restricted` collections regardless of any Group mapping, and General-collection access is granted only if the Organization has explicitly opted in (see the categories table below); never counted for gap-review visibility; always audited distinctly from linked users (Phase 2 §10). | None — this entity should stay deliberately minimal and restrictive. |
| **Service Account** | A machine identity for Public API callers and internal jobs — never a human Role. | Relativity Global DB — new entity. *—1 Organization, *—1 Role/Permission set of its own. | Created by an org admin issuing an API key. Updated on scope/permission change. Deleted/revoked on key rotation or offboarding. | Must never inherit a human's Role by accident; scoped narrowly by default (least privilege). | Foundation for the Public API (Phase 2 §11) — no further model change needed when that ships. |
| **External Identity** *(conceptual, not persisted)* | The raw identity data read off a provider's wire format (Slack `user_id` + `team_id`, a verified email address) before resolution. | N/A — exists only in-memory during `identifyUser()` (Phase 2 §4) until resolved into an Identity Link or a new Guest. | N/A. | Never trusted as an authorization input on its own — must always be resolved first. | — |
| **Identity Link** | The persisted mapping from one External Identity to one Principal (TeamMember or Guest). | Relativity Global DB — new entity, generalizes today's implicit Slack-only linkage. *—1 Organization, *—1 Provider, *—1 (TeamMember or Guest). | Created the first time an external identity is seen (as a Guest link) or explicitly by an admin/self-service claim flow (as a TeamMember link). Updated when a Guest is later claimed by a real TeamMember. Deleted on disconnect/unlink. | Unique constraint on (Organization, Provider, externalIdentityId) — one external identity maps to exactly one Principal per org, never two. | Same table shape serves Slack, Teams, and future email/Discord identities — no schema change needed to add a provider. |
| **Authorization Context** *(conceptual, request-scoped, not persisted)* | The resolved `{organizationId, principal, role, entitledCollectionIds}` object produced once per request by `authorize()` (Phase 2 §4). | Computed fresh by Relativity Core; embedded (signed) inside the Service Request sent to AIKB. Never written to a table — a stale cached copy is exactly the bug this design avoids. | Created and discarded within a single request lifecycle. | This is the single value every downstream authorization decision hangs on — it must never be reconstructed by anyone but Relativity Core. | Short-lived caching (seconds, not minutes) is an optimization, not a model change — must never outlive a Collection/Group edit by more than that window. |

**Principal resolution categories (default-deny).** Identifying *which* Principal (if any) an incoming request maps to must distinguish six cases. Only mapped active Team Members receive normal access; verified-but-unmapped internal users are opt-in; every other case defaults to deny:

| Category | Description | Default access |
|---|---|---|
| Mapped active Team Member | External/internal identity resolves via Identity Link to an active `TeamMember` row | Normal Group-based Collection access, per Role/Group |
| Verified internal but unmapped user | Identity is verified as belonging to the Organization (e.g., email domain match) but has no Identity Link/TeamMember yet | **Deny by default.** May receive General-collection-only access, and only if the Organization has explicitly enabled that policy |
| Provider guest | An identity known to the provider (e.g., present in a Slack workspace) with no verified relationship to the Organization | **Deny.** Resolved as `Guest` with the hard default-deny floor above |
| External/Slack Connect user | An identity explicitly external to the Organization's own provider account (e.g., a Slack Connect shared-channel participant from another company) | **Deny.** Always resolved as `Guest`; never eligible for the opt-in General-access policy above |
| Deactivated user | A `TeamMember` whose `status` is `disabled`/`revoked` | **Deny.** Must fail closed even if a stale Identity Link still points to it — status is re-checked on every resolution, never cached |
| Bot/system identity | An automated caller (a provider's own bot user, a webhook relay) | **Deny**, unless explicitly registered as a `ServiceAccount` with its own scoped grant — never silently treated as a human Principal |

Every one of these six cases must be distinguished before any Collection entitlement is computed, and the default is always deny, not fall-through to a lenient case — this directly closes the Phase 1 finding that AIKB's Slack prototype had no real identity resolution at all (§0).

### 2.2 Knowledge Core (owned entirely by AIKB — Relativity never performs retrieval)

| Entity | Purpose | Source of truth / Relationships | Lifecycle | Security | Future extensibility |
|---|---|---|---|---|---|
| **Knowledge Collection** | *Definition + entitlement* live in Relativity (§2.1 Group); *assignment* (which documents belong to it) and *retrieval enforcement* live in AIKB — see Phase 2 §3 for the full split rationale, and "Distributed Collection lifecycle" below for how AIKB independently verifies a Collection without a live cross-database call. | AIKB `document_collections` (Document *—* Collection) plus a local `collection_registry` (below). Collection *identity* (id, name, restriction level) is defined in Relativity; AIKB keeps a synchronized copy, not a live query — no physical cross-DB FK, same pattern already used for `client_id` today (Phase 1 §7). | Assignment rows created at ingestion/tagging time. Updated when a document is re-tagged. Deleted when a document is deleted or untagged. The Collection's own definition lifecycle (created/updated/archived) is Relativity's and propagates to AIKB's registry via the sync events below. | **Precise guarantee:** AIKB stores organization knowledge, but retrieval must filter by organization and currently authorized collections before any chunks enter answer generation, the LLM prompt, citations, or the response. Enforced by filtering `collection_id IN (:entitledCollectionIds)` inside the same SQL statement as the `organization_id` filter (Phase 2 §3, Must Have) — and, per the registry below, every id in that list is independently verified to exist, be active, and belong to the requesting Organization before it is trusted. | **Initial implementation is intentionally flat**: no Collection inheritance, entitlement is only ever an explicit Group-to-Collection grant. This is sufficient for a Slack MVP and avoids shipping an unreviewable inheritance chain on day one. Inheritance/deny-override logic (Phase 2 §3) remains a **Future** capability layered into Relativity's entitlement resolution later, without changing AIKB's flat `collection_id IN (...)` enforcement. |
| **Knowledge Document** | One ingested, indexed file. | AIKB `knowledge_documents` (unchanged from Phase 1 §7). 1—* Chunk, *—* Collection, *—1 originating Import or Sync. | Created by an Import/Sync ingestion run. Updated on reindex (content-hash change). Deleted (soft, `status='deleted'`) on user delete or source removal. | Content-hash dedup (Phase 1 §8) prevents redundant storage of identical content across sources — keep. | No change needed to add new source providers — `source_provider` is just a string. |
| **Knowledge Chunk** | A retrievable segment of a Document's text, with its embedding vector. | AIKB `knowledge_chunks`, pgvector `embedding` column. *—1 Document. | Created during ingestion (chunking + embedding step). Never updated in place — a content change deletes and recreates chunks for that document (Phase 1 §8 pattern, keep). Deleted on document delete/reindex. | The unit vector search actually returns — collection/org filtering happens at this table's query, not upstream. | Chunking strategy (size/overlap) is swappable without schema change — it only affects what gets written, not the table shape. |
| **Knowledge Source** *(conceptual, attribute not entity)* | Where a Document's content originated: provider + external file id — distinct from **Folder** (§1), which is the provider-side hierarchy/path a source file lived in and may be recorded alongside Source, not replaced by Collection. | Fields on `Document` (`source_provider`, `source_file_id`) and `Import` (same, at time of import). | Set once at Document creation; immutable identity of "which external file this is." | Used for dedup (Phase 1 §8) and for `document_import_log`-style provenance display — never used as an authorization signal. | Adding a new provider is a new string value, not a new table. |
| **Embedding** *(conceptual, attribute not entity)* | The numeric vector representation of a Chunk's text. | `Chunk.embedding` column (pgvector `VECTOR(1536)`, Phase 1 §7). | Written once per chunk at ingestion; regenerated only on reindex/model change. | Never exposed to a caller directly — only consumed inside the retrieval RPC. | Model/dimension changes require a migration + full reindex, not a schema redesign. |
| **Vector Index** *(infrastructure, not an app-level entity)* | The database index (ivfflat/HNSW) enabling fast similarity search over `Chunk.embedding`. | AIKB Postgres, owned operationally by whoever runs migrations — not created/updated/deleted by application code. | Created once at schema setup; tuned/rebuilt as data scale grows (an ops task, not a feature). | Index choice/parameters affect recall, not authorization — security is entirely enforced in the surrounding SQL `WHERE` clause, not the index itself. | Swapping ivfflat → HNSW or a different vector store is an infra migration invisible to the domain model. |
| **Citation** *(value object, embedded — not its own table)* | A structured reference from one Answer to the Document(s)/page(s) that supported it. | Embedded JSON on `Message.sources` (AIKB, Phase 1 §14). References `documentId`, optional `pages[]`. | Created at answer-generation time, alongside the Message. Never updated independently of its parent Message. Deleted only when the parent Message is deleted. | If the answer is classified as a knowledge gap, citations are forced empty (Phase 1 §14 behavior, keep) — a gap answer must never claim false grounding. | Could gain chunk-level (not just document-level) precision later without a schema change — it's already a flexible JSON shape. |

**Distributed Collection lifecycle (Relativity ↔ AIKB).** Definition and entitlement remain Relativity's; assignment and enforcement remain AIKB's. To let AIKB verify a signed collection list without a live cross-database call on every request, AIKB maintains a lightweight **local Collection registry**, kept in sync rather than queried live:

```
aikb.collection_registry
  collection_id     -- Relativity's Collection id, copied verbatim
  organization_id
  status             -- active | archived
  version            -- monotonically increasing, bumped on every Relativity-side edit
  updated_at
```

Synchronization is event-driven, using the platform's standard naming (§4):

| Event | Relativity action | AIKB action |
|---|---|---|
| `knowledge.collection.created` | New Collection defined | Insert registry row, `status=active`, `version=1` |
| `knowledge.collection.updated` | Name/restriction/entitlement edited | Update registry row, bump `version` and `updated_at` |
| `knowledge.collection.archived` | Collection retired (never hard-deleted, to preserve historical Citation/Conversation references) | Update registry row, `status=archived` — archived collections are excluded from `entitledCollectionIds` resolution and fail the verification below |

**Retrieval-time verification (Must Have):** before running any retrieval, AIKB checks every id in the signed `entitledCollectionIds` list against `collection_registry` — the id must exist, `status=active`, and `organization_id` must match the request's `organizationId`. Any id failing this check is dropped from the filter, never passed through: a stale or forged collection id can only narrow a query, never widen it. This is the AIKB-side half of the guarantee above — a signed list from Relativity is necessary but not sufficient; AIKB independently confirms it before trusting it.

### 2.3 Knowledge Lifecycle & Conversation (owned entirely by AIKB)

| Entity | Purpose | Source of truth / Relationships | Lifecycle | Security | Future extensibility |
|---|---|---|---|---|---|
| **Knowledge Import** | A one-shot ingestion run: upload, ZIP, a picked Drive file. Subsumes today's `knowledge_ingestion_jobs` + `document_import_log` (Phase 1 §7/§8) into one lifecycle concept. | AIKB `knowledge_ingestion_jobs` (execution state) + Relativity `document_import_log` (provenance/display) — kept as two tables (different consumers) but one *conceptual* entity; do not add a third. 1—* Document. | Created when a Surface/admin triggers ingestion. Updated through queued→running→completed/failed. Not deleted — retained as history. | `forceReindex` and content-hash dedup (Phase 1 §8) already prevent duplicate processing — keep. | Batch imports (folder upload, ZIP) are just an Import with multiple resulting Documents — no separate "batch" entity needed. |
| **Knowledge Sync** | A *recurring* ingestion relationship — e.g., a connected Google Drive folder kept continuously fresh. Distinct from Import because it needs its own cursor/watermark state, not just a one-time job record. | AIKB — **does not exist today** (Phase 1 confirmed Google Drive sync was deliberately removed, §19). New entity when reintroduced. *—1 OAuth Connection (credential to poll with), 1—* Document (currently managed by this sync). | Created when an org enables continuous sync for a Connected Source. Updated on each poll (lastSyncedAt, cursor). Deleted/paused on disconnect. | Must reuse the same Collection-assignment and org-scoping as manual Import — a synced document is not exempt from authorization. | This is the entity that makes Google Drive/Dropbox/SharePoint/OneDrive/Box/Notion (§6) full "keep-fresh" sources rather than one-shot imports — build once, reuse per provider. |
| **Knowledge Gap** | A question the knowledge base couldn't answer, recorded for admin review. One universal model, per Phase 2 §8. | AIKB `knowledge_gaps`, extended with `origin`, `originMetadata`, `idempotencyKey`, `reportedBy` (Phase 2 §8). *—1 Conversation, *—1 Message (optional), *—1 Collection context (which entitled set was searched). | Created exclusively by the knowledge pipeline itself (system-detected, inside `/ask`) or via explicit user flag (`/gaps`) — **never** by a Surface Adapter directly (Must Have, Phase 2 §8). Updated on review (status change). Deleted only by retention policy, never by a user. | Unique `idempotencyKey` constraint prevents duplicate rows from redelivered Slack events or retried requests. | New origins (Teams, email) require zero schema change — `origin` is just a string enum. |
| **Knowledge Conversation** | A dialogue between one Principal and the knowledge base, regardless of surface. | AIKB `conversation` (Phase 2 §7). *—1 Organization (ref), *—1 Principal (ref), 1—* Message, 0—* KnowledgeGap. | Created on first question (explicitly or implicitly via `/ask` without a `conversationId`). Updated on `lastActivityAt`/title changes. Soft-deleted by user action (Phase 1 §11 pattern, keep). | `entitledCollectionIdsAtCreation` is **audit metadata only** — a record of what was searched at the time, kept so a historical answer's implied scope is never retroactively reinterpreted. It must never be reused as a live authorization grant: every new question and every follow-up turn is reauthorized using a newly resolved Authorization Context (§2.1) at request time. Conversation history must never preserve stale access after a Principal loses a Group or Collection entitlement. | Every future surface (Teams, email, voice) reuses this table unchanged — only `origin`/`originThreadId` vary. |
| **Knowledge Message** | One turn (user question or assistant answer) inside a Conversation. | AIKB `message`. *—1 Conversation, 0—* Attachment, 0—* Citation (embedded). | Created on each turn. Never updated (immutable once written — an answer is not silently edited after the fact). Soft-deleted alongside its Conversation. | `role` (user/assistant/system) constrains what content means — must never be spoofable by a Surface Adapter. | None needed — deliberately minimal and provider-agnostic. |
| **Attachment** | A raw file sent as part of a Conversation turn (a pasted image, an emailed PDF) — distinct from a Knowledge Document. | AIKB, attached to `Message`; bytes in shared Storage (Phase 1 §9 bucket pattern). | Created when a Surface receives a file alongside a question. Updated never. Deleted with its Message, or explicitly promoted into a Knowledge Document via an explicit Import action. | **Must Have principle:** an Attachment is never automatically ingested as organizational knowledge — promotion to Document requires an explicit action, preventing a stray Slack image from silently becoming "company knowledge." | Any future surface that can receive files (email, Teams) reuses this unchanged. |

### 2.4 Integrations & OAuth (owned entirely by Relativity — no provider-specific logic in AIKB)

| Entity | Purpose | Source of truth / Relationships | Lifecycle | Security | Future extensibility |
|---|---|---|---|---|---|
| **OAuth Provider** | Platform-global registry row describing one provider type's OAuth mechanics (Slack, Teams, Google, Dropbox, Microsoft Graph, ...). | Relativity Global DB, new table (Phase 2 §6) — replaces one-off `config/index.js` blocks per provider. Not org-scoped — one row per provider *type*, shared by all orgs. 1—* OAuth Connection. | Created by platform engineers when a new provider is supported. Updated on scope/endpoint changes. Deleted only when a provider is fully sunset. | Holds no secrets itself — just `authUrl`, `tokenUrl`, `scopes[]`, `authStyle`. Client id/secret live in the secrets manager (§2.4 OAuth Token note). | Adding a provider is one new row + a small adapter, not new callback routes (Phase 2 §6 uniform `/oauth/:providerId/*` pattern). |
| **OAuth Connection** | One Organization's authorization grant to one Provider Account. Replaces today's `oauth_tokens` row-per-provider pattern (Phase 1 §14). | Relativity Global DB. *—1 Organization, *—1 OAuth Provider, *—1 Provider Account, 1—* OAuth Token (history). | Created at OAuth callback completion. Updated on status change (connected/expired/revoked), scope regrant. Deleted on disconnect (must also revoke at the provider, Phase 2 §6 Should Have). | This is the row `requireMemberContext`-equivalent authorization checks must trace back to — a Connection with no valid Token is not usable, and must fail closed, not silently degrade. | Multiple Connections per Organization per Provider (e.g., two Slack workspaces) fall out naturally — no schema change. |
| **OAuth Token** | The actual encrypted credential material (access + refresh token) for one Connection. | Relativity Global DB, split from Connection metadata (Phase 2 §6) specifically to close Phase 1's plaintext-storage finding. *—1 OAuth Connection. | Created at token exchange. Updated on refresh (new row per rotation, or in-place update with `encryptionKeyVersion` bump — implementation detail for Phase 4). Deleted on disconnect/revocation. | **Staged requirement.** Initial (Must Have, sufficient for the Slack MVP): AES-256-GCM with a versioned *application* encryption key (`encryptionKeyVersion`), a random IV per encryption, and an authentication tag verified on every decrypt — no plaintext fallback under any failure mode. Future hardening (Should Have, not a Slack prerequisite): move the application key into a managed KMS, derive per-organization data keys from it, and automate rotation. Full KMS integration is not required before Slack ships unless the current infrastructure already provides it. Tokens are never logged and never returned in any API response, not even to admins. | Refresh remains a Relativity-owned **domain responsibility** — one shared sweep across all providers, no per-provider refresh code — but the execution mechanism (scheduled job, queue-driven worker, or another approach) is intentionally left open, to be selected during implementation planning rather than prescribed here. |
| **Provider Account** | The specific external account/tenant/workspace a Connection points to (a Slack team, a Microsoft 365 tenant, a Dropbox account). Absorbs "Workspace" (§1). | Relativity Global DB, new entity. *—1 OAuth Provider, 1—* Channel (only if `accountType = collaborative`), 1—* OAuth Connection (an org may reconnect the same account). | Created the first time a given external account is seen during OAuth. Updated (display name, metadata) on refresh. Deleted only when no Connection references it. | Distinguishes "which external tenant" from "our credential to it" — lets an admin see *what* is connected even if the Connection is currently expired. | `accountType` field (collaborative vs. mailbox vs. file-store) is what makes this one table serve Slack, Gmail, and Google Drive alike. |
| **Channel** | A sub-scope within a collaborative Provider Account (a Slack channel, a Teams channel) — used for origin metadata and future channel-to-Collection default mapping. | Relativity Global DB, new entity. *—1 Provider Account. | Created lazily, the first time a message from that channel is seen (or explicitly, via admin-configured channel mapping). Updated on rename. Deleted when the provider reports the channel is gone. | Not itself a security boundary — Collection entitlement (§2.2), not Channel identity, is what gates retrieval. A channel can be *labeled* with a default Collection for convenience, but that's a UI default, not an authorization rule. | This is the row Phase 1's `slack_channel_id`-to-`client_id` mapping (the "Blocking prerequisite #1" from Phase 1 §10) generalizes into. |
| **Integration** | The org-facing, configured instance: "this Organization has Slack enabled," tying one Knowledge Surface type to one or more OAuth Connections plus settings. | Relativity Global DB, new entity. *—1 Organization, *—1 Knowledge Surface (type reference), *—* OAuth Connection, settings JSON (default Collection, enabled toggle). | Created when an admin enables a Surface for their org. Updated on settings change. Deleted/disabled on admin action — distinct from deleting the underlying OAuth Connection (an Integration can be paused without losing the credential). | This is the row an admin UI actually renders as "Connected: Slack ✅" — the single place to check "is this org allowed to receive Slack questions right now." | This is precisely the layer that makes §11 of Phase 2 (roadmap) hold — a new Integration row, not new pipeline code. |
| **Knowledge Surface** *(type/contract, not a per-org row)* | The code contract every interface implements (Phase 2 §4): `authenticate/identifyUser/receiveQuestion/authorize/queryKnowledge/formatAnswer/deliverAnswer/recordAudit`. | Relativity codebase — a small registry of implemented types (`portal`, `slack`, `teams`, `gmail`, `outlook`, `api`), referenced by id from `Integration`. | "Created" = a new adapter is written and registered. Not an org-scoped row with its own lifecycle — it's a *type*, instantiated per-org via Integration. | `authorize()` and `queryKnowledge()` steps are never overridden per-surface (Must Have, Phase 2 §4) — this is the mechanism that prevents another AIKB-style shadow implementation. | Adding a surface = adding one registry entry + one adapter implementation — the definition of "should require very little work" (Phase 2 design objective 1). |

### 2.5 Platform Infrastructure & Contracts (shared conventions, not owned by either repo alone)

| Entity | Purpose | Source of truth / Relationships | Lifecycle | Security | Future extensibility |
|---|---|---|---|---|---|
| **Background Event** | The `domain.entity.action`-shaped event envelope (Phase 2 §9) used both for Relativity's durable outbox and AIKB's Inngest functions. | A **shared schema/convention**, not a single table — Relativity's `outbound_events` and AIKB's Inngest event stream are physically separate durable stores that must agree on envelope shape when crossing TB3. | Created whenever a domain fact needs asynchronous follow-up. Updated as processing state advances (queued→running→done, mirroring Phase 1's job-status pattern). **Operational record, not a historical one** — retained only for a limited, configurable window sufficient for debugging and replay-detection, then purged. It is never treated as the platform's audit trail; that is Audit Event's exclusive role, below. | Carries the same `idempotencyKey`/signature discipline as a synchronous Service Request, including the same contract-versioning and replay-control fields: `schemaVersion`, `requestId`, `issuedAt`, `expiresAt`, `issuer`, `audience` (§2.5 Service Request). Payloads are **minimized/redacted** — a Background Event carries the ids and routing metadata needed to process it, not full question/answer text, provider payloads, retrieved chunk content, or secrets; anything sensitive is looked up fresh from its owning system when the event is processed. | New event types are just new `domain.entity.action` strings — no schema change to the envelope itself. |
| **Audit Event** | A permanent, structured record of "what happened, by whom," for compliance and debugging — distinct from a Background Event (a transient, short-retention operational signal, not a permanent record). | Relativity Global DB, new `audit_log` table (Phase 2 §10, Must Have). *—1 Organization, *—1 Principal (actor), references the acted-upon entity by type+id. | Created by every Knowledge Surface lifecycle step (Phase 2 §4) and every OAuth/admin action. Never updated (append-only). Retained according to a defined retention policy — an append-only historical record, not subject to Background Event's short operational window. | **Stores metadata by default** — actor, action, target entity type/id, timestamp, origin, outcome — never the full question text, answer text, provider payloads, retrieved chunk content, or secrets/tokens. This is the trail that answers "who did what to which entity, when" across every surface — Phase 1 found this only as scattered `console.log` calls; this table is the fix. | One shared writer used by every surface — no per-integration audit logic to keep in sync. |
| **Notification** | An outbound alert to a User/admin — "a knowledge gap needs review," "an OAuth connection is expiring." | Relativity Global DB, new entity — **does not exist today**. *—1 Organization, *—1 recipient (User/Guest/ServiceAccount), triggered by an Audit Event or Background Event. | Created when a triggering condition fires (scheduled sweep or event-driven). Updated on read/dismissed state. Deleted by retention policy. | Delivery is provider-specific (email, Slack DM, in-app) and lives in the Delivery Router (Phase 2 §1) — the Notification record itself is provider-agnostic. | Reuses the exact same Surface delivery machinery built for answering questions — "notify a user" and "answer a user" are the same delivery primitive pointed at different content. |
| **Service Request** | The signed envelope Relativity sends to AIKB (Phase 2 §5): org, principal, entitled collections, origin, idempotency key, plus contract-versioning and replay-control fields — `schemaVersion`, `requestId`, `issuedAt`, `expiresAt`, `issuer`, `audience`. | **Ephemeral, per-request** — not persisted as its own row, though its key fields (including `requestId`) are echoed into the resulting Conversation/Message/Audit Event for traceability. | Constructed fresh by Relativity Core on every AIKB call. | Every field is either derived server-side (Authorization Context) or explicitly caller-supplied-and-signed — never a raw unsigned `organizationId`. `expiresAt` bounds how long a signed request can be replayed; `issuer`/`audience` bind the signature to a specific caller and recipient so it can't be replayed against a different consumer; `schemaVersion` lets AIKB reject or branch on an envelope shape it doesn't understand instead of guessing. | Adding a new field (e.g., a future `deviceId` for a Voice Assistant surface) is additive and backward compatible — old callers simply omit it. |
| **Service Response** | AIKB's reply to a Service Request: answer, citations, conversation id, gap status — echoes the originating `requestId` and carries its own `schemaVersion` so a caller can detect a contract mismatch. | **Ephemeral, per-request** — persisted content lives in Conversation/Message/KnowledgeGap; the response envelope itself is not separately stored. | Constructed fresh by AIKB on every request. | Never includes chunk content beyond what the Citation model exposes — no accidental over-return of unfiltered context. | New response fields (e.g., confidence score) are additive. |
| **Idempotency Key** | A caller-supplied (or provider-delivery-derived) token preventing duplicate processing of the same logical event. | Used two ways: (a) an **ephemeral** request attribute on every Service Request/Background Event; (b) a **persisted**, uniquely-constrained column on `KnowledgeGap` and `outbound_events` (Phase 2 §5, §8, §9, all Must Have). | Generated by the Surface Adapter at receipt time (from the provider's own dedup primitive where one exists) or by the caller for Portal/API. Never regenerated downstream. | The mechanism that makes Slack's redelivery-on-timeout behavior safe — without it, every retry becomes a duplicate gap or duplicate answer. | One shared derivation strategy per provider type — not reinvented per integration. |
| **Thread** *(provider vocabulary, not a persisted entity — see §1)* | The external system's own notion of a threaded exchange (Slack `thread_ts`, an email `Message-ID` chain). | Captured only as `Conversation.originThreadId`. | N/A — has no lifecycle of its own; it's metadata on Conversation. | Never used as an authorization key — only as a lookup key to resolve which Conversation to continue. | — |

## 3. Diagrams

### 3.1 Context diagram

```
                     ┌───────────────────────────────┐
                     │        People & Systems         │
                     │  Org Admins · Team Members ·     │
                     │  Guests · External Providers ·   │
                     │  API Clients                     │
                     └───────────────┬───────────────┘
                                     │
                      (every interface, per Knowledge Surface)
                                     ▼
                     ┌───────────────────────────────┐
                     │           RELATIVITY            │
                     │  Identity · Authorization ·      │
                     │  OAuth Platform · Surfaces ·      │
                     │  Delivery                        │
                     └───────────────┬───────────────┘
                          signed Service Request
                                     ▼
                     ┌───────────────────────────────┐
                     │              AIKB                │
                     │  Retrieval · Reasoning ·         │
                     │  Conversations · Gaps ·           │
                     │  Durable Execution                │
                     └───────────────┬───────────────┘
                                     ▼
                     ┌───────────────────────────────┐
                     │      LLM / Embedding Provider    │
                     └───────────────────────────────┘
```

### 3.2 Ownership diagram

```
RELATIVITY OWNS                         AIKB OWNS
────────────────                        ─────────
Organization                            Knowledge Document
User / TeamMember / Role / Permission   Knowledge Chunk (+ Embedding)
Group                                   Vector Index (infra)
Guest / Service Account                 Knowledge Collection (assignment)
Identity Link                           Knowledge Import / Sync (execution)
OAuth Provider / Connection / Token     Knowledge Gap
Provider Account / Workspace / Channel  Knowledge Conversation / Message
Integration                             Attachment
Knowledge Surface (type registry)       Citation (embedded)
Audit Event
Notification
Background Event (outbox side)

SHARED CONVENTIONS (no single owner — must match on both sides)
─────────────────────────────────────────────────────────────
Service Request / Service Response envelope shape
Idempotency Key derivation rules
Background Event naming (`domain.entity.action`)
Knowledge Collection *definition* (Relativity) vs *assignment* (AIKB) — split by design, §2.2
```

### 3.3 Entity relationship diagram (condensed, cross-repo)

```
RELATIVITY                                          AIKB
──────────                                          ────
Organization 1───* TeamMember *───1 User            Organization(ref) 1───* Conversation
Organization 1───* Group  *───* TeamMember           Conversation 1───* Message
Organization 1───* Guest                             Message 0───* Attachment
Organization 1───* ServiceAccount                     Message ── Citation (embedded JSON)
Organization 1───* IdentityLink ───1 (TeamMember|Guest) Organization(ref) 1───* KnowledgeGap
Organization 1───* OAuthConnection ───1 OAuthProvider  KnowledgeGap *───1 Conversation
OAuthConnection 1───* OAuthToken                       KnowledgeGap *───1 Message (optional)
OAuthConnection *───1 ProviderAccount 1───* Channel     Organization(ref) 1───* Document
Organization 1───* Integration ───1 KnowledgeSurface    Document 1───* Chunk
Integration *───* OAuthConnection                       Document *───* Collection(assignment)
Organization 1───* Collection(definition) ───* Group    Document *───1 Import or Sync
Collection ──(self)── parent Collection [Future — flat/no inheritance initially]   Import 1───* Document
Organization 1───* SurfaceConversationLink              Sync 1───* Document
   SurfaceConversationLink ─(logical FK, no physical FK)─▶ AIKB.Conversation.id
Organization 1───* AuditEvent
Organization 1───* Notification
```

### 3.4 Interaction diagram (generic question flow, entity-level — see Phase 2 §1 for the component-level version)

```
1. Surface Adapter receives raw event
2. → resolves External Identity → Identity Link → Principal (TeamMember | Guest)
3. → Relativity Core computes Authorization Context (entitledCollectionIds)
4. → Relativity Core looks up/creates SurfaceConversationLink → conversationId
5. → Relativity Core sends signed Service Request to AIKB
6. → AIKB creates/appends Message (role=user) on Conversation
7. → AIKB runs Chunk retrieval filtered by (organizationId, entitledCollectionIds)
8. → AIKB constructs answer, builds Citation(s), creates Message (role=assistant)
9. → AIKB evaluates for Knowledge Gap; if triggered, creates KnowledgeGap (idempotent)
10. → AIKB returns Service Response
11. → Relativity Core formats + delivers via Surface Adapter
12. → Relativity Core writes Audit Event for every step above
```

### 3.5 Data flow diagram (ingestion path, entity-level)

```
Surface/Admin triggers upload
   → Knowledge Import created (status=queued)
   → file bytes → shared Storage
   → Import status=running
   → Knowledge Document created (status=pending, source recorded)
   → parse → chunk → embed
   → Knowledge Chunk rows created (embedding populated)
   → Document.status=indexed, Import.status=completed
   → (if applicable) Document tagged into Knowledge Collection(s)
```

## 4. API & naming standards

| Area | Standard | Rationale |
|---|---|---|
| **Internal service contract** (Relativity↔AIKB) | RPC-style verb endpoints (`/ask`, `/gaps`, `/resolve-collections`, Phase 2 §5) | Not a public resource API — optimized for the specific, small set of platform-internal operations; changing this to REST-resource style buys nothing for a two-party contract. |
| **Public API** (external, Phase 2 §11) | Resource-oriented REST: plural nouns, verb-free paths — `POST /v1/conversations`, `POST /v1/conversations/{id}/messages`, `GET /v1/collections` | External consumers expect REST conventions; different audience from the internal contract, so a different convention is justified, not inconsistent. |
| **Resource naming** | Singular entity name in code/docs (`Organization`), plural in routes (`/organizations/{id}`) | Matches common REST convention and today's existing route shapes. |
| **Error naming** | `{ error: { code: SCREAMING_SNAKE_CASE, message, requestId } }`, code namespaced by domain (`AUTH_INVALID_TOKEN`, `KNOWLEDGE_COLLECTION_NOT_FOUND`, `OAUTH_CONNECTION_EXPIRED`) | One parseable shape for every client (Portal, future mobile, Public API) instead of ad hoc error bodies per route. |
| **Status naming** | lowercase `snake_case` enums (`open`, `reviewed`, `resolved`, `connected`, `expired`, `revoked`, and multi-step workflows like Organization deletion's `active`/`deletion_requested`/`deleting`/`deletion_failed`/`deleted`, §2.1) | Matches existing DB CHECK-constraint conventions found in Phase 1 — extend, don't replace. |
| **Event naming** | `domain.entity.action`, lowercase, dot-namespaced; past tense for completed facts, present-imperative for requests (Phase 2 §9) | One vocabulary shared by Relativity's outbox and AIKB's Inngest functions — this is what makes Background Event a genuinely shared convention (§2.5). |
| **Database naming** | `snake_case` tables, plural (`organizations`, `knowledge_documents`); FK columns `{entity}_id`; every table gets `created_at`/`updated_at`, soft-deletable tables also get `deleted_at` | Matches existing convention (Phase 1 §7) — generalized to every new table in this model, not just chat tables. |
| **Migration naming** | `{YYYYMMDD}_{description}.sql`, one change per file (Phase 1 §7 convention, kept) | Already proven in both repos; formalize as the standard rather than an accident of history. Should Have: adopt a real migration-tracking tool instead of manual SQL-editor runs (AIKB's current process, Phase 1 §7). |
| **Environment variable naming** | `{DOMAIN}_{PURPOSE}` SCREAMING_SNAKE_CASE (`AIKB_*`, `GLOBAL_*` already used); OAuth secrets as `OAUTH_{PROVIDER}_CLIENT_ID`/`_CLIENT_SECRET`, not ad hoc per-provider names | Directly closes the Phase 1 finding of duplicate/dead Slack env vars (`SLACK_CLIENT_ID` vs `SLACK_APP_ID`) — one naming rule prevents that drift recurring for Teams/Gmail/Outlook. |
| **Configuration naming** | All `process.env` reads centralized in one `config/index.js` per repo; no scattered direct reads outside standalone scripts | Matches existing pattern (Phase 1 §16) — keep, enforce as a lint rule going forward. |
| **Queue/event naming** | Inngest function ids `kebab-case`, matching their event's domain (`knowledge-document-ingest` already this shape) | Consistency between the human-readable function id and the machine event name it handles. |

## 5. Design principles

**Identity & Authorization**
1. AIKB never owns identity — it receives a resolved, signed Authorization Context on every request.
2. Relativity never performs retrieval, embedding, or LLM reasoning — those live exclusively in AIKB.
3. Every retrieval is authorized (collection-filtered) *inside* the same SQL statement that performs the vector search, never as a post-filter. Precisely: AIKB stores organization knowledge, but retrieval must filter by organization and currently authorized collections before any chunks enter answer generation, the LLM prompt, citations, or the response — the guarantee is about what leaves retrieval, not an absolute claim that AIKB never receives or stores unauthorized knowledge, since AIKB does store all of an organization's ingested knowledge.
4. Authorization is resolved fresh on every request — including every follow-up turn within an existing Conversation — in Relativity Core, and signed; never re-derived downstream by AIKB or a Surface Adapter, and never satisfied by reusing a prior turn's cached grant. A Conversation's `entitledCollectionIdsAtCreation` (§2.3) is audit metadata describing what was searched historically — it is never reused as a live authorization grant for a new turn.
5. Every Principal-resolution path defaults to deny (§2.1). Only a mapped active Team Member receives normal Group-based access; a verified-but-unmapped internal user receives General-collection-only access solely if the Organization explicitly opts in; provider guests, external/Slack Connect users, deactivated users, and unregistered bot/system identities are never granted access by default.
6. Role governs coarse platform permissions; Group governs knowledge-collection entitlement — the two axes are never conflated.

**Surfaces & Integrations**
7. Every interface implements the Knowledge Surface contract; no interface bypasses `authorize()` or `queryKnowledge()`.
8. No provider-specific logic exists inside AIKB — all provider awareness lives in Relativity's Surface Adapters.
9. Every OAuth provider is configured through the same OAuth Provider/Connection/Token framework — no bespoke per-provider token table.
10. A new Knowledge Surface is addable without modifying the knowledge pipeline, identity model, or authorization logic.
11. Delivery (sending an answer back to a user) is always provider-specific and always lives in the Surface Adapter, never in AIKB.

**Data & Contracts**
12. Every cross-repository request is idempotent, using a caller-supplied idempotency key enforced at the database-constraint level, not just in application logic.
13. Every cross-repository request is signed; no endpoint trusts an unsigned `organizationId` from the caller alone.
14. Conversations and Messages are owned exclusively by AIKB; Relativity stores only the external-thread-to-conversation mapping.
15. Knowledge gaps exist in exactly one system and are created exclusively by the knowledge pipeline itself, never by a Surface Adapter directly.
16. Every answer must be grounded — if no chunk clears the retrieval threshold, the system returns "not found," never a fabricated answer.
17. Citations always reference a real Document/Chunk; no citation is synthesized from the model's own general knowledge.

**Security**
18. Tokens are always encrypted at rest with a rotatable key; no credential is ever stored in plaintext.
19. Every inbound webhook is signature-verified before any other processing occurs, including before it's written to a queue.
20. Restricted collections are never included in a Guest's entitlement, regardless of Group-mapping configuration — enforced as an independent, non-bypassable check.
21. Replay protection (timestamp windows + idempotency keys) is mandatory on every externally-triggered event, not optional per integration.
22. Every request that crosses a trust boundary produces an Audit Event; audit logging is not opt-in per feature.

**Naming & Consistency**
23. One canonical name exists per concept platform-wide; deprecated synonyms (Client, chat "Session," Document Group) are retired from all new code, even where legacy tables keep old names until migrated.
24. Every persisted entity has an explicit, documented lifecycle: who creates it, who updates it, who deletes/soft-deletes it — never left to be inferred from code.
25. Every background event follows the `domain.entity.action` convention, identically on both sides of the repository boundary.

**Extensibility & Maintainability**
26. A capability is built once, in its owning repository, and reused by every surface/integration — never duplicated per interface.
27. Schema changes to shared concepts (Organization, Collection, Conversation) update this domain model document before implementation — the model stays authoritative, not aspirational.
28. Prefer extending an existing entity's optional fields over introducing a new, overlapping entity — collapse near-duplicate concepts (as this document does for Workspace/Provider Account, Session/Conversation) rather than letting them coexist.
29. Every engineering decision favors the platform's multi-year reuse goal over the fastest path for whichever integration is currently being built.
30. When unsure where a capability belongs, ask: does it decide *who can see what* (Relativity), or does it *reason over what's allowed* (AIKB) — never both in the same code path.

## 6. Future validation

| Integration | Role in the model | Model change required? |
|---|---|---|
| **Slack** | Knowledge Surface (conversational) | None — the reference implementation the model is built around. |
| **Teams** | Knowledge Surface (conversational) | None — new Provider Account `accountType=collaborative`, new adapter, same everything else. |
| **Discord** | Knowledge Surface (conversational) | None — same shape as Slack/Teams. |
| **Gmail** | Knowledge Surface (question via email) **and** Connected Source (Sync, for Drive-attached content) | None — the model already separates the Surface role from the Import/Sync role; Gmail can implement either or both against the same OAuth Connection. |
| **Outlook** | Same dual role as Gmail | None — mirrors Gmail via Microsoft Graph. |
| **Google Drive** | Connected Source (Import/Sync only — not a Knowledge Surface, no one "asks a question" through Drive) | None — this is exactly the Knowledge Sync entity's purpose (§2.3). |
| **Dropbox** | Connected Source (Import/Sync) | None. |
| **SharePoint** | Connected Source (Import/Sync) | None — new Provider Account `accountType=file-store`. |
| **OneDrive** | Connected Source (Import/Sync) | None. |
| **Box** | Connected Source (Import/Sync) | None. |
| **Notion** | Connected Source (Import/Sync) | None — Notion pages map to Document the same way a PDF does; parsing logic differs, ownership model doesn't. |
| **Chrome Extension** | Knowledge Surface — thin Portal variant | None — reuses Portal's Supabase session for `identifyUser`. |
| **Desktop App** | Knowledge Surface — thin Public-API-shaped variant | None. |
| **CLI** | Knowledge Surface, Service Account identity | None — this is precisely why Service Account exists as its own identity type. |
| **Public API** | Knowledge Surface, Service Account identity | None — see §4 for the one genuinely new thing (resource-oriented route conventions), which is an API-style decision, not a domain-model change. |
| **Mobile App** | Knowledge Surface — thin variant, User identity via normal login | None. |
| **Voice Assistant** | Knowledge Surface with an added speech-to-text pre-step | None to the domain model — `receiveQuestion()` gains a transcription call (Relativity already does this for Portal voice input today, Phase 1 §10), everything downstream is unchanged. |

**No integration in this list requires a model change.** The two integrations that might *look* like they need one — Gmail/Outlook (both ask questions *and* could feed documents) and the six file-storage providers (which aren't Surfaces at all) — are exactly what motivated separating **Knowledge Surface** (ask role) from **Knowledge Import/Sync** (feed role) as independent, combinable capabilities over the same OAuth Connection (§2.4, §2.3). That separation is the model's answer to "does this force a redesign," and it doesn't.

## 7. How to use this document

- **Before writing a new entity/table:** check §2 for an existing canonical entity first; if it looks new, check §1's overlap table for a reason it might already be covered.
- **Before naming anything** (a route, an env var, a DB column, an event): check §4.
- **Before adding an integration:** check §6 — if it fits an existing role (Surface or Connected Source), no model change is needed, only an adapter.
- **When a design decision feels ambiguous:** apply principle 30 (§5) — identity/authorization questions belong to Relativity, reasoning/retrieval questions belong to AIKB.

This phase ends with the canonical Relativity Systems Domain Model. No implementation, migration, or code change follows automatically — Phase 4 (implementation planning) requires explicit approval to begin.

---

# Phase 4: Slack MVP Implementation Roadmap

**Status:** Implementation planning only. No production code, migrations, package installs, commits, pushes, or environment changes were made in producing this document. Builds directly on the approved Phase 1 (Architecture Discovery), Phase 2 (Platform Architecture), and Phase 3 (Domain Model) sections above.

**Method note:** this plan was produced after re-reading the current state of both repositories (not just Phase 1's findings) — `aikb/routes/slack.js`, `aikb/server.js`, `aikb/config/index.js`, `aikb/routes/knowledge.js`, `aikb/middleware/resolveContext.js`, `aikb/inngest/functions.js`, `Relativity/routes/auth.js`, `Relativity/services/slackService.js`, `Relativity/services/supabaseService.js`, `Relativity/middleware/{apiKey,clientAuth}.js`, `Relativity/config/index.js`, `Relativity/vercel.json`, both `.env.example` files, both `package.json` files, and both repos' git state. Relativity is currently checked out on `feature/slack-integration-phase-1` with an uncommitted `.env.example` edit already staging `SLACK_CLIENT_ID`/`SLACK_CLIENT_SECRET`/`SLACK_SIGNING_SECRET`/`SLACK_REDIRECT_URI`/`SLACK_TOKEN_ENCRYPTION_KEY` — that file was left untouched by this phase; it is prior in-progress work, not something this plan created.

## 1. Scope and non-goals

**In scope:** a concrete, dependency-ordered implementation plan (branches, migrations, tests, deployment order, rollback) that takes both repositories from their current state (Milestones 1–3 already implemented, see their Implementation Notes below) to the Slack MVP described in the objective — one workspace per organization, `@RelativityBot` mentions answered from AIKB's shared, organization-scoped knowledge pipeline (§2.4 — full organization scope by default, with an optional "General"-only restriction available as Milestone 5 if confirmed necessary before launch), encrypted credentials, deduplicated delivery, and Slack-tagged knowledge gaps.

**Out of scope for Phase 4 execution (design only, no code):**
- Direct messages (Milestone 7).
- Fine-grained per-employee authorization / Identity Link / Guest resolution (Milestone 7, Phase 3 §2.1).
- The full Relativity Group/Collection/entitlement model from Phase 3 §2.1–§2.2 — Milestone 5 ships a deliberately smaller, optional stand-in (see §2.5 below) if a "General"-only restriction is confirmed necessary; the full model remains future work regardless.
- Migrating Google Drive/Dropbox onto the new encrypted credential tables.
- KMS-managed envelope encryption, key rotation automation, secrets-manager consolidation (Phase 2 §10/§12 Should-Haves).
- Teams/Gmail/Outlook adapters (Phase 2 §11 future roadmap — the contract is shaped to not block them, but none are built here).
- Refactoring the existing shared `x-api-key` gate on AIKB's non-Slack routes (`/ingest`, `/reindex`, `/documents/:clientId`, etc.) — Milestone 4 adds a **new, additional** minimal signed envelope for Slack-originated traffic only (§4.10), per the objective's explicit instruction not to refactor every AIKB endpoint in this milestone. That remains a tracked but separate hardening item.
- RLS, structured `audit_log`, multi-org users, custom Collection inheritance — Phase 2 §12 Should-Have/Future items, unaffected by this plan.

## 2. Final implementation decisions

These resolve every "recommend and explain" instruction in the objective. Where a decision narrows or simplifies something Phase 2/3 described as the long-term target, that is called out explicitly as a deliberate MVP simplification, not a silent deviation.

### 2.1 Milestone 1 — how to neutralize the legacy AIKB route

**Decision: Option C — return a deliberate `410 Gone` / disabled response**, not immediate deletion (A) and not a runtime feature flag (B).

- **Why not A (delete outright):** deletion with no trace makes it impossible to tell, from the outside, whether a real Slack app is still configured to POST at `aikb/routes/slack.js`'s old URL (Phase 1 found this handler live and signature-verified with its own `SLACK_SIGNING_SECRET`, independent of Relativity's unused one — meaning a real Slack app may exist pointed at it). A silent 404 looks identical to "route never existed" in logs.
- **Why not B (feature flag):** a flag keeps the unsafe code — the SHA1 channel→client-ID hash (`slackChannelToClientId`) and the static `SLACK_BOT_TOKEN` reply path — present and re-enablable by an accidental env var flip. Per Phase 1 §6 Medium, dead-but-present code is already an audit-surface problem; a flag that can silently re-arm a cross-tenant data leak is worse than either removing it or gutting it.
- **Why C:** replace the entire route body with a handler that (a) still returns Slack's `url_verification` challenge if one arrives (harmless, and lets an admin confirm which URL is still wired up), and (b) for every `event_callback`, logs a structured warning (`[slack] legacy endpoint hit — team_id=..., event_type=...`) and returns `410 Gone` with `{ error: 'This Slack integration endpoint has been retired. See Relativity Slack integration.' }`. The unsafe logic (`slackChannelToClientId`, `postSlackMessage` using the global bot token, the direct `searchChunks` call bypassing `/query`) is deleted from the file entirely — not flagged off — so there is no code path left that can leak across tenants. Rollback, if ever needed, is a plain `git revert` of the commit; there is no legitimate reason to want the old behavior back (it was never safe), so no runtime toggle is provided.

**Deployment ordering:** this ships first, alone, with no dependency on anything else in this plan. It is deployed to AIKB (Railway) before any other milestone begins, because it is a pure risk-reduction change (closes a live cross-tenant leak path) that nothing downstream depends on. The 410 response stays in place through every milestone that follows; only after Milestone 4 (renumbered — Relativity's new events endpoint, §2.4 §4.4) is verified working in staging does a final housekeeping step delete the now-inert route file and its `SLACK_BOT_TOKEN`/`SLACK_SIGNING_SECRET` reads from `aikb/config/index.js` — tracked as a small follow-up, not a blocking step. (This paragraph originally referenced the pre-Milestone-1-3 milestone numbering; renumbered here only to avoid misdirecting the next implementation session — no behavioral claim about Milestone 1 itself has changed.)

### 2.2 Milestone 2 — credential storage shape

**Decision: a scoped variant of Option C** — introduce a small, generic `oauth_connections` / `oauth_credentials` pair now, but *not* the full future registry (no `oauth_providers` table, no cross-provider refresh sweep, no migration of Dropbox/Google Drive). This is smaller than Phase 2 §6's eventual target but is shaped so Teams (or a second Slack workspace type) is a new `provider` value later, never a schema rewrite.

Rejected alternatives:
- **A (encrypted columns on existing `oauth_tokens`)** — rejected. `oauth_tokens` today holds Slack/Dropbox/Google Drive rows in one shared table with no column indicating whether a given row's `access_token` is encrypted. Adding encrypted columns there means either a `is_encrypted` flag (ambiguous, easy to get wrong at read time) or a partial migration that leaves plaintext Dropbox/Google rows and encrypted Slack rows physically indistinguishable except by provider — worse for the "no plaintext fallback" requirement than a clean split.
- **B (bespoke `slack_connections` table)** — rejected. Fastest to build, but the objective explicitly says "does not force a rewrite for Teams later," and a Slack-only table with Slack-only column names is exactly what would need re-deriving for Teams.

**Exact schema (new migration, additive only — does not touch `oauth_tokens`):**

```sql
-- Relativity Global DB
CREATE TABLE oauth_connections (
  id                    uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id       uuid NOT NULL REFERENCES clients(id),
  provider              text NOT NULL CHECK (provider IN ('slack')),  -- extend CHECK list when Teams ships
  status                text NOT NULL DEFAULT 'connected'
                          CHECK (status IN ('connected','revoked')),
  external_account_id   text NOT NULL,      -- Slack team_id
  external_account_name text,               -- Slack team name, display only
  bot_user_id           text,               -- Slack bot_user_id, needed to strip @mentions
  scopes_granted        text[] NOT NULL DEFAULT '{}',
  connected_by_member_id uuid REFERENCES client_members(id),
  created_at            timestamptz NOT NULL DEFAULT now(),
  updated_at            timestamptz NOT NULL DEFAULT now(),
  revoked_at            timestamptz
);

-- One active Slack workspace per organization
CREATE UNIQUE INDEX oauth_connections_one_active_per_org
  ON oauth_connections (organization_id, provider)
  WHERE status = 'connected';

-- One active organization per Slack team ID
CREATE UNIQUE INDEX oauth_connections_one_active_per_team
  ON oauth_connections (provider, external_account_id)
  WHERE status = 'connected';

CREATE TABLE oauth_credentials (
  id                    uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  connection_id         uuid NOT NULL REFERENCES oauth_connections(id) ON DELETE CASCADE,
  access_token_ciphertext bytea NOT NULL,
  iv                    bytea NOT NULL,       -- random per encryption, 12 bytes for GCM
  auth_tag              bytea NOT NULL,
  encryption_key_version int NOT NULL DEFAULT 1,
  created_at            timestamptz NOT NULL DEFAULT now(),
  updated_at            timestamptz NOT NULL DEFAULT now()
);
```

No `refresh_token`/`expires_at` columns — Slack bot tokens (`xoxb-…`) issued via OAuth v2 install do not expire under normal operation, so there is nothing to refresh for the MVP. If a future provider needs refresh, add nullable columns then; do not add unused columns now.

**Encryption module (`services/tokenEncryption.js`, new, no new dependency — Node's built-in `crypto`):**
- AES-256-GCM, key = `SLACK_TOKEN_ENCRYPTION_KEY` (64 hex chars → 32 bytes), validated at startup (`require_env` + length/hex-format check, fail fast rather than silently truncating).
- `encrypt(plaintext) -> { ciphertext, iv, authTag, keyVersion }`: fresh `crypto.randomBytes(12)` IV every call.
- `decrypt({ ciphertext, iv, authTag, keyVersion })`: selects the key for `keyVersion` (today, only version 1 exists — the lookup is a `{1: SLACK_TOKEN_ENCRYPTION_KEY}` map so a version 2 key can be added later without changing callers), verifies the auth tag (GCM throws on mismatch — that throw propagates, there is no plaintext fallback).
- Both functions reject non-buffer/malformed input rather than coercing it — corrupted ciphertext must throw, never decrypt to garbage silently.
- Never logged: no `console.log`/`console.error` call in this module or its callers is allowed to include the token, ciphertext, IV, or auth tag — only `connection_id`/`organization_id` for correlation.

**Migration order:**
1. Create `oauth_connections` + `oauth_credentials` (new file, e.g. `20260714_oauth_connections.sql`), additive, no data touched, safe to deploy standalone.
2. Add `services/tokenEncryption.js` and its unit tests (no schema impact).
3. In the **same PR** (Milestone 2), delete existing plaintext Slack rows from `oauth_tokens` (`DELETE FROM oauth_tokens WHERE provider = 'slack'`) — Phase 1 §6/§9 already confirmed these rows are write-only dead data ("nothing in Relativity ever reads this token again"), and Phase 1 §11 Decision 4 already approved discarding the old Slack OAuth flow outright. This is executing that already-approved decision, not a new one. Dropbox/Google Drive rows in `oauth_tokens` are untouched.

**Google Drive / Dropbox migration:** deferred, not part of Phase 4. They keep using the existing plaintext `oauth_tokens` table and `upsertToken`/`services/{dropboxService,googleDriveService}.js` unchanged. Migrating them onto `oauth_connections`/`oauth_credentials` is a Should-Have follow-up (Phase 2 §12) with no Slack MVP dependency.

**Key rotation:** `encryption_key_version` exists from day one so a future rotation is additive (bump the map, background job re-encrypts old rows under the new key, old ciphertext keeps decrypting under its recorded version until re-encrypted). No rotation tooling is built in Phase 4 — the column is the seam, not the mechanism.

**Disconnect/revocation:** `POST /api/integrations/slack/disconnect` (Milestone 3) deletes the `oauth_credentials` row (ciphertext gone immediately) and sets `oauth_connections.status = 'revoked'`, `revoked_at = now()` (row kept for audit/history, per Phase 3 §2.4's "must also revoke at the provider" guidance — that provider-side revocation call, `apps.uninstall` or `auth.revoke`, is a **Should Have** best-effort call attempted in the same request but not required to succeed for the disconnect to complete locally).

**Status API safety:** `GET /api/integrations/slack/status` returns only `{ connected: boolean, workspaceName: string | null, connectedAt: string | null }` — no token, no ciphertext, no IV, ever, in any field or error path.

### 2.3 Milestone 3 — OAuth state and connection rules

- **State token:** `crypto.randomBytes(32).toString('hex')` generated server-side. Only its SHA-256 hash is persisted (objective: "raw state should not be stored when hashing is practical") in a new small `oauth_states` table: `id, state_hash (unique), organization_id, member_id, created_at, expires_at (created_at + 10 min), used_at (nullable)`. The raw token is the state param sent to Slack; Relativity never stores it, only its hash — a leaked DB row cannot be replayed as a live state value.
- Callback hashes the incoming `state` query param, looks up by `state_hash`, rejects if not found, expired, or `used_at IS NOT NULL` (single-use, enforced by setting `used_at` in the same transaction as the lookup). `organization_id` and `member_id` come **only** from this row — never from anything client-supplied — closing the confused-deputy gap in the current `base64(JSON)` state (which is unsigned and technically forgeable, since it's just base64, not a MAC or a server-side lookup).
- **Authorization:** `/start` and `/disconnect` require `clientAuth` (existing middleware) **plus** a role check restricting to `owner`/`admin` (`req.member.role`) — `clientAuth` alone (used today for Dropbox/Google start) only proves org membership, not admin rights; Slack install/uninstall needs the tighter check per the objective.
- **Callback error handling:** Slack's `?error=` query param (e.g. `access_denied`) redirects to `/portal.html?error=slack_denied` (matching the existing Dropbox/Google pattern already in `routes/auth.js`); a `code`/`state` exchange failure or a non-`ok:true` Slack response redirects to `/portal.html?error=slack_failed` — never leaks the raw Slack error string into the redirect URL.
- **Token exchange validation:** after `oauth.v2.access`, explicitly check `data.ok === true`, `data.access_token` starts with `xoxb-`, `data.team?.id` is present, `data.bot_user_id` is present — reject and redirect to `slack_failed` if any are missing, rather than trusting the shape blindly (current `slackService.exchangeCodeForToken` only checks `ok`).
- **One-workspace-per-org / one-org-per-workspace:** enforced at the DB layer by the two partial unique indexes in §2.2, with an app-level pre-check in the callback so a reconnect (same org, new Slack auth) cleanly revokes-then-inserts rather than hitting a raw constraint violation as the user-facing error.
- **Portal UI (`feat/slack-portal-ui`):** one card in the existing integrations section of the portal, following the same visual/data pattern already used for `dropboxConnected`/`googleDriveConnected` in `GET /auth/me`: label "Slack", state "Connected"/"Not connected", workspace name when connected, a "Connect" button (calls `/start`, redirects to the returned `url`) or "Disconnect" button (calls `/disconnect`, confirms first). No analytics, channel picker, or user-mapping UI — matches the objective's explicit exclusion.

### 2.4 Milestone 4 — Slack App Mentions and Grounded AIKB Answers

**Status:** Specification for the next implementation session. Nothing in this section has been built — no code, migration, package, or environment change exists for it yet. This section **replaces** the two milestones originally planned here before Milestones 1–3 were implemented ("Milestone 4 — service-request contract mechanism" and "Milestone 5 — durability shape," which previously occupied this position in the document; their content has been removed rather than kept as a dead draft, since nothing in either was ever implemented and this section supersedes both in full). Building the signed request contract without also building the endpoint that calls it left nothing end-to-end testable, and the Milestones 6/7/8 that followed are renumbered to Milestones 5/6/7 below (§3) now that the old 4 and 5 have merged into one shippable unit. This is a **scoping correction to unimplemented planning**, not a rewrite of completed work — Milestones 1–3 (§2.1–§2.3 above, and their Implementation Notes at the end of this document) are unchanged.

This milestone delivers the platform's **first customer-visible Slack Q&A path**: a user mentions `@RelativityBot`, Relativity securely receives and verifies the event, maps the Slack workspace to exactly one organization, calls AIKB's shared knowledge pipeline, and posts a grounded, cited answer back in the original thread — or a safe fallback if AIKB cannot answer. It is app-mention Q&A, **not** a complete Slack platform; §4.17 below enumerates everything deliberately deferred.

**Re-verified current state (repositories re-read directly on 2026-07-14, current paths — not assumed from Phase 1 or from the superseded original Milestone 4/5 draft this section replaces):**

| Fact | Where confirmed |
|---|---|
| AIKB's legacy `/api/slack/events` still exists but is fully retired: signature-verified, answers `url_verification`, returns `410 Gone` for every `event_callback`, contains no channel-hash tenant derivation and no global bot token reply path | `aikb/routes/slack.js` |
| AIKB `knowledge.js` mounts `router.use(requireApiKey)` (shared `x-api-key`, non-constant-time compare) at the router level, so **every** route including `/query` requires it | `aikb/routes/knowledge.js:21-36` |
| `POST /api/knowledge/query` additionally requires `requireMemberContext` — a **Supabase Auth Bearer JWT** validated against a real `client_members` row | `aikb/middleware/resolveContext.js:22-66` |
| Slack traffic has no browser session and therefore no Supabase Auth JWT — `/query` as it exists today is structurally uncallable by a Slack-originated request | (derived from the two rows above) |
| `knowledge_chat_sessions`, `knowledge_chat_messages`, `knowledge_gaps` all already accept `member_id = NULL` | `aikb/services/supabaseService.js:525,660,731`; `aikb/migrations/003_chat_history.sql`, `004_member_id.sql` |
| No `origin`/`origin_metadata`/`idempotency_key` column exists yet on any AIKB table | `aikb/migrations/001`–`004` (confirmed by inspection, none present) |
| AIKB migrations are numbered `001`–`004` sequentially; the next file is `005_*.sql` | `aikb/migrations/` |
| Relativity migrations are named `{YYYYMMDD}_{description}.sql`; the latest is `20260715_oauth_states.sql` | `Relativity/supabase/migrations/` |
| `oauth_connections` (Relativity Global DB) has `client_id`, `provider`, `external_account_id` (Slack `team_id`), `status` (`active`\|`revoked`\|`expired`\|`error`), `provider_metadata jsonb` (holds `bot_user_id`, `team_name`, etc. — no dedicated `bot_user_id` column) | `Relativity/supabase/migrations/20260714_oauth_connections.sql` |
| `oauthConnectionsService.js` already exports `getActiveConnectionByExternalAccount(provider, externalAccountId)` and `getDecryptedCredentialForConnection(connectionId)` — both are exactly what tenant mapping (§4.6) and token delivery (§4.13) need, and neither requires new code to add | `Relativity/services/oauthConnectionsService.js:151-165,177-194` |
| `config/index.js`'s `slack` block already threads `SLACK_SIGNING_SECRET` through, unread by any code — Milestone 3's note explicitly reserved it for this milestone | `Relativity/config/index.js`; Milestone 3 Implementation Note, "Known limitations" |
| Relativity's `app.js` calls `express.json()` with **no `verify` callback** — no raw request body is currently retained anywhere in the app | `Relativity/app.js:14` |
| `services/slackService.js`'s own file comment states it deliberately excludes Slack Web API message-posting "until the Slack Events milestone" — i.e., this milestone | `Relativity/services/slackService.js:3-5` |
| Slack scopes currently granted are exactly `app_mentions:read` and `chat:write` — sufficient for this milestone; no scope change is required | `Relativity/services/slackService.js:17` (`REQUIRED_SCOPES`) |

None of the rows above were true when the original, superseded Milestone 4/5 draft was written — that draft predates Milestones 2–3's actual implementation and assumed a different starting point. The design below is fitted to what is actually in the repositories today.

#### 4.1 Goal

At the end of Milestone 4, a Slack workspace connected to a Relativity organization (Milestone 3) must be able to:

1. Send an `app_mention` event to `@RelativityBot`.
2. Have Relativity verify the request really came from Slack.
3. Have Relativity map the Slack team to exactly one active organization.
4. Extract one natural-language question from the mention.
5. Send the question through AIKB's shared knowledge-query pipeline — the same retrieval/generation/citation code the portal already uses, not a second implementation.
6. Receive a grounded answer and citations from AIKB, or a knowledge-gap result.
7. Post exactly one reply in the original Slack thread.
8. Receive a safe fallback message when AIKB cannot answer.
9. Never produce a duplicate reply when Slack redelivers the same event.

This is the platform's first customer-visible Slack Q&A milestone — everything before it (Milestones 1–3) was infrastructure with no end-user-visible behavior change.

#### 4.2 Approved ownership (restated, non-negotiable)

| Relativity owns | AIKB owns |
|---|---|
| Slack Events HTTP endpoint | Retrieval |
| Slack HMAC signature verification | Embeddings |
| Slack replay/timestamp validation | Prompt construction |
| Slack `url_verification` challenge | Answer generation |
| Event normalization | Citations |
| Slack team → organization mapping | Conversations |
| Loading/decrypting the org's Slack bot token | Knowledge-gap detection and persistence |
| Provider-specific formatting | Knowledge data |
| Slack Web API delivery (`chat.postMessage`) | Provider-agnostic query behavior |
| Event deduplication at the provider boundary | |
| Safe provider-facing errors | |
| Integration audit metadata | |

**Non-negotiable:** Relativity must never implement its own retrieval, LLM answer generation, citation generation, or knowledge-gap storage for Slack. AIKB must never receive or store Slack bot tokens or Slack client secrets. AIKB remains provider-agnostic — nothing in its request handling may branch on `origin === 'slack'` beyond tagging metadata.

#### 4.3 Exact MVP flow

```
Slack
  |
  v
POST /api/integrations/slack/events                          (Relativity, new route)
  | verify X-Slack-Signature / X-Slack-Request-Timestamp (raw body, timing-safe, 5-min window)
  | url_verification -> respond with challenge, stop here
  | filter: event_callback + event.type === 'app_mention', reject bot_id/subtype/edits/deletes
  | dedupe on Slack event_id (INSERT ... UNIQUE, conflict -> 200 ack, stop, no reprocessing)
  | map event.team_id -> oauth_connections (provider='slack', status='active') -> client_id
  | strip <@BOT_ID> mention, trim, reject empty/oversized question
  | 200 ack to Slack now (this satisfies Slack's response-time budget -- nothing above this
  |   line does LLM work)
  v
POST /api/knowledge/ask                                        (AIKB, new route, service-authenticated)
  | verify signed request, resolve client_id, accept + enqueue (does not wait for the answer)
  v
AIKB Inngest: knowledge/slack.question.requested
  | same retrieval + generation + citation logic /query already uses (shared function, not
  |   duplicated)
  | persists Conversation + Message, evaluates + persists Knowledge Gap if applicable
  v
POST /api/integrations/slack/deliver                            (Relativity, new route, reversed
                                                                   service auth)
  | conditional update: only the first delivery attempt proceeds
  | decrypt the org's Slack bot token (in memory, only here)
  | format answer + citations for Slack
  | chat.postMessage, thread_ts = original event's thread_ts or ts
  | record safe delivery + audit metadata
```

Slack-specific formatting and delivery live only in Relativity, per §4.2. AIKB never calls Slack's API and never sees a bot token.

#### 4.4 Inbound Slack endpoint

`POST /api/integrations/slack/events` — a new route added to the existing `Relativity/routes/integrations/slack.js` router (already mounted at `/api/integrations/slack` in `app.js`), alongside the existing `/start`, `/callback`, `/status`, `/disconnect` routes from Milestone 3.

Must support: Slack's `url_verification` challenge; `event_callback` events where `event.type === 'app_mention'`.

Must safely ignore (200 ack, no processing, no AIKB call): bot-generated messages (`event.bot_id` present); events generated by RelativityBot itself (`event.user === connection.provider_metadata.bot_user_id`); any event type other than `app_mention`; edited-message and deleted-message subtypes; duplicate deliveries (same `event_id`); malformed events (missing `team_id`, missing `event`, unparseable body).

Explicitly not in this milestone: direct-message handling (no `im`-scoped event is subscribed to; §4.17). AIKB's retired endpoint (`aikb/routes/slack.js`) is not re-enabled or extended — it stays exactly as Milestone 1 left it.

**Raw-body handling (concrete implementation concern, not optional):** Relativity's `app.js` today calls `app.use(express.json())` with no `verify` option (see the "Re-verified current state" table above) — no raw bytes are retained anywhere in the app. Slack signature verification is computed over the *exact* raw request bytes, not a re-serialized `JSON.stringify(req.body)` (re-serialization can silently change key order/whitespace and invalidate a legitimate signature, or worse, validate a tampered one if re-serialization happens to normalize it back). The implementer must add a `verify: (req, res, buf) => { req.rawBody = buf; }` callback to the existing global `express.json()` call (cheapest, lowest-risk option — it affects no other route's behavior, since every other route already ignores `req.rawBody`) before writing the signature-verification code that depends on it.

#### 4.5 Slack security requirements

- Verify `X-Slack-Signature` using `SLACK_SIGNING_SECRET` (already wired through `config/index.js`, unused since Milestone 3 — this is what it was reserved for).
- Verify `X-Slack-Request-Timestamp`; reject requests outside a five-minute replay window.
- Compute the signature over the exact raw request body (`req.rawBody`, §4.4) — never a re-parsed/re-serialized body.
- Use `crypto.timingSafeEqual` for the signature comparison, not `===` (mirrors the already-correct pattern in AIKB's retired `routes/slack.js:47`, which Relativity's new verifier should be modeled on).
- Handle missing or malformed signature/timestamp headers as a rejection, not a crash.
- Never trust `team_id`, `channel_id`, `user_id`, `event_id`, or `thread_ts` until the request's signature has already been verified.
- Never identify an organization from a channel-id hash or any browser/client-supplied identifier — team→organization mapping is a trusted database lookup only (§4.6), exactly the failure mode Milestone 1 retired.
- Never log the raw signature, bot token, client secret, authorization code, encrypted envelope, or full event body. (AIKB's retired handler's `logRetiredEventCallback` — team-id-hash-only, no message text — is the right logging shape to carry forward.)
- Never expose internal database identifiers (`client_id`, `connection_id`, session/message ids) in a Slack-facing reply.
- Return safe HTTP responses on every rejection path — no provider secret, stack trace, or internal error string in a body Slack (or its logs) will see.

#### 4.6 Tenant mapping

- Read `team_id` from the **verified** Slack event (post-signature-check only).
- Look up the active connection via the already-existing `oauthConnectionsService.getActiveConnectionByExternalAccount('slack', team_id)` (`Relativity/services/oauthConnectionsService.js:151`) — no new lookup function is needed, this was built in Milestone 2 for exactly this purpose and has not been called yet.
- That row's `client_id` is the resolved organization. Require the organization itself to remain active (existing `requireActiveClient`-equivalent check, mirroring the pattern already used in AIKB's `/query` and in Relativity's own client-scoped routes).
- No match (unknown `team_id`), a `revoked`/`expired`/`error` status, or an inactive organization → 200 ack to Slack (it is not an error Slack should retry), no AIKB call, a structured warning log, no reply posted.
- Never derive organization identity from `channel_id` — only `team_id` via the trusted `oauth_connections` row.
- Never accept an organization/client id supplied anywhere in the Slack payload itself — the payload has no such field to trust in the first place, and none should ever be added.
- Never fall back to a default/global bot token — the token used for delivery (§4.13) always comes from the specific connection resolved here.

**Invariant (already enforced at the database layer, not just in application code):** the partial unique indexes `uq_oauth_connections_active_per_client_provider` and `uq_oauth_connections_active_external_account` (`20260714_oauth_connections.sql`) already guarantee one active Slack connection per organization and one active organization per Slack team — Milestone 4 relies on, and must not weaken, that existing guarantee.

#### 4.7 Event deduplication

Slack `event_id` is the idempotency key. New, additive Relativity_Global table (exact filename TBD by the implementer following the existing `{YYYYMMDD}_{description}.sql` convention — the next unused date after `20260715_oauth_states.sql`, e.g. `slack_event_log`):

```sql
CREATE TABLE slack_event_log (
  id                uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  provider          text NOT NULL DEFAULT 'slack',
  external_event_id text NOT NULL,          -- Slack event_id
  client_id         uuid NOT NULL REFERENCES clients(id),
  connection_id     uuid NOT NULL REFERENCES oauth_connections(id),
  event_type        text NOT NULL,          -- 'app_mention'
  channel_id        text NOT NULL,
  event_ts          text NOT NULL,
  thread_ts         text,                   -- nullable: absent when the mention itself starts the thread
  question          text,
  idempotency_key   text NOT NULL,          -- derived from external_event_id, echoed to AIKB
  status            text NOT NULL DEFAULT 'received'
                       CHECK (status IN ('received','enqueued','answered','delivered','failed')),
  attempt_count     integer NOT NULL DEFAULT 0,
  error_code        text,                   -- safe internal code only, never a raw provider error
  received_at       timestamptz NOT NULL DEFAULT now(),
  processing_started_at timestamptz,
  completed_at      timestamptz,
  failed_at         timestamptz,
  UNIQUE (provider, external_event_id)
);
```

Behavior: first delivery inserts and proceeds; a repeated delivery hits the unique-constraint conflict and returns its existing row's status (200 ack, no second AIKB call, no second reply); concurrent duplicate deliveries race on the same DB constraint, so only one ever wins the insert; `failed` rows may be retried by the sweep (§4.8) according to `status`/`attempt_count`, but a `delivered` row is never reprocessed under any retry path. Prefer this as an additive migration in `Relativity_Global` (per the objective) — it is not executed as part of producing this specification.

#### 4.8 Acknowledgement and processing model

Slack must receive `200` quickly. Two phases, deliberately not one:

**A. Provider acknowledgement** (inside the `/events` request): verify signature/timestamp → handle `url_verification` → filter event type/self/bot → insert the `slack_event_log` row (dedup) → resolve tenant (§4.6) → extract question (§4.11) → respond `200`.

**B. Knowledge processing** (after acknowledgement, on Relativity's next request from AIKB, not inside the same Express request): AIKB does the retrieval/generation work and calls back when done.

**Why not a single synchronous request that also waits for AIKB's full answer:** Relativity is deployed as a Vercel serverless function (Phase 1 §2, unchanged). Two hard constraints combine here: (1) Slack expects an HTTP response quickly, and (2) a Vercel function that has already returned a response cannot be relied upon to keep running an un-awaited background promise to completion — there is no `waitUntil`-equivalent in this codebase today. So the *only* safe way to "ack fast, then do slow work" is to hand the slow work to something that is still running after Relativity's response — which is AIKB's own already-existing Inngest process on Railway (an always-on host), not a fire-and-forget promise inside the Express handler.

Concretely: Relativity's `/events` handler makes one synchronous, *fast* call to AIKB's new `POST /api/knowledge/ask` before returning its own response to Slack — but that AIKB call only needs to accept-and-enqueue the question into AIKB's existing Inngest pipeline, not compute the answer. This keeps the whole `/events` request short (dedup insert + one fast HTTP round trip), satisfies Slack's ack budget, and does not depend on any code executing after Relativity's response is sent. The implementer should confirm this against observed `/query` latency in practice before assuming the accept-and-enqueue leg is fast enough in the target deployment — if AIKB's enqueue acknowledgement itself is not reliably fast, persist the `slack_event_log` row and return Slack's ack *before* attempting the AIKB call at all, and let the sweep (below) make the first delivery attempt instead.

AIKB's Inngest function (`knowledge/slack.question.requested`, following the existing `knowledge/document.ingest` naming and `step.run` pattern in `aikb/inngest/functions.js`) performs retrieval + generation using the same internal logic `/query` already runs (§4.9), then calls back to a new `POST /api/integrations/slack/deliver` on Relativity with the result. Relativity's sweep (a new Vercel Cron entry, since none exists in `vercel.json` today) periodically retries `slack_event_log` rows stuck in `received`/`enqueued` past a timeout, with capped attempts, to recover from a transient AIKB outage without ever duplicating a `delivered` row — this is the retry backstop the objective requires; it is not a second job queue, it is the same table from §4.7 read again on a timer.

This design does not duplicate AIKB's durable-execution mechanism (Inngest stays exactly where Phase 2 §9 put it), does not introduce an unmanaged background promise in a Vercel function, and keeps all Slack-specific code (verification, mapping, formatting, delivery) inside Relativity.

#### 4.9 AIKB contract

**Do not build a second Slack-only retrieval path.** AIKB's `/query` handler (`aikb/routes/knowledge.js:293-496`) already contains the exact pipeline this milestone needs — intent classification, retrieval-query rewriting, `searchChunksWithTitleBoost`, `generateRagAnswer`, gap detection, message persistence. The only reason Slack traffic cannot call `/query` directly is its auth gate (`requireMemberContext` demands a Supabase Auth JWT, see the "Re-verified current state" table above) — not its logic.

**Minimal extension, not a rewrite:** refactor `/query`'s handler body (lines 293–496) into a shared internal function (e.g. `runKnowledgeQuery({ clientId, question, sessionId, memberId, origin, originMetadata, idempotencyKey })`) called by both the existing `/query` route (`origin: 'portal'`, `memberId` from `requireMemberContext`) and a new `POST /api/knowledge/ask` route (`origin: 'slack'`, `memberId: null` — already a supported value everywhere it's written, per the "Re-verified current state" table). `/query`'s request/response shape and its portal callers are untouched.

Normalized request to `/ask` includes at minimum: `clientId`, `question`, `origin: 'slack'`, `idempotencyKey` (derived from Slack `event_id`), and Slack origin metadata (`teamId`, `channelId`, `threadTs` or message `ts`, `eventId`). Response includes at minimum: `answer`, `sources`/citations, a conversation/session identifier, `isKnowledgeGap`, and a safe gap reason when applicable — this is exactly `/query`'s existing response shape (`answer, sources, sessionId, isKnowledgeGap, gapReason`), reused as-is.

Retrieval scope for this milestone is **the same as the portal today** — every document belonging to the organization, `client_id`-scoped exactly as `/query` already enforces via `searchChunksWithTitleBoost`/`match_knowledge_chunks`. This milestone does **not** add a "General"-only collection restriction as an automatic part of Milestone 4: Phase 3's Collection/Group model does not exist yet, and building a bespoke stand-in unconditionally is exactly the kind of "entire future groups/permissions system" the objective says not to build before the first bot test. **This is a deliberate scope decision that needs confirmation before coding**, flagged again in §2.5 below: shipping Milestone 4 means any Slack workspace member who can mention the bot can retrieve anything the portal can already retrieve for that organization — no narrower and no broader than today's portal access, but also not scoped down to a subset. If that is not acceptable, the "General"-only restriction (§2.5, renumbered from the original Milestone 6) must land *before* Milestone 4 is exposed to real Slack traffic, not after.

Do not call AIKB's retired `/api/slack` endpoint. Do not bypass `/query`'s existing `client_id` scoping.

#### 4.10 Authentication between Relativity and AIKB

**Honest current state:** there is no signed request contract between the two repositories today. Every route in `aikb/routes/knowledge.js` — including `/query` — sits behind one static, shared, non-expiring `x-api-key` (`requireApiKey`, non-constant-time comparison, `aikb/routes/knowledge.js:21-36`), and `/query` additionally requires a human Supabase Auth JWT that Slack traffic cannot produce. Phase 2 §10 and Phase 3 principle 13 describe a future signed `ServiceRequest` envelope (organization, principal, entitled collections, origin, idempotency key, all signed) as the long-term target — **that full platform is not implemented, and this milestone does not implement it either.** Claiming otherwise would misdirect the next implementation session into believing infrastructure exists that does not.

**What this milestone actually needs, and no more:** a way for AIKB to trust a per-request `clientId` and `idempotencyKey` from a caller that is not a human with a JWT. Reusing the shared `x-api-key` alone is not sufficient for this one new path, specifically because it is the *first* machine-to-machine (non-human) caller AIKB will ever trust with a client-scoped write — every existing `x-api-key`-only route already has this same weakness (Phase 1 §6 Critical), but refactoring those is explicitly out of scope for this milestone (§1 non-goals) and remains a separate, tracked hardening item.

**Decision:** add one small, additive HMAC-signed envelope, scoped **only** to the new `/ask` route (and its `/deliver` callback, signature reversed) — not a rewrite of the shared `x-api-key`, not the full future `ServiceRequest` envelope (no `entitledCollectionIds`, no multi-origin principal registry, no asymmetric signing):

```
{
  requestId: uuid,
  issuedAt: ISO8601,
  expiresAt: ISO8601,        // issuedAt + 60s
  clientId: uuid,
  idempotencyKey: string,    // derived from Slack event_id
  signature: hex-HMAC-SHA256 // over requestId.issuedAt.expiresAt.clientId.idempotencyKey.sha256(body)
}
```

New env var: `SERVICE_REQUEST_SIGNING_SECRET`, shared between Relativity and AIKB — distinct from `API_KEY` (unchanged, still gates the routes this milestone doesn't touch), `SLACK_SIGNING_SECRET` (Slack↔Relativity only), and `INTEGRATION_CREDENTIAL_ENCRYPTION_KEY` (credential-at-rest only). `POST /api/knowledge/ask` sits behind both the existing router-level `requireApiKey` (defense in depth, unchanged) and a new `requireServiceRequest` middleware that additionally verifies the signature (`crypto.timingSafeEqual`), rejects an expired `expiresAt`, and resolves `clientId` via the existing `requireActiveClient`-equivalent check. `POST /api/integrations/slack/deliver` on Relativity verifies the same envelope shape in reverse (AIKB signs, Relativity verifies), so a request claiming to be an AIKB callback cannot be forged by an arbitrary caller either.

This is a temporary, narrowly-scoped compatibility layer, isolated behind `requireServiceRequest` — it does not touch `/query`, `/ingest`, `/reindex`, or any other existing route, and it does not attempt to solve collection entitlement (there is none to sign yet, §4.9). The gap to the full future `ServiceRequest` platform (multi-origin principal resolution, signed `entitledCollectionIds`, contract versioning) remains open and is not silently closed by this milestone.

#### 4.11 Question extraction

For `app_mention`: strip the leading `<@BOT_ID>` mention token (matching `oauth_connections.provider_metadata.bot_user_id` for the resolved connection, §4.6) from the event text; trim whitespace; reject an empty (post-strip) question without calling AIKB; impose a maximum length (e.g. 2000 characters, configurable) and reject oversized questions the same way; preserve ordinary punctuation; treat Slack markup/formatting inside the text as inert data, never as instructions to the model (consistent with Phase 2 §10's prompt-injection demarcation principle); never forward the entire raw Slack event into the AIKB request — only the extracted, normalized question plus the narrow origin metadata listed in §4.9.

Safe empty-question response (sent directly by Relativity, no AIKB call): *"Please include a question after mentioning me."*

#### 4.12 Thread behavior

Always reply in a thread. If the mention occurred inside an existing thread, reply using its `thread_ts`; otherwise use the mention event's own `ts` as the new thread root. Post exactly one main answer per event. Do not implement conversation memory for unmentioned thread replies, and do not subscribe to `message.channels` or any broad message-history event — this milestone is app-mention-initiated only, matching the exact bot event subscribed in §4.16.

#### 4.13 Slack delivery

A new, focused delivery service (e.g. `Relativity/services/slackDeliveryService.js` — kept separate from `services/slackService.js`, which is OAuth-only by its own existing file comment, and separate from anything AIKB-facing): retrieve the active Slack connection for the resolved organization; call `oauthConnectionsService.getDecryptedCredentialForConnection(connectionId)` (already exists, see the "Re-verified current state" table) to decrypt the bot token in memory, immediately before use; reject a revoked/inactive connection; call Slack's `chat.postMessage` with the resolved `channel` and `thread_ts`; apply a request timeout; reject a non-2xx HTTP response, invalid JSON, or a body with `ok: false`; map any Slack-side failure to a safe internal error code; never log the token; never send the token to AIKB; never fall back to a global/static bot token. This service only performs delivery — it does not call AIKB and does not implement retrieval logic.

#### 4.14 Answer and citation formatting

```
<answer text>

Sources:
* <document title>
* <document title>
```

Only human-readable citation fields (document title, optional page/section) are ever included; duplicate source titles are deduplicated; the number of displayed sources is capped to a small number (e.g. 5); internal document ids, chunk ids, storage paths, signed URLs, database ids, embeddings, and raw metadata are never included. Respect Slack's message-size limits, truncating safely if a rendered answer would exceed them. Plain `mrkdwn` is sufficient for this milestone — no Block Kit unless plain text genuinely cannot express the answer.

Knowledge-gap fallback (AIKB-determined, not decided by Relativity): *"I couldn't find that information in your organization's knowledge base."* AIKB remains the sole owner of gap detection and persistence (§4.2) — Relativity never writes a second gap record for the same event.

#### 4.15 Safe error behavior

| Condition | User-facing Slack reply |
|---|---|
| Empty question after stripping the mention | "Please include a question after mentioning me." |
| Workspace not connected / connection revoked | (silent 200 ack, no reply — §4.6; there is no member to notify and no channel context implies who should see an error) |
| AIKB unavailable, timed out, or returned a malformed response | "I couldn't complete that request right now. Please try again shortly." |
| Slack delivery itself fails (posted after the fact, logged only) | (no further Slack reply possible — logged internally with a safe error code) |
| AIKB reports a knowledge gap / no relevant content | "I couldn't find that information in your organization's knowledge base." |

Never reveal HTTP status codes, database errors, Supabase details, AIKB URLs, API keys, OAuth connection ids, stack traces, or raw Slack provider error bodies in any reply or log line reachable outside the two repositories' own internal logs.

#### 4.16 Audit, observability, and Slack app configuration

**Audit/observability:** structured, metadata-only log lines (no message text, no answer text, no token, no signature, no raw body, no encrypted credential, no client secret, no full AIKB response) for: Slack event received; signature verified or rejected; event deduplicated; workspace mapping resolved or rejected; AIKB request started; AIKB request completed or failed; Slack delivery started; Slack delivery completed or failed. Safe fields: `provider`, `event_id`, `event_type`, `client_id`, `connection_id` (internal logs only), duration, and a safe outcome code. This is metadata-only logging in the spirit of Phase 2 §10's `audit_log` design — it does not require building that full structured `audit_log` table in this milestone, only following its same never-log-content discipline.

**Slack app configuration (manual, post-deployment):**
- **Event Subscriptions Request URL:** `https://relativitysystems.ai/api/integrations/slack/events` — do not enable Event Subscriptions until this endpoint is deployed and can pass Slack's URL verification challenge; enabling it against a not-yet-deployed URL will fail verification and Slack will not save the subscription.
- **Bot event to subscribe:** `app_mention` only.
- **Bot scopes:** unchanged — `app_mentions:read`, `chat:write` (already granted per Milestone 3; this milestone adds no new scope). Do not add `incoming-webhook`, `channels:history`, `groups:history`, `im:history`, `mpim:history`, `users:read`, or `users:read.email`.
- **Reinstall:** Slack's own app-management model treats OAuth scope grants and Event Subscriptions as separate configuration axes — subscribing to a bot event does not itself change what a workspace has already authorized, so a reinstall should not be required *solely* to add the `app_mention` event subscription when no scope is changing. This expectation should be **confirmed against the actual Slack app dashboard behavior observed at implementation time** (Slack's UI has, in some configuration-change flows historically, prompted for reinstall) and the true observed behavior recorded in that milestone's Implementation Note — do not carry this paragraph forward as fact without that confirmation.

#### 4.17 MVP constraints / deferred features

Milestone 4 must **not** implement: Slack direct messages; slash commands; Slack Home tab; interactive buttons; modals; reactions; message editing; automatic replies to ordinary channel messages; message-history or channel-history scopes; direct-message or user-profile scopes; Slack user → Relativity member identity mapping; employee-level or collection-based permissions; per-channel collection mappings; streaming answers; typing indicators; file uploads or Slack file ingestion; conversation memory for unmentioned thread replies; broad channel monitoring; scheduled Slack summaries; an administrative Slack configuration UI; analytics dashboards; multiple bot personas; or any Teams/Gmail/Outlook work.

**Milestone 4 prioritizes one reliable, secure app-mention Q&A path over feature completeness.**

#### 4.18 Test requirements

`node:test`, no real Slack/OpenAI/AIKB/Supabase network calls, matching the existing convention in both repos.

- **Signature verification:** valid accepted; invalid rejected; missing header rejected; stale timestamp rejected; malformed timestamp rejected; raw-body verification preserved through the `express.json({ verify })` change; timing-safe comparison used.
- **URL verification:** valid challenge returned; invalid signature rejects the challenge request too (verification happens before any type check).
- **Event filtering:** `app_mention` accepted; unsupported event types ignored; bot-originated and self-originated (`bot_user_id`) events ignored; empty-after-strip mention handled without an AIKB call; malformed payload rejected safely.
- **Tenant mapping:** active workspace resolves to the correct organization; unknown workspace rejected safely (200 ack, no processing); revoked connection rejected; inactive organization rejected; no channel-id fallback path exists; no client/organization id from the Slack payload is ever trusted.
- **Deduplication:** first event accepted; repeated event does not call AIKB twice or post twice; concurrent duplicate inserts are safe (unique constraint); a `delivered` row is never reprocessed by the sweep.
- **AIKB (`/ask`):** normalized request reaches the shared `runKnowledgeQuery` function with `origin: 'slack'`; `idempotencyKey` derived from `event_id`; `clientId` comes only from the trusted service-request envelope, never the request body directly; Slack token never appears anywhere in the AIKB request; malformed AIKB response and timeout both handled by Relativity's delivery path; knowledge-gap result produces the safe fallback; existing `/query` regression suite passes unmodified (proves this milestone didn't touch the portal path).
- **Delivery:** active encrypted credential retrieved and decrypted only server-side; `chat.postMessage` called with the correct channel and `thread_ts`; `ok: false` handled; non-2xx handled; timeout handled; token never appears in a response or log; a revoked credential is never used for delivery.
- **Formatting:** answer and citations rendered; duplicate citations removed; no internal id/path/URL ever appears in a formatted reply; long answers handled safely; the no-answer fallback renders correctly.
- **Regression:** OAuth connect/disconnect/status (Milestone 3) still work; Google Drive and Dropbox behavior unchanged; AIKB's retired Slack endpoint remains retired (still returns `410`); full Relativity test suite passes; full AIKB test suite passes (AIKB is modified in this milestone, unlike Milestone 3).

#### 4.19 Migration expectations

Expect one additive Relativity_Global migration (`slack_event_log`, §4.7) and one additive AIKB migration (the next sequential file after `004_member_id.sql`, i.e. `005_*.sql`, adding nullable `origin`/`origin_metadata`/`idempotency_key` columns — unique, nullable — to `knowledge_gaps` and to `knowledge_chat_sessions`, so AIKB can independently guard against reprocessing the same Slack event even if Relativity's own dedup were ever bypassed). Neither migration was executed in producing this specification. The implementer must confirm the exact filenames at the time of coding (following each repo's existing naming convention, confirmed above) rather than trusting a name guessed here, provide the tables/functions/indexes actually added, the manual execution order, verification SQL, rollback considerations, and confirmation that neither migration was run automatically. Do not modify the already-applied Milestone 2/3 migrations.

#### 4.20 Success criteria (definition of done)

Milestone 4 is complete only when:

1. Slack Event Subscriptions accepts the deployed request URL.
2. A connected workspace can mention `@RelativityBot`.
3. Relativity verifies the request signature and timestamp correctly.
4. The workspace maps to the correct organization and no other.
5. Exactly one AIKB request is made per Slack event.
6. AIKB answers using the shared `/query` pipeline logic (`runKnowledgeQuery`), not a duplicate implementation.
7. A grounded answer appears in the original Slack thread.
8. Safe citations appear when available.
9. A no-answer result produces the approved fallback.
10. Knowledge-gap detection and persistence remain owned exclusively by AIKB.
11. Duplicate Slack deliveries never produce duplicate replies.
12. Revoked Slack connections cannot be used for delivery.
13. No Slack credential ever reaches AIKB.
14. Cross-organization isolation holds (org A's Slack workspace cannot retrieve org B's documents).
15. All automated tests in §4.18 pass, including full regression on both repos.
16. Connect, disconnect, reconnect, and status (Milestone 3) still work unmodified.

#### 4.21 Known limitations after Milestone 4

Working: app mentions, one-shot questions, grounded AIKB answers, citations, threaded replies.

Still intentionally absent: direct messages; unmentioned follow-up messages; Slack user identity linking; per-user or per-channel collection authorization; advanced thread continuity; interactive actions; file ingestion from Slack; streaming; the full future signed `ServiceRequest` envelope (§4.10); and any generalized multi-provider event infrastructure beyond what this milestone required.

#### 4.22 Ordered implementation plan

1. Inspect current Relativity and AIKB Slack/query code (re-confirm the "Re-verified current state" table above still holds at coding time).
2. Confirm the current cross-repository query contract (`/query`'s exact request/response shape) before refactoring it into `runKnowledgeQuery`.
3. Design the minimal event-deduplication schema (§4.7).
4. Add raw-body Slack signature verification to Relativity (`express.json({ verify })`, §4.4).
5. Add `POST /api/integrations/slack/events` to the existing Slack integration router.
6. Add `url_verification` support.
7. Add `app_mention` filtering and question normalization (§4.11).
8. Add trusted workspace→organization resolution using the existing `getActiveConnectionByExternalAccount` (§4.6).
9. Add the acknowledgement/durable-processing handoff (§4.8).
10. Refactor `/query`'s handler into `runKnowledgeQuery` and add `POST /api/knowledge/ask` calling it with `origin: 'slack'` (§4.9, §4.10).
11. Add the encrypted-token Slack delivery service (§4.13).
12. Add thread-reply and safe citation formatting (§4.14).
13. Add structured, metadata-only observability (§4.16).
14. Add the full test suite (§4.18).
15. Run all Relativity tests.
16. Run all AIKB tests.
17. Update this report with the actual Milestone 4 Implementation Note, following the same format as Milestones 2 and 3.
18. Provide the manual Supabase, Vercel, Railway, and Slack dashboard steps actually required.
19. Do not commit or push until reviewed.

### 2.5 Milestone 5 — company-wide knowledge scope (deferred hardening, not a Milestone 4 blocker)

**Renumbered from the original Milestone 6.** Because Milestone 4 (§2.4) deliberately uses the same full-organization retrieval scope the portal already has (§4.9) rather than waiting on a collection-restriction stand-in, this milestone is now optional hardening that can land before or after Milestone 4 ships, not a hard dependency of it. If the company-wide retrieval scope in §4.9 is judged unacceptable before Milestone 4 goes live with real Slack traffic, this milestone must land first — that decision needs explicit confirmation, not a default assumption either way.

**Decision: the "safer alternative" branch, not a full Relativity Collection entity.** Building Phase 3 §2.1–§2.2's Group/Collection/entitlement model (new Relativity tables, admin UI for group membership, signed `entitledCollectionIds`) is exactly the "entire future groups/permissions system" the objective says not to build before the first bot test. Instead:

- **Owner of the definition:** AIKB, temporarily — a single hardcoded, validated value `'general'` on a new `collection` text column on `knowledge_documents` (`CHECK (collection IS NULL OR collection = 'general')`, default `NULL`). This is a **deliberate, narrower simplification than Phase 3's already-flat Collection model** (Phase 3 put Collection *definition* in Relativity even in its flattest form) — flagged as a decision that should be confirmed before coding, since it's a real, if small, divergence from the approved Phase 3 domain model, chosen only because the alternative (building Relativity-side Collection/Group tables with no other consumer yet) is disproportionate to "ship a company-wide Slack bot answering from opted-in documents."
- **Validation:** the `CHECK` constraint is the entire validation — no separate registry table, no sync events (those are Phase 3 §2.2's answer to a *multi-collection*, *Relativity-owned-definition* world; a single hardcoded literal needs neither).
- **Assignment:** a new `PATCH /api/knowledge/document/:id/collection` route in AIKB (body: `{ clientId, collection: 'general' | null }`), gated the same way the existing management routes are (`requireApiKey`, called by Relativity's admin-only proxy — Relativity resolves the org/admin check server-side, exactly like every other admin action, then forwards with the shared `x-api-key`, unchanged from today's pattern for this one narrow new route). A corresponding small Relativity proxy route + portal UI toggle ("Include in Slack") on the existing document list.
- **Existing documents:** `collection` defaults `NULL` on every existing row — nothing becomes retroactively Slack-visible; an admin must explicitly flip each document.
- **Enforcement:** AIKB's `/ask` handler (Milestone 4, `runKnowledgeQuery`), when a `general`-only mode is signaled, adds `AND collection = 'general'` to the same retrieval RPC call that already filters `client_id` — inside the SQL, before any chunk reaches the LLM prompt.
- **Portal/`/query` behavior:** completely unchanged — the `collection` column is only ever read by the `/ask` path; the existing portal `/query` route continues to search every document for the client, exactly as today.
- **Admin UI:** one toggle per document row in the existing list (label: "Include in Slack"), plus an empty-state string when zero documents are opted in.
- **Zero documents opted in:** `/ask` returns the existing "no relevant information found" no-chunks-found path (a legitimate signal that the org hasn't opted anything in yet, tagged `origin: slack`, not a bug).

### 2.6 Milestones 4 & 6 — a minimal schema dependency, not a reordering

**Renumbered from the original "Milestones 4 & 7."** Milestone 4's Slack event handler needs *some* idempotent-gap-creation support to exist before it can safely go live (the objective requires "duplicate events do not create duplicate answers" and "unsupported questions can become knowledge gaps"). Rather than pulling the fuller gap/conversation-metadata milestone forward in its entirety, the **minimal slice** — the `origin`/`originMetadata`/`idempotencyKey` columns and the unique constraint on `knowledge_gaps` (and, per §4.19, on `knowledge_chat_sessions`) — ships as part of Milestone 4's AIKB branch, since `/ask` cannot be written correctly without them (§4.19). The **fuller** scope (system-vs-user `reportedBy` distinction beyond the default, richer `originMetadata` shape, any conversation-level polish) ships in its own branch afterward, renumbered here as **Milestone 6**. This is a scoping split within the milestone order, not a reordering of the security-dependency chain.

## 3. Milestone order (confirmed, renumbered after §2.4's Milestone 4/5 merge)

1. Neutralize the legacy AIKB Slack route (§2.1) — no dependency. **Done.**
2. Encrypted credential storage in Relativity (§2.2) — no dependency. **Done.**
3. Slack OAuth connection flow in Relativity (§2.3) — depends on 2. **Done.**
4. Slack app mentions and grounded AIKB answers (§2.4) — depends on 2 and 3; uses full-organization retrieval (no dependency on 5). **Done — implemented, committed to `main` in both repositories, deployed, and verified end-to-end against a real Slack workspace in production on 2026-07-16. See the Milestone 4 Implementation Note and its "Production deployment (2026-07-16)" addendum below for what changed between the spec and the shipped system.**
5. Company-wide knowledge scope / "General"-only restriction (§2.5) — depends on 4's `/ask` route existing to filter; optional hardening, not a hard blocker of 4 (§2.5, §4.9's confirmation-needed callout).
6. Full knowledge-gap and conversation metadata (§2.6) — depends on 4's minimal schema slice; can land after 4 without blocking it.
7. Direct messages + employee-level authorization — explicitly deferred, no Phase 4 work.

## 4. Repository ownership per milestone

| Milestone | Relativity | AIKB |
|---|---|---|
| 1. Neutralize legacy route | — | Owns entirely |
| 2. Encrypted credentials | Owns entirely (Global DB) | — |
| 3. Slack OAuth | Owns entirely | — |
| 4. Slack app mentions + grounded answers | Owns events endpoint, dedup log, tenant mapping, delivery, `/deliver` callback, signing client | Owns `runKnowledgeQuery` refactor, `/ask` route, `requireServiceRequest` verification, Inngest handoff |
| 5. Knowledge scope | Small proxy route + portal toggle | Owns `collection` column, retrieval filter |
| 6. Gap/conversation metadata | — | Owns entirely (`knowledge_gaps`, chat sessions) |
| 7. DMs + fine-grained auth | Deferred | Deferred |

## 5. Database migration plan

**Relativity (Global Supabase project):**
1. `20260714_oauth_connections.sql` — `oauth_connections`, `oauth_credentials`, both partial unique indexes (§2.2). **Applied in spirit (Milestone 2 branch); not executed against Supabase.**
2. `20260715_oauth_states.sql` — `oauth_states` table (§2.3). **Applied in spirit (Milestone 3 branch); not executed against Supabase.**
3. New, Milestone 4: `slack_event_log` table (§4.7) — next unused `{YYYYMMDD}_{description}.sql` date after `20260715_oauth_states.sql`. Additive. Not yet written.

**AIKB (AIKB Supabase project):**
1. `001_knowledge_base_schema.sql` → `004_member_id.sql` — existing, unchanged.
2. New, Milestone 4: next sequential `005_*.sql` — adds nullable `origin`/`origin_metadata`/`idempotency_key` (unique) to `knowledge_gaps` and to `knowledge_chat_sessions` (§4.19). Additive. Not yet written.
3. New, Milestone 5 (optional hardening, not a Milestone 4 dependency): next sequential `006_*.sql` — `collection text CHECK (collection IS NULL OR collection = 'general') DEFAULT NULL` on `knowledge_documents` (§2.5). Additive.

No migration in this plan alters an existing column's type, drops a column, or touches `knowledge_chunks`/`match_knowledge_chunks` beyond the retrieval-time `WHERE` clause added in application code (Milestone 5) — the RPC signature itself is unchanged, only the calling code's filter arguments. Milestone 4 does not require a `service_request_log` table (no separate AIKB-side idempotency store beyond the `knowledge_gaps`/`knowledge_chat_sessions` unique constraints already listed) — kept intentionally minimal per §2.4's "no more than this milestone needs" framing (§4.10).

## 6. Cross-repository deployment order

1. **AIKB** `chore/disable-legacy-slack` (§2.1) — deployed, first. **Done.**
2. **Relativity** `feat/encrypted-integration-credentials` (§2.2) — deploy any time after step 1; requires `INTEGRATION_CREDENTIAL_ENCRYPTION_KEY` provisioned before this code path is exercised. **Done — committed to `main`, migrated, and deployed.**
3. **Relativity** `feat/slack-oauth` + `feat/slack-portal-ui` (§2.3) — deploy after step 2; requires the Slack app to exist in api.slack.com with `SLACK_CLIENT_ID`/`SLACK_CLIENT_SECRET`/`SLACK_REDIRECT_URI` matching. **Done — committed to `main`, migrated, and deployed.**
4. **AIKB** Milestone 4's AIKB branch (§2.4 §4.9–§4.10: `runKnowledgeQuery` refactor + `/ask` route + `requireServiceRequest`) — deploy after step 1; the new route is inert until something calls it, so this is safe to ship ahead of step 5. Requires `SERVICE_REQUEST_SIGNING_SECRET` provisioned (manual step, §10). **Done — committed to `main` (`feat: add asynchronous Slack ask pipeline`) and deployed to Railway.**
5. **Relativity** Milestone 4's Relativity branch (§2.4 §4.4–§4.13: events endpoint, dedup, tenant mapping, delivery) — deploy **last**, after step 4 is live in the target environment, and only after the Slack app's Event Subscriptions URL is manually pointed at the new `POST /api/integrations/slack/events` endpoint (manual step, §10) — this is the moment real Slack traffic starts flowing anywhere, and it must land after AIKB's old route (step 1) is already returning `410`, so there is never a window where two live handlers could both answer the same event. **Done — committed to `main` (`feat: add Slack app mention processing`) and deployed to Vercel; Event Subscriptions manually enabled and verified after a follow-up fix (`fix: remove unsupported Vercel cron schedule`, see the Production deployment addendum in the Milestone 4 Implementation Note).**
6. **AIKB** Milestone 5's `feat/slack-knowledge-scope` (§2.5, optional) — deploy any time after step 4; independent of step 5, must be confirmed live before step 5 sees real traffic only if the "General"-only restriction is judged necessary before launch (§4.9).
7. **AIKB** Milestone 6's gap/conversation-metadata polish (§2.6) — can deploy any time after step 4; does not gate step 5, but should land before Phase 4 is considered "done" for that scope (§12).
8. (Housekeeping, after step 5 is verified in staging) delete the now-fully-inert legacy route file/config from AIKB — small follow-up, not a blocking milestone.

## 7. Branch and PR plan

| Branch | Repo | Depends on | Migration | Tests | Deploys | Rollback |
|---|---|---|---|---|---|---|
| `chore/disable-legacy-slack` | AIKB | none | none | Old route returns 410 for `event_callback`; still answers `url_verification` | 1st, standalone | `git revert`. **Done.** |
| `feat/encrypted-integration-credentials` | Relativity | none | `oauth_connections`, `oauth_credentials`, delete plaintext Slack `oauth_tokens` rows | See Milestone 2 Implementation Note | After step 1 above | Revert PR. **Done.** |
| `feat/slack-oauth` + `feat/slack-portal-ui` | Relativity | `feat/encrypted-integration-credentials` | `oauth_states` | See Milestone 3 Implementation Note | After credentials branch + manual Slack app setup | Revert PR. **Done.** |
| `feat/slack-ask-endpoint` (Milestone 4) — **actual branch name: `feat/slack-ask-pipeline`, see below** | AIKB | `chore/disable-legacy-slack` | `005_*.sql` (§5) | Full list in §4.18 | After step 1. **Done — committed to `main`, deployed.** | Revert PR; new route is additive, `/query` untouched |
| `feat/slack-events` (Milestone 4) | Relativity | `feat/slack-oauth`, AIKB `feat/slack-ask-endpoint` deployed | `slack_event_log` (§5), `vercel.json` cron entry — **cron entry was added then removed in production; see Production deployment addendum below** | Full list in §4.18 | Last, after manual Slack Event Subscriptions URL cutover. **Done — committed to `main`, deployed.** | Revert PR + point Slack app's Event Subscriptions URL back to nothing (safe: dead URL) — no data loss, `slack_event_log` rows persist |
| `feat/slack-knowledge-scope` (Milestone 5) | AIKB | `feat/slack-ask-endpoint` | `006_*.sql` (§5) | Filter enforced inside retrieval SQL, unclassified/NULL documents excluded, portal `/query` unaffected | After `feat/slack-ask-endpoint`, before or after `feat/slack-events` per §4.9's confirmation | Revert PR; column default `NULL` means no behavior change until documents are opted in |
| `feat/slack-gap-metadata` (Milestone 6) | AIKB | `feat/slack-ask-endpoint` | (schema already landed in that branch) | Gap dedup constraint test, origin/metadata fields populated correctly | Any time after `feat/slack-ask-endpoint` | Revert PR |

## 8. Test plan

Milestone 4's authoritative, detailed test requirements are specified in **§4.18** (not duplicated here, to avoid drift between two lists). The bullets below are retained as the Milestone 2/3 planning record — their actual final test names/counts are documented in each milestone's Implementation Note below, not here.

**Relativity unit tests (`node --test`, matching the existing convention in `Relativity/test/*.test.js`) — Milestones 2–3, see Implementation Notes for actuals:**
- Token encryption/decryption round-trip; random IV differs across calls for identical plaintext; corrupted ciphertext/auth-tag throws, never returns garbage plaintext; missing/malformed encryption key fails startup, not silently.
- OAuth state: created with correct expiry, rejected once expired, rejected on reuse (single-use), rejected if `client_id`/`member_id` don't match the authenticated caller at callback time.
- Slack authorization URL construction: correct `client_id`, `redirect_uri`, `scope` (`app_mentions:read,chat:write`).
- Cross-organization connection isolation; disconnect isolation; token non-disclosure in any response or log line.

**AIKB unit/integration tests — Milestone 1, see Implementation history above:**
- Old Slack route returns 410 for `event_callback`, still answers `url_verification`.

**End-to-end staging QA for Milestone 4 (manual, against a real Slack workspace and staging deployments of both repos):**
- Connect Slack from the portal; confirm a row appears in `oauth_connections`/`oauth_credentials` and the stored token is not human-readable in the DB (ciphertext only).
- Mention `@RelativityBot` in a channel; receive a threaded answer within a reasonable time.
- Verify citations in the reply reference a real document belonging to that organization only.
- Re-deliver the same Slack event (Slack's own retry, or a manual replay of the same `event_id`) and confirm no duplicate answer is posted.
- Ask a question with no relevant content; confirm exactly one AIKB knowledge gap is created, tagged `origin=slack`.
- Force AIKB to be briefly unreachable during a mention; confirm the sweep (§4.8) retries and eventually delivers (or fails gracefully) rather than silently dropping the question.
- From a second Relativity organization's Slack workspace, confirm its bot cannot answer using org A's documents (cross-tenant isolation, end to end).
- Disconnect Slack from the portal; confirm the bot stops responding to mentions in that workspace afterward.

## 9. Rollback plan

Every branch in §7 has its own scoped rollback (git revert of an additive migration + code change). At the milestone level:
- **Milestone 1** is unconditionally safe to leave in place indefinitely — it only removes unsafe behavior, nothing depends on reverting it.
- **Milestones 2–3** are inert until Milestone 4 sends real traffic through them — reverting either before Milestone 4 ships has zero user-facing impact.
- **Milestone 4** is the first milestone with a live-traffic blast radius. Its rollback is: (a) point the Slack app's Event Subscriptions URL at nothing (Slack tolerates a dead URL — deliveries fail and are eventually given up on, no crash on Relativity's side), (b) revert the `feat/slack-events` deploy. `slack_event_log` rows already written are retained, not lost, so no in-flight question data disappears — they simply stop being processed until the endpoint is restored.
- **Milestone 5** rollback (revert the `collection` filter, if it was ever enabled) would widen retrieval back to "all documents" for Slack — which is Milestone 4's own default scope anyway (§4.9), so this rollback carries no additional risk beyond what Milestone 4 already ships with.

## 10. Manual Slack/Vercel/Railway/Supabase setup steps

*(Originally listed for completeness and future execution — none of these had been performed at the time this section was written. All steps below were subsequently carried out as part of Milestone 4's production rollout on 2026-07-16, with one deviation from the plan: the `vercel.json` cron entry described in the Vercel bullet below was deployed, rejected by the Vercel Hobby plan, and removed rather than upgraded to. See the Milestone 4 Implementation Note's "Production deployment (2026-07-16)" addendum for the full account.)*

- **Slack app (api.slack.com/apps):** app exists, OAuth redirect URL and bot scopes (`app_mentions:read`, `chat:write`) already configured and exercised successfully per Milestone 3's Current Verified State. For Milestone 4: leave Event Subscriptions **unset** until Milestone 4 is verified in staging, then set the Request URL to `https://relativitysystems.ai/api/integrations/slack/events` and subscribe to the `app_mention` bot event only (§4.16).
- **Vercel (Relativity):** `SLACK_CLIENT_ID`/`SLACK_CLIENT_SECRET`/`SLACK_SIGNING_SECRET`/`SLACK_REDIRECT_URI`/`INTEGRATION_CREDENTIAL_ENCRYPTION_KEY` provisioning is a Milestone 2/3 prerequisite already documented in those Implementation Notes. For Milestone 4: additionally provision `SERVICE_REQUEST_SIGNING_SECRET` (shared with AIKB) and add a `crons` entry to `vercel.json` for the delivery sweep (§4.8). **Update (2026-07-16): the `crons` entry was added and deployed, but the target Vercel project is on the Hobby plan, which rejects any cron schedule finer than once daily — the `*/5 * * * *` entry caused the deployment to fail outright, not just run less often than intended. The entry was removed (commit `fix: remove unsupported Vercel cron schedule`) to restore deployability. The delivery sweep is consequently not currently scheduled at all; see the Production deployment addendum in the Milestone 4 Implementation Note.**
- **Railway (AIKB):** provision `SERVICE_REQUEST_SIGNING_SECRET` (same value as Relativity's). The legacy `SLACK_BOT_TOKEN`/`SLACK_SIGNING_SECRET` env vars on AIKB remain unused since Milestone 1 and can be removed once the legacy route file itself is deleted (§6 step 8).
- **Supabase (Global project):** run the Relativity migrations from §5 in order, including the new Milestone 4 `slack_event_log` migration.
- **Supabase (AIKB project):** run the AIKB migrations from §5 in order, including the new Milestone 4 `005_*.sql`.
- **Secret generation:** `INTEGRATION_CREDENTIAL_ENCRYPTION_KEY` and `SERVICE_REQUEST_SIGNING_SECRET` should each be generated via `crypto.randomBytes(32).toString('hex')` (or equivalent) and stored only in the two platforms' environment variable managers — never committed, never logged.

## 11. Risks and mitigations

| Risk | Mitigation |
|---|---|
| A real Slack app is currently pointed at AIKB's legacy URL and Milestone 1's 410 response goes unnoticed | Milestone 1's structured warning log on every hit gives an operator a concrete signal to investigate and reconfigure that Slack app's Event Subscriptions URL |
| Vercel serverless function timeout during the synchronous enqueue call to AIKB (§4.8) | The call only needs to enqueue, not answer — kept deliberately fast; if it still exceeds a safe budget, the `slack_event_log` row is already persisted before the call, so the sweep recovers it regardless |
| Two orgs racing to connect the same Slack workspace simultaneously | DB-level partial unique index (§2.2, already applied) is the actual backstop, not just an app-level check — a race loses at the constraint, not silently at the app layer |
| Milestone 4 ships with full-organization retrieval scope for Slack, not a "General"-only subset | Called out explicitly in §4.9 and §2.5 as a decision needing confirmation before Milestone 4 is exposed to real traffic, plus a path (Milestone 5) to narrow it if required |
| AIKB currently has no automated test runner beyond what Milestone 1 added | Milestone 1's branch already added a minimal `node:test`-based suite, so Milestone 4's AIKB branch has a place to add tests from the start |
| Delivery sweep (Vercel Cron) is a new operational surface with its own failure mode (cron itself not firing) | Sweep interval kept short (a few minutes) and `slack_event_log` rows are inspectable directly in Supabase for manual recovery if the cron mechanism itself needs debugging — no data is lost even if the sweep temporarily doesn't run |
| The Milestone 4 signed envelope (§4.10) has no built-in rotation story | Adding a `schemaVersion`-equivalent field is deferred until rotation is actually needed — tracked as a known gap, not solved in this milestone |

## 12. Definition of Done

Phase 4's Slack MVP is complete when:
- AIKB's legacy `routes/slack.js` returns 410 for every event and contains no cross-tenant-hash or global-bot-token logic (Milestone 1). **Done.**
- Slack bot tokens are stored only as AES-256-GCM ciphertext with per-row random IVs and auth tags, in `oauth_credentials`, never in plaintext anywhere (Milestone 2). **Done.**
- An organization admin can connect and disconnect exactly one Slack workspace via the portal, with state, org-binding, and admin-only enforcement all covered by passing tests (Milestone 3). **Done.**
- Milestone 4's full definition of done (§4.20) is met: mentioning `@RelativityBot` reliably produces one grounded, cited threaded reply per distinct question via AIKB's shared query pipeline, never sends the bot token to AIKB, and never leaks one organization's documents to another's Slack workspace. **Done — verified end-to-end against a real Slack workspace in production on 2026-07-16 (§4.20 items 1–4, 7–14 confirmed live).** **Partial exception, not a design failure:** the "survives an AIKB outage without silently dropping the question" guarantee is currently weaker than specified — it depended on the Vercel Cron sweep (§4.8) as the backstop for a stuck `received`/`enqueued` row, and that sweep is not currently scheduled (see the Production deployment addendum below). A transient AIKB outage today is still caught by AIKB's own Inngest `onFailure` callback (§4.8's second layer) proactively notifying Relativity, but the sweep's independent recovery path for a failed *initial* `/ask` call is inactive until a scheduler is restored. This is documented as an explicit operational limitation, not a reopening of Milestone 4.
- Only documents explicitly opted into "General" are ever retrievable or citable via Slack, enforced inside the retrieval SQL itself, with portal retrieval behavior unchanged — **only if Milestone 5 is judged necessary before launch** (§4.9, §2.5); otherwise Milestone 4 ships with the same retrieval scope the portal already has, by deliberate, confirmed decision.
- Slack-originated knowledge gaps are idempotent and correctly tagged with origin/thread metadata from Milestone 4's minimal schema slice (§4.19); the fuller Milestone 6 polish (system-vs-user `reportedBy` distinction, richer metadata) lands before Phase 4 as a whole is considered fully "done," without blocking Milestone 4's own launch.
- The full E2E staging checklist (§8) passes against a real Slack workspace.
- Every "needs confirmation" callout in §4.9 and §11 has been explicitly approved or revised before the corresponding code ships.

Direct messages and fine-grained employee authorization (Milestone 7) remain explicitly out of scope for "done."

---

## Milestone 2 Implementation Note (Relativity — `feat/encrypted-integration-credentials`)

**Status:** Implemented on branch `feat/encrypted-integration-credentials` in the Relativity repository. Not committed, not pushed, migration not executed against Supabase, no packages installed, no production environment changes.

**Final schema selected:** the scoped generic model from §2.2 above, exactly as planned — `oauth_connections` (metadata: `client_id`, `provider` restricted by CHECK to `slack|microsoft|gmail|google_drive|dropbox`, `status`, `external_account_id`/`external_account_name`, `scopes_granted`, `provider_metadata jsonb`, `connected_by_member_id`) plus `oauth_credentials` (`connection_id`, `access_token_encrypted jsonb`, `refresh_token_encrypted jsonb` nullable, `expires_at` nullable, `encryption_key_version`). Two partial unique indexes enforce one active connection per `(client_id, provider)` and one active organization per `(provider, external_account_id)`. No `oauth_providers` registry table was added — deliberately smaller than the long-term model, per the original plan. Migration file: `supabase/migrations/20260714_oauth_connections.sql`.

**Encryption envelope format:** implemented exactly as specified — `{ version: 1, algorithm: "aes-256-gcm", iv: "<base64>", authTag: "<base64>", ciphertext: "<base64>" }`, AES-256-GCM, 12-byte random IV per call, 16-byte auth tag, no `keyVersion` field inside the envelope itself (that lives as the separate `encryption_key_version` column on `oauth_credentials`, currently always `1`) — keeps the encryption module fully decoupled from the DB schema. `services/integrationCredentialEncryption.js` reads `SLACK_TOKEN_ENCRYPTION_KEY` directly from `process.env` at call time (not the cached `config/index.js` snapshot) so validation is genuinely lazy — caught at the moment of use, not only at server start.

**Consistency/transaction approach:** Option A from §2.6/Milestone 6 planning (RPC) — a new PL/pgSQL function `replace_active_oauth_connection(...)` in the migration performs the revoke-old → insert-connection → insert-credential sequence inside one function call, which Postgres executes as a single transaction. Plaintext never crosses the RPC boundary — `services/oauthConnectionsService.js#createOrReplaceConnection` calls `encryptCredential()` before constructing the RPC arguments. A failed credential insert raises inside the function body and rolls back the connection insert with it, so an "active" connection row can never exist without a matching credential row. Verified by a unit test asserting that a simulated RPC failure results in exactly one write call total (the RPC itself) and zero follow-up table writes from the Node service — there is no fragile multi-step client-side sequence that could itself partially fail.

**Legacy compatibility strategy:** `supabaseService.js`'s `upsertToken`/`getToken`/`getClientConnectionStatus` are unchanged in behavior — only deprecation comments were added, explicitly steering new Slack code away from them and documenting that `getClientConnectionStatus`'s Slack field must switch to `oauthConnectionsService#getSafeConnectionStatus` in Milestone 3 (since Slack rows in `oauth_tokens` no longer get new writes after this migration). Google Drive and Dropbox are completely untouched — same table, same functions, same behavior.

**Plaintext Slack-row deletion:** the migration's final statement (`DELETE FROM oauth_tokens WHERE provider = 'slack'`) is destructive and irreversible — it removes the existing plaintext Slack `access_token`/`refresh_token` rows captured by the old OAuth flow. No token value is read, selected, or logged by the statement or by any code in this milestone. This executes the already-approved Phase 1 §11 Decision 4 (discard and rebuild the Slack OAuth flow). Any workspace connected under the old flow must reconnect once Milestone 3 ships — there is no migration path for the old plaintext token into the new encrypted schema, by design.

**Key-rotation limitation:** not implemented. `encryption_key_version` (DB column) and `version` (envelope field) exist specifically as the seam for a future rotation; today only version 1 exists and rotation would require new code (a versioned key lookup + a re-encryption sweep), tracked as a Should-Have, not a Milestone 2 deliverable.

**Tests added:** 25 new tests across two files, all passing, run via `node --test` (no new test framework) — full suite: 51/51 passing (26 new + 25 pre-existing). `test/integrationCredentialEncryption.test.js` covers the full encryption contract (round-trip, random IV, missing/malformed/wrong-length key, malformed envelope, unsupported version/algorithm, corrupted ciphertext/auth tag, wrong key, no plaintext leakage in the envelope or in error messages). `test/oauthConnectionsService.test.js` covers safe-status-shape mapping (a deliberately poisoned row with a fake `access_token_encrypted` field proves the mapping allowlists rather than spreads) and query-scoping via a dependency-injected fake Supabase client (cross-client isolation, provider-scoped external-account lookup, client-scoped revoke/delete, no default-client fallback, and the single-atomic-write-call proof above) — this repo has no real test-database pattern, so DI against a fake client was used instead of live Supabase calls, consistent with the instruction not to add a new testing framework or make real network calls.

**Files changed:**
- New: `supabase/migrations/20260714_oauth_connections.sql`
- New: `services/integrationCredentialEncryption.js`
- New: `services/oauthConnectionsService.js`
- New: `test/integrationCredentialEncryption.test.js`
- New: `test/oauthConnectionsService.test.js`
- Modified: `config/index.js` (added `slack.tokenEncryptionKey` passthrough for discoverability only — not read by the encryption service)
- Modified: `services/supabaseService.js` (deprecation comments only — no behavior change)
- Modified: `.env.example` (added `SLACK_TOKEN_ENCRYPTION_KEY` documentation comment; the surrounding restructuring in this file predates this milestone — pre-existing uncommitted work on the branch, not introduced here)

### Milestone 2 Revision (pre-merge targeted fixes)

**Status:** Applied on the same branch, `feat/encrypted-integration-credentials`. Not committed, not pushed, migration still not executed, no packages installed, no production environment changes. Scope stayed within Milestone 2 — no Slack OAuth routes, no Google Drive/Dropbox migration, no Milestone 3 work.

**Environment variable renamed:** the encryption key variable is now `INTEGRATION_CREDENTIAL_ENCRYPTION_KEY` (preferred, primary) everywhere — `services/integrationCredentialEncryption.js`, `config/index.js` (moved out of the `slack` block into a new provider-neutral `integrationCredentials.encryptionKey`), `.env.example`, both test files, and this report. `SLACK_TOKEN_ENCRYPTION_KEY` is retained only as a **deprecated, temporary fallback**: `integrationCredentialEncryption.js` reads the generic variable first and falls back to the legacy name only if the generic one is unset, logging a one-time deprecation warning (never the key value) on first fallback use. Validation (hex format, exact 64-char/32-byte length) is byte-for-byte identical regardless of which variable supplied the string — the fallback does not weaken validation. `.env.example` now shows `INTEGRATION_CREDENTIAL_ENCRYPTION_KEY=` as the live placeholder and `SLACK_TOKEN_ENCRYPTION_KEY=` commented out, documented as deprecated. Both variables are never intended to be required permanently — the fallback is explicitly temporary and will be removed in a future migration once every environment is updated.

**Envelope version vs. key version:** confirmed and reinforced as two deliberately separate fields, never merged. The envelope's `version` (from `ENVELOPE_VERSION`, currently `1`) describes ciphertext *serialization/algorithm format*; `oauth_credentials.encryption_key_version` (from the service's `CURRENT_ENCRYPTION_KEY_VERSION`, currently also `1`, coincidentally) describes *which configured key* encrypted the row. New tests assert the envelope object never contains a `keyVersion`/`encryption_key_version`/`key_version` field, and that the RPC call sends both as distinct arguments (`p_access_token_encrypted.version` vs. `p_encryption_key_version`) sourced from two different constants in two different files.

**Exact persistence-format round-trip test added:** `test/integrationCredentialEncryption.test.js` now includes a test that runs the literal production path — `encryptCredential()` → `JSON.stringify()` (what Supabase does writing to the `oauth_credentials.access_token_encrypted` JSONB column) → `JSON.parse()` (what Supabase returns on read) → `decryptCredential()` — and asserts: the serialized form never contains the plaintext, the parsed-back object has exactly the five documented fields, `version` stays a JS `number` (not a string) through the round trip, the other four fields stay strings, and decryption of the parsed-back object still succeeds and returns the original plaintext.

**Future legacy-token migration note (not implemented now):** once Slack is stable in production, tracked future work is: migrate Google Drive credentials off `oauth_tokens` onto `oauth_connections`/`oauth_credentials`; migrate Dropbox credentials the same way; onboard Microsoft/Gmail directly onto `oauth_connections`/`oauth_credentials` (never onto the legacy plaintext table); and only remove `oauth_tokens` entirely after every active provider has migrated off it. None of this was implemented in Milestone 2 — Google Drive and Dropbox continue to use `oauth_tokens` unchanged.

**Migration/service consistency review:** cross-checked the SQL migration against `oauthConnectionsService.js` directly, item by item — RPC parameter names match exactly (verified by a test that parses the actual `CREATE FUNCTION` signature out of the migration file and compares it to the exact argument keys the service sends); `encryption_key_version` is an `integer` column matching the integer the service sends; `provider` values the service will accept (`SUPPORTED_PROVIDERS`, a new exported constant) are checked by a test against the migration's `CHECK (provider IN (...))` constraint, parsed from the file itself rather than a second hand-copied list; `status` values (`STATUS`, also newly exported) are checked the same way against `CHECK (status IN (...))`; both partial unique indexes were confirmed (by a test asserting on the migration text) to be scoped `WHERE status = 'active'`, meaning a revoked row never blocks reconnecting the same client/provider or the same external account; the external-account uniqueness rule was confirmed to include `provider` in the index (provider-scoped, not global) and to explicitly guard `external_account_id IS NOT NULL`; and `oauth_credentials.connection_id ... ON DELETE CASCADE` was confirmed present, so `deleteConnection` cannot orphan a credential row. These are now enforced as tests that read the actual migration file, not just prose claims, so a future edit to either side that silently drifts from the other fails a test instead of only surfacing as an opaque Postgres error in production. One real gap was found and fixed during this review: the service previously accepted any string as `provider`/relied entirely on the database CHECK constraint to reject bad values — it now validates against `SUPPORTED_PROVIDERS` itself first, failing with a clear application error before ever calling the database.

**Destructive Slack-row cleanup — reviewed and strengthened:** the migration still ends with `DELETE FROM oauth_tokens WHERE provider = 'slack'`, preceded now by a `DO $$ ... RAISE NOTICE ... $$` block that counts and reports how many rows are about to be deleted (via `count(*)`, never selecting a token column) as a migration-time audit trail, plus a documented manual `SELECT count(*) FROM oauth_tokens WHERE provider = 'slack';` an operator can run independently beforehand. No plaintext backup logic was added, per instruction. Stated explicitly, as required: **affected Slack workspaces must reconnect** through the new OAuth flow (Milestone 3) after this runs; **Google Drive and Dropbox rows are completely untouched**; **this migration is intentionally destructive** for legacy plaintext Slack rows, executing the already-approved Phase 1 §11 Decision 4; and **rollback cannot recover the deleted plaintext tokens** — there is no application-level plaintext backup by design, so recovery is only possible from a database backup or point-in-time-recovery snapshot taken before the migration ran, if one exists.

**Active-connection replacement semantics — verified:** confirmed via the migration's transaction boundary (single PL/pgSQL function body — see the strengthened comment on `replace_active_oauth_connection`) that `createOrReplaceConnection` cannot leave two active connections for the same client/provider, an active connection without credentials, or an active connection pointing at failed credential storage — any failure anywhere in the function rolls back everything the function did, including the prior connection's revoke. A new test confirms the Node-service-observable half of this: when the RPC call fails, `createOrReplaceConnection` throws, returns no fabricated safe-status object, and makes no follow-up query of any kind — the "was the old connection actually replaced" question is answered entirely by the RPC's atomicity, never approximated by client-side compensating logic.

**Safe read boundaries — verified:** new tests confirm no metadata/status/revoke/delete operation ever queries `oauth_credentials` — every one of `getActiveConnectionForClient`, `getActiveConnectionByExternalAccount`, `getSafeConnectionStatus`, `markConnectionRevoked`, and `deleteConnection` touches only `oauth_connections` (with `markConnectionRevoked`'s one exception being a `.delete()` on `oauth_credentials`, never a read). Only `getDecryptedCredentialForConnection` reads `oauth_credentials`, and it explicitly allowlists columns (`access_token_encrypted, refresh_token_encrypted, expires_at`) rather than `select('*')`. An exhaustive test confirms `toSafeConnectionStatus`'s output never contains any of: `access_token`, `refresh_token`, `access_token_encrypted`, `refresh_token_encrypted`, `accessToken`, `refreshToken`, `ciphertext`, `iv`, `authTag`, `encryption_key_version`, or `encryptionKeyVersion`.

**Test totals after this revision:** full suite 69/69 passing (up from 51/51). 20 new tests added in this revision (across the two Milestone 2 test files, which now total 46 tests combined, up from 26); 23 pre-existing tests elsewhere in the suite are unaffected.

**Files changed in this revision:**
- Modified: `services/integrationCredentialEncryption.js` (env var rename with deprecated fallback, envelope/key-version doc reinforcement, exported `KEY_ENV_VAR`/`LEGACY_KEY_ENV_VAR`)
- Modified: `services/oauthConnectionsService.js` (`SUPPORTED_PROVIDERS`/`STATUS` constants and validation, exported for consistency testing)
- Modified: `supabase/migrations/20260714_oauth_connections.sql` (preflight row-count `NOTICE` block before the destructive delete, strengthened atomicity/rollback commentary on the RPC)
- Modified: `config/index.js` (renamed/relocated the encryption-key passthrough field)
- Modified: `.env.example` (renamed variable, deprecated legacy variable documented and commented out)
- Modified: `test/integrationCredentialEncryption.test.js` (rename, fallback tests, envelope/key-version distinction test, exact persistence round-trip test)
- Modified: `test/oauthConnectionsService.test.js` (SQL/service consistency tests, safe-read-boundary tests, strengthened replacement-semantics test, provider-validation test)

## Milestone 3 Implementation Note (Relativity — `feat/slack-oauth`)

**Status:** Implemented on branch `feat/slack-oauth` in the Relativity repository. Not committed, not pushed, migration not executed against Supabase, no packages installed, no production environment changes. Scope stayed within Milestone 3 — no Slack Events endpoint, no `app_mention` handling, no calls to AIKB, no message posting, no changes to AIKB (untouched).

**Final route structure:** a new, focused route module, `routes/integrations/slack.js`, mounted at `/api/integrations/slack` in `app.js`, exposing exactly:
- `GET /api/integrations/slack/start` — `clientAuth` + owner/admin only. Returns `{ url }`.
- `GET /api/integrations/slack/callback` — public (Slack redirects the browser here with no bearer token); always resolves to a redirect, never JSON.
- `GET /api/integrations/slack/status` — `clientAuth` only (any active member), matching the existing `GET /auth/me` convention of exposing connection booleans to any active member, not just owner/admin.
- `POST /api/integrations/slack/disconnect` — `clientAuth` + owner/admin only.

All four routes are thin Express adapters; the actual orchestration lives in a new `services/slackIntegrationService.js`, which composes three other focused services (`oauthStateService`, `slackService`, the existing `oauthConnectionsService`) and is fully unit-testable via dependency injection with zero Express/HTTP dependency — consistent with this repo's existing pattern (`services/oauthConnectionsService.js`) of keeping heavy logic in DI'd services and routes thin.

**OAuth state design:** a new table, `oauth_states` (migration below), and a new `services/oauthStateService.js`. `generateAndStoreState` produces 32 random bytes (`crypto.randomBytes(32)`, 64 hex characters) as the raw state — sent to Slack as-is, in the `state` query parameter, and **never written to the database**; only its SHA-256 hash (`state_hash`) is persisted, bound to `client_id` and `member_id` resolved server-side from the authenticated session (never from the browser), `provider = 'slack'`, and a 10-minute `expires_at`. Single-use consumption is one atomic conditional `UPDATE ... WHERE state_hash = $1 AND provider = $2 AND consumed_at IS NULL AND expires_at > now() RETURNING *` (no RPC needed — a single UPDATE statement already executes as one Postgres command, and Postgres re-evaluates the WHERE clause under the row lock it takes, so two concurrent callback requests for the same state can never both succeed). When the atomic update matches nothing, a strictly read-only follow-up `SELECT` (never mutates, never grants what the update didn't) classifies *why* — `not_found`, `reused`, `expired`, or `provider_mismatch` — so the callback can choose the right safe redirect (`invalid_state` vs. `expired_state`) without ever leaking a raw Slack error, the state value, or a workspace ID into the URL.

**Callback verification sequence, exactly as required:** validate Slack config → hash the incoming state → atomically consume it → resolve `client_id`/`member_id` from the consumed row only → re-verify (fresh reads, not cached from `/start` time) that the organization (`clients.is_active`) and the member (`client_members.status === 'active'` and `role` still `owner`/`admin`, via a new `supabaseService.getClientMemberById(memberId, clientId)`) are still valid *at callback time* → exchange the code via `slackService.exchangeCodeForToken` → validate the Slack response shape (`ok === true`, `access_token`, `team.id`, `bot_user_id` all present; scopes/enterprise/app_id/token_type captured when present) → persist via `oauthConnectionsService.createOrReplaceConnection` → redirect to a safe success path. Every rejection branch (denial, missing code/state, unknown/reused/expired/provider-mismatched state, deactivated or demoted member, inactive org, Slack `ok:false`, missing token/team/bot fields, a connection-persist failure) resolves to one of five safe redirects (`?integration=slack&status=connected` or `&error=access_denied|invalid_state|expired_state|connection_failed`) — `handleCallback` is written to never throw; there is no code path that can leak a raw Slack error string, the state value, or the access token into a redirect URL, a JSON body, or a log line (only safe internal codes like `SLACK_OAUTH_FAILED` are logged, per `services/slackService.js`'s `ERROR_CODES`).

**Workspace metadata stored** (via `oauthConnectionsService.createOrReplaceConnection`, exactly as specified): `provider: 'slack'`, `externalAccountId: team.id`, `externalAccountName: team.name`, `scopesGranted` (parsed from Slack's `scope` field), `connectedByMemberId`, and `providerMetadata: { team_id, team_name, enterprise_id, enterprise_name, bot_user_id, app_id, token_type }`. The bot token is passed as `accessToken` and is encrypted by `createOrReplaceConnection` before it ever reaches a database call — this milestone never touches `services/integrationCredentialEncryption.js` or the encryption envelope format directly, only calls the Milestone 2 service exactly as designed.

**Encrypted credential path:** unchanged from Milestone 2 — `oauth_connections`/`oauth_credentials`, AES-256-GCM, `replace_active_oauth_connection` RPC as the atomicity boundary. This milestone is purely a new caller of that existing, already-tested path; no changes were made to `services/oauthConnectionsService.js`'s logic (only two call sites elsewhere were updated — see below).

**Old Slack-flow retirement — chosen strategy: 410 Gone, not deletion, not a feature flag.** `routes/auth.js`'s `GET /slack/start` and `GET /slack/callback` now return `410 Gone` with a safe JSON body pointing at the new routes; every line of the old unsafe logic (unsigned `base64(JSON)` state trusting a client-supplied `clientId`, the `incoming-webhook` scope, `upsertToken`/`updateClientSlackChannel` calls) is deleted, not flagged off — there is no config toggle that could silently re-arm it. This mirrors the same reasoning the architecture plan used for Milestone 1's AIKB legacy route: a plain 404 looks identical to "route never existed," while a deliberate 410 gives an operator (or a stale bookmark / cached `portal.js`) a clear signal if the old URL is ever hit again. Unlike the AIKB Events route Milestone 1 addressed, nothing external polls `/auth/slack/{start,callback}` on its own — they're only reachable via a user click through the portal, and the portal now calls the new routes exclusively — so full deletion was also a defensible option; 410 was chosen for the audit-trail benefit at negligible cost. `SLACK_APP_ID`/`SLACK_APP_SECRET` (the old flow's only config dependency) are removed from `config/index.js` and `.env.example` along with it. This executes the retirement Milestone 2's migration comment already promised: "any Slack workspace connected under the old flow must reconnect through the new OAuth flow (Milestone 3) after this runs."

**Two small necessary consistency fixes, not refactors — both directly caused by Milestone 2 retiring `oauth_tokens` for Slack:**
- `supabaseService.js#getClientConnectionStatus` (used by `GET /auth/me` and the API-key-gated `GET /auth/status/:clientId`) now sources Slack's `connected` boolean from `oauthConnectionsService.getSafeConnectionStatus(clientId, 'slack')` instead of `oauth_tokens` — exactly the switch Milestone 2's own code comment flagged as required once Milestone 3 landed. Dropbox and Google Drive are untouched, still reading `oauth_tokens` via `getToken` exactly as before.
- `supabaseService.js#getAllClientsWithStatus` (the internal admin dashboard's client list) had the same latent problem — its Slack column would have silently shown every workspace as disconnected forever after Milestone 2's `DELETE FROM oauth_tokens WHERE provider = 'slack'`. Fixed the same way: a metadata-only query (`client_id` from `oauth_connections` where `provider = 'slack' AND status = 'active'`, never touching `oauth_credentials`) replaces the `oauth_tokens` lookup for the `slack` column only; `dropbox`/`google_drive` columns are unchanged.

**Portal UI added:** the smallest UI consistent with §11's exclusions — one row in a new "Integrations" panel on the Overview tab (`public/portal/portal.html`), styled with the portal's existing `panel`/`badge` classes (no new visual system introduced). Shows "Slack" + a Connected/Not connected badge + workspace name when connected. "Connect Slack"/"Disconnect" buttons are hidden client-side for non-owner/admin members (`isOwnerAdmin`, already computed in `portal.js` from `GET /auth/me`) — server-side authorization in the route middleware is what actually enforces this, the UI hiding is only a convenience. Connect calls `GET .../start` and redirects the browser to the returned `url`; Disconnect confirms, then calls `POST .../disconnect` and refreshes status; status loads from `GET .../status` on page load. The existing generic `?connected=`/`?error=` redirect handling in `portal.js` was extended (not replaced) with a Slack-specific branch that recognizes `?integration=slack&status=connected` and the four `?integration=slack&error=<code>` values, mapping each to a distinct, safe user-facing banner message — no raw Slack error, state, or workspace ID ever reaches this code, only the already-sanitized redirect query params `routes/integrations/slack.js` produces. No channel selection, permissions UI, user mapping, analytics, DM settings, knowledge-scope controls, or bot-testing UI was added, per the explicit exclusion list.

**Tests:** 79 new tests across four new files, plus the existing suite unaffected — full suite **144/144 passing** (up from 69/69), run via `node --test`, no new test framework, no real Slack or Supabase network call anywhere.
- `test/oauthStateService.test.js` (18 tests) — DI'd fake Supabase client (extended with `insert`/`is`/`gt` beyond Milestone 2's fake, since this table's atomic-consume query needs them): raw-state randomness/length, hash-only persistence (raw state never appears in what's written), single-use consumption (second consume of the same state is `reused`), unknown/expired/provider-mismatched states each rejected with a distinct classification, and the exact WHERE-clause shape of the atomic update.
- `test/slackService.test.js` (23 tests) — `buildAuthorizationUrl` (exact `client_id`/`redirect_uri`, exactly `app_mentions:read,chat:write`, explicitly asserts `incoming-webhook`/`im:history`/`channels:history`/`groups:history`/`users:read`/`users:read.email` are absent, never exposes the secret), `validateOAuthResponse` (pure, every required-field-missing case, `ok:false`, enterprise metadata capture), and `exchangeCodeForToken`/`revokeToken` via a DI'd fake `httpClient` (non-2xx rejected, invalid JSON rejected, network error rejected, Slack `ok:false` rejected, client secret never leaked into a thrown message, `revokeToken` never throws and returns `false` on failure).
- `test/slackIntegrationService.test.js` (32 tests) — the full `handleCallback` orchestration via DI'd fakes for all four collaborators: every rejection path (denial, missing code/state, unknown/reused/expired/provider-mismatched state, deactivated member, member who lost owner/admin role, inactive org, Slack failure, missing bot/team fields, connection-persist failure) asserted against the exact expected redirect constant; the success path asserted against the exact `createOrReplaceConnection` argument shape (including `providerMetadata`); an explicit assertion that the legacy `upsertToken`/`updateClientSlackChannel` are never called (the fakes throw if invoked); `mapSlackStatusResponse` allowlisting (a poisoned row with a fake `access_token_encrypted` field proves it maps rather than spreads, mirroring Milestone 2's equivalent test); `disconnect` idempotency, cross-org scoping, best-effort revoke-even-on-Slack-failure, and no-token-in-response.
- `test/slackRoutes.test.js` (8 tests) — a real Express app (`app.js`) booted on an ephemeral port, testing only paths that provably make no network call (missing-auth-header 401s, which `clientAuth` rejects before any Supabase call; the callback's denial/missing-param early returns, resolved before any state lookup): confirms `/api/integrations/slack/{start,status,disconnect}` require authentication, the callback's safe redirects for denial and missing state, and that the old `/auth/slack/{start,callback}` now return `410`.

**Migration:** `supabase/migrations/20260715_oauth_states.sql` — additive only (new `oauth_states` table + three indexes), no destructive statements, does not touch `oauth_connections`, `oauth_credentials`, or `oauth_tokens`. Not executed against Supabase in this milestone, per instruction.

**Environment variables:** `SLACK_CLIENT_ID`, `SLACK_CLIENT_SECRET`, `SLACK_REDIRECT_URI` (read by `services/slackService.js` via `config/index.js`'s `slack` block), `SLACK_SIGNING_SECRET` (plumbed through `config/index.js` for discoverability; not read by anything yet — reserved for the future Slack Events milestone), and the existing `INTEGRATION_CREDENTIAL_ENCRYPTION_KEY` (with its Milestone 2 temporary fallback to `SLACK_TOKEN_ENCRYPTION_KEY`, unchanged). `SLACK_APP_ID`/`SLACK_APP_SECRET` (the retired flow's only config) were removed from `config/index.js` and `.env.example`. `.env.example` was updated with placeholders and accurate comments only — no real values changed, consistent with the existing file already having placeholder `SLACK_CLIENT_ID`/`SLACK_CLIENT_SECRET`/`SLACK_SIGNING_SECRET`/`SLACK_REDIRECT_URI` lines staged from prior in-progress work (this milestone de-duplicated a leftover duplicate `SLACK_REDIRECT_URI=` line and consolidated the comments).

**Known limitations (tracked, not blocking this milestone):**
- No key rotation (unchanged from Milestone 2 — `encryption_key_version` is the seam, not the mechanism).
- No scheduled cleanup job for expired/consumed `oauth_states` rows — the migration documents a safe manual/cron `DELETE` an operator can run; the rows hold no secret material (only a hash) so this is hygiene, not a security gap.
- Google Drive and Dropbox remain on the legacy plaintext `oauth_tokens` path — migrating them is still a separate, not-yet-scheduled Should-Have (Phase 2 §12), unaffected by this milestone.
- No Slack Events endpoint, signature verification, `app_mention` handling, message posting, or AIKB integration — explicitly out of scope for Milestone 3; `SLACK_SIGNING_SECRET` is wired through config now specifically so that future milestone doesn't need a config-layer change.
- `/status` is available to any authenticated active member (not owner/admin-restricted), matching this repo's existing convention for connection-status visibility (`GET /auth/me` already exposes the same booleans); revisit if a stricter policy is ever adopted for other providers.

**Manual steps required before this can be tested against real Slack (none performed by this milestone at the time it was written — updated below, all now complete as of the Milestone 4 production rollout on 2026-07-16):**
1. In the Slack app configuration (api.slack.com/apps), set the OAuth & Permissions Redirect URL to exactly `https://relativitysystems.ai/api/integrations/slack/callback` for production, or the equivalent `APP_BASE_URL`-based URL for local/staging. **Done.**
2. Configure Bot Token Scopes to exactly `app_mentions:read` and `chat:write` — remove `incoming-webhook` and any other legacy scope if the app was previously configured for the old flow. **Done.**
3. Populate `SLACK_CLIENT_ID`, `SLACK_CLIENT_SECRET`, `SLACK_SIGNING_SECRET`, `SLACK_REDIRECT_URI`, and `INTEGRATION_CREDENTIAL_ENCRYPTION_KEY` in the real environment (Vercel project settings for production; local `.env` for development). **Done — confirmed indirectly by the working OAuth connection and Milestone 4's live signature verification, both of which are structurally impossible without these being set correctly in production.**
4. Execute `supabase/migrations/20260715_oauth_states.sql` against the target Supabase project (manually, in order after `20260714_oauth_connections.sql` if that has not already been applied) before exercising `/start`. **Done.**
5. Any workspace previously connected under the old flow must reconnect through `/api/integrations/slack/start` — there is no data migration path from the deleted plaintext `oauth_tokens` rows. **Done — the workspace exercised in Milestone 4's production verification connected through the new flow.**

**Files changed:**
- New: `supabase/migrations/20260715_oauth_states.sql`
- New: `services/oauthStateService.js`
- New: `services/slackIntegrationService.js`
- New: `routes/integrations/slack.js`
- New: `test/oauthStateService.test.js`
- New: `test/slackService.test.js`
- New: `test/slackIntegrationService.test.js`
- New: `test/slackRoutes.test.js`
- Modified: `services/slackService.js` (full rewrite — OAuth v2 `buildAuthorizationUrl`/`exchangeCodeForToken`/`revokeToken`/`validateOAuthResponse`, DI'd `httpClient`, safe error codes; no message-posting logic)
- Modified: `services/supabaseService.js` (`getClientConnectionStatus` and `getAllClientsWithStatus` switched to `oauthConnectionsService` for Slack only, as described above; added `getClientMemberById`)
- Modified: `routes/auth.js` (old `/slack/start`/`/slack/callback` replaced with `410 Gone`; removed now-unused `slackConfig`/`slackService` imports)
- Modified: `app.js` (mounted `routes/integrations/slack.js` at `/api/integrations/slack`)
- Modified: `config/index.js` (`slack` block replaced: `clientId`/`clientSecret`/`signingSecret`/`redirectUri`; `appId`/`appSecret` removed)
- Modified: `.env.example` (Slack section consolidated/de-duplicated, comments updated; no real values touched)
- Modified: `public/portal/portal.html` (new Integrations panel with a single Slack row)
- Modified: `public/portal/portal.css` (small, scoped `.integration-*` rules reusing existing `.panel`/`.badge` classes)
- Modified: `public/portal/portal.js` (Slack status/connect/disconnect wiring; extended OAuth redirect-param handling with Slack-specific safe error messages)

## Milestone 4 Implementation Note (Relativity — `feat/slack-events`; AIKB — `feat/slack-ask-pipeline`)

**Status (updated 2026-07-16): Deployed to production.** Originally implemented on branch `feat/slack-events` in Relativity and `feat/slack-ask-pipeline` in AIKB with nothing committed, pushed, or migrated. Both branches have since been merged to `main` in their respective repositories (Relativity: `feat: add Slack app mention processing`; AIKB: `feat: add asynchronous Slack ask pipeline`), both migrations (`20260716_slack_event_log.sql`, `005_slack_origin_tracking.sql`) have been executed against their target Supabase projects, both services have been deployed (Vercel and Railway respectively), the Slack app's Event Subscriptions have been manually enabled, and the end-to-end flow has been verified against a real Slack workspace in production. One deployment-time issue was found and fixed after the initial deploy — a Vercel Cron entry rejected by the Hobby plan — documented in full in the "Production deployment (2026-07-16)" addendum at the end of this note. No packages were installed beyond what was already planned (Relativity's existing `axios`/`@supabase/supabase-js` and AIKB's global `fetch` cover every outbound call this milestone needed).

### Final architecture

The exact flow specified in §4.3 was implemented as designed, with one deliberate, documented deviation from the pseudocode's literal step order (see "Known limitations" below):

```
Slack --app_mention--> POST /api/integrations/slack/events (Relativity)
  verify signature (raw body, timing-safe, 5-min window)
  url_verification -> respond with challenge, stop
  filter (app_mention only; reject bot/self/edited/deleted/malformed)
  resolve team_id -> oauth_connections (trusted DB lookup)
  extract + normalize question
  insert slack_event_log row (dedupe on (provider, external_event_id))
  if duplicate -> 200 ack, stop
  if empty question -> reply directly (no AIKB call), 200 ack, stop
  else: call POST /api/knowledge/ask (fast accept-and-enqueue) -> 200 ack

POST /api/knowledge/ask (AIKB) -- requireApiKey + requireServiceRequest (HMAC envelope)
  enqueue Inngest event knowledge/slack.question.requested, return {accepted:true} immediately

Inngest: knowledge/slack.question.requested (AIKB, Railway)
  step 1: runKnowledgeQuery({..., origin:'slack'})  <- SAME shared pipeline /query uses
  step 2: POST /api/integrations/slack/deliver (signed envelope, reversed direction)
  onFailure (retries exhausted): POST /deliver with {error:true} so the user isn't left silent

POST /api/integrations/slack/deliver (Relativity) -- requireServiceRequest (AIKB signs, Relativity verifies)
  look up slack_event_log by idempotencyKey; verify clientId matches
  conditional claim: UPDATE ... WHERE status='enqueued' -> 'answered' (only first attempt proceeds)
  decrypt bot token server-side, format answer+citations, chat.postMessage
  mark delivered/failed

GET /api/integrations/slack/sweep (Relativity, Vercel Cron, CRON_SECRET-gated)
  retries slack_event_log rows stuck in received/enqueued past 30s, capped at 3 attempts;
  on exhaustion, best-effort posts the temporary-failure fallback, then marks failed
```

### Branches

- Relativity: `feat/slack-events` (based on `main`, up to date with `origin/main` at start).
- AIKB: `feat/slack-ask-pipeline` (based on `main`, up to date with `origin/main` at start). AIKB required real changes (new route, migration, refactor, Inngest function), so a feature branch was created there too, per the objective's instruction.

### Files created or modified

**Relativity (`feat/slack-events`):**
- New: `supabase/migrations/20260716_slack_event_log.sql`
- New: `services/slackSignatureService.js` — raw-body HMAC signature verification (§4.4-§4.5).
- New: `services/serviceRequestAuth.js` — the additive HMAC service-request envelope (§4.10), byte-for-byte mirrored in AIKB.
- New: `middleware/requireServiceRequest.js` — verifies the envelope on inbound `POST /deliver`.
- New: `services/slackEventLogService.js` — `slack_event_log` CRUD/state transitions (§4.7-§4.8).
- New: `services/slackQuestionService.js` — mention-stripping/question normalization (§4.11).
- New: `services/slackAnswerFormatter.js` — answer/citation formatting, fallback copy (§4.14).
- New: `services/slackDeliveryService.js` — `chat.postMessage` I/O (§4.13), separate from OAuth-only `slackService.js`.
- New: `services/aikbAskClient.js` — signs and sends the fast accept-and-enqueue call to AIKB.
- New: `services/slackEventsService.js` — orchestrates `/events` end-to-end, plus the Cron sweep.
- New: `services/slackDeliverService.js` — orchestrates `/deliver` (claim, decrypt, format, post, mark).
- Modified: `services/oauthConnectionsService.js` — added `getConnectionById` (metadata-only, read-only; needed by `/deliver` and the sweep to re-verify a connection by id).
- Modified: `app.js` — `express.json({ verify })` now captures `req.rawBody`.
- Modified: `config/index.js` — added `slack.questionMaxLength`, `serviceRequest.signingSecret`, `aikb.askTimeoutMs`, `cron.secret`.
- Modified: `.env.example` — documented `SERVICE_REQUEST_SIGNING_SECRET`, `SLACK_QUESTION_MAX_LENGTH`, `AIKB_ASK_TIMEOUT_MS`, `CRON_SECRET`.
- Modified: `vercel.json` — added a `crons` entry for the sweep.
- Modified: `routes/integrations/slack.js` — added `POST /events`, `POST /deliver`, `GET /sweep`.
- New tests: `test/slackSignatureService.test.js`, `test/serviceRequestAuth.test.js`, `test/slackQuestionService.test.js`, `test/slackAnswerFormatter.test.js`, `test/slackDeliveryService.test.js`, `test/aikbAskClient.test.js`, `test/slackEventLogService.test.js`, `test/slackEventsService.test.js`, `test/slackDeliverService.test.js`, `test/slackEventsRoutes.test.js`.

**AIKB (`feat/slack-ask-pipeline`):**
- New: `migrations/005_slack_origin_tracking.sql`
- New: `services/serviceRequestAuth.js` — mirrors Relativity's file exactly (signing-string format must match byte-for-byte).
- New: `middleware/serviceRequest.js` — `requireServiceRequest`, verifies inbound `/ask` calls.
- New: `services/runKnowledgeQuery.js` — the shared pipeline extracted from `/query`'s handler body.
- New: `services/relativityDeliverClient.js` — signs and POSTs the `/deliver` callback.
- Modified: `routes/knowledge.js` — `/query` refactored to call `runKnowledgeQuery` (unchanged request/response shape); added `POST /ask`.
- Modified: `services/supabaseService.js` — `createChatSession` accepts an optional `{ origin, originMetadata, idempotencyKey }`; added `getChatSessionByIdempotencyKey`.
- Modified: `inngest/functions.js` — added `slackQuestionRequested` (`knowledge/slack.question.requested`).
- Modified: `config/index.js` — added `serviceRequest.signingSecret`, `relativity.apiBaseUrl`, `relativity.deliverTimeoutMs`.
- New tests: `test/serviceRequestAuth.test.js`, `test/requireServiceRequest.test.js`, `test/runKnowledgeQuery.test.js`, `test/relativityDeliverClient.test.js`, `test/knowledgeAskRoute.test.js`.

### Route structure

- `POST /api/integrations/slack/events` (Relativity, new) — Slack Events API request URL.
- `POST /api/integrations/slack/deliver` (Relativity, new) — AIKB's callback.
- `GET /api/integrations/slack/sweep` (Relativity, new) — Vercel Cron entry.
- `POST /api/knowledge/ask` (AIKB, new) — the fast accept-and-enqueue leg.
- `POST /api/knowledge/query` (AIKB, unchanged route/shape, refactored internals).
- Milestone 3's `/start`, `/callback`, `/status`, `/disconnect` are unmodified.

### Signature verification and raw-body handling

`app.js`'s single `express.json()` call gained a `verify` callback that stashes the exact raw bytes on `req.rawBody`, affecting no other route (every other route already ignored it). `services/slackSignatureService.js#verifySlackRequest` recomputes `v0:<timestamp>:<rawBody>` over those exact bytes, using `crypto.timingSafeEqual` for the comparison, rejecting missing/malformed headers and timestamps outside a five-minute window (checked symmetrically — future-dated timestamps are rejected too) before ever touching the signature. `url_verification` requests go through the identical middleware — there is no bypass.

### Workspace-to-organization mapping

`services/slackEventsService.js#resolveWorkspace` calls the existing (previously unused) `oauthConnectionsService.getActiveConnectionByExternalAccount('slack', team_id)`, then re-verifies the organization is active via `supabaseService.getClientById`. Any lookup failure, `null` result, or inactive organization is treated as a safe rejection (200 ack, no processing) — never surfaced as an error. `channel_id` and any Slack-payload-supplied identifier are never used for tenant resolution.

**Deliberate ordering deviation from §4.3's pseudocode:** the spec's diagram lists dedup-on-`event_id` before team→organization mapping. This implementation maps first, then dedupes, because `slack_event_log` (as specified in §4.7) requires `client_id`/`connection_id` as `NOT NULL` — the dedup row cannot be inserted before the workspace is resolved. This costs one extra read (a redelivered event re-resolves the workspace before hitting the dedup check) but changes no correctness or security property: AIKB is still called at most once per event, and Slack still gets exactly one reply.

### Event acknowledgement and durability design

Chosen design: **a fast synchronous accept-and-enqueue call, backed by a Vercel Cron sweep** — not a bare fire-and-forget promise, and not a second bespoke job queue duplicating AIKB's Inngest architecture.

- `POST /events` persists the `slack_event_log` row, resolves the workspace, extracts the question, then makes one synchronous, short-timeout (`AIKB_ASK_TIMEOUT_MS`, default 4s) call to AIKB's `POST /ask`, which itself does nothing slower than an `inngest.send()` before responding. Only after that round trip does Relativity ack Slack. This keeps the whole request short enough for Slack's response budget without ever leaving an un-awaited promise running after the response is sent (there is no `waitUntil`-equivalent in this Vercel deployment).
- If the `/ask` call itself fails (AIKB down/slow), the row stays at `received`; Slack still gets a prompt `200` (redelivery would only hit the same dedup row, so there's nothing productive Slack retrying could do), and the row becomes eligible for the sweep.
- The actual RAG work happens in AIKB's existing, already-durable Inngest pipeline (Railway, always-on) — `step.run` granularity means a failed `deliver-to-relativity` retry never re-runs the OpenAI call.
- `GET /sweep` (Vercel Cron, `CRON_SECRET`-gated) finds rows stuck in `received`/`enqueued` past 30 seconds, retries up to 3 attempts total, and on exhaustion posts a best-effort "couldn't complete that request" fallback before marking the row `failed`. A `delivered` row is never touched by any retry path.
- AIKB's `onFailure` handler (after Inngest's own 3 retries are exhausted) proactively calls `/deliver` with an error payload, so a hard AIKB-side failure gets a Slack reply well before the sweep's 30-second/3-attempt window would otherwise catch it.

**Tradeoff accepted:** this is two small, composed mechanisms (a fast synchronous hop + a periodic sweep) rather than a single elegant abstraction, because the constraint is genuinely two-sided — Relativity has no background-execution primitive, and AIKB already has a correct one that should not be duplicated.

### Deduplication schema and state transitions

`slack_event_log` (`Relativity_Global`, migration `20260716_slack_event_log.sql`): the deployed table matches §4.7's design with one addition confirmed by re-reading the migration file directly — a `response_metadata jsonb` column not present in the original schema sketch, plus two supporting indexes (`idx_slack_event_log_sweep` on `(status, received_at)` for the sweep's lookup, `idx_slack_event_log_client_id` for per-client debugging) also not shown in §4.7's abbreviated `CREATE TABLE`. Neither addition changes the dedup/state-machine behavior described below. `UNIQUE (provider, external_event_id)` is the sole dedup mechanism — first delivery inserts and proceeds, a redelivery (or a true concurrent race) hits the constraint and is caught in application code as `{ inserted: false, row: <existing> }`, never a second AIKB or Slack call. States: `received -> enqueued -> answered -> delivered`, with `failed` reachable from any state. `claimForDelivery` is a single conditional `UPDATE ... WHERE status = 'enqueued'` — the exact "only the first delivery attempt proceeds" guarantee, verified under concurrent `Promise.all` calls in `test/slackEventLogService.test.js` and `test/slackDeliverService.test.js`.

### AIKB request/response contract

**Honest statement on cross-repo auth (per the objective's explicit instruction):** there is still no full signed `ServiceRequest` platform. `POST /ask` sits behind the existing, unchanged `x-api-key` gate (`requireApiKey`, still shared/static/non-expiring — not touched by this milestone) **plus** one small, additive, narrowly-scoped HMAC envelope (`services/serviceRequestAuth.js`, both repos), used only by `/ask` and the reversed `/deliver` callback. `SERVICE_REQUEST_SIGNING_SECRET` is new, distinct from `AIKB_API_KEY`/`SLACK_SIGNING_SECRET`/`INTEGRATION_CREDENTIAL_ENCRYPTION_KEY`. No `entitledCollectionIds`, no multi-origin principal registry, no asymmetric signing, no contract versioning — that full future platform remains unbuilt.

`/query`'s 293-line handler body was extracted verbatim (same intent classification, retrieval, generation, gap-detection, and message-persistence logic, same response shape) into `services/runKnowledgeQuery.js#runKnowledgeQuery({ clientId, question, sessionId, memberId, memberRole, origin, originMetadata, idempotencyKey })`. `/query` now calls it with `origin: 'portal'` and is behaviorally unchanged (verified: the full pre-existing regression path still passes; no portal-facing response field changed). `/ask` triggers `runKnowledgeQuery` with `origin: 'slack'` **asynchronously**, inside the new `knowledge/slack.question.requested` Inngest function — `/ask` itself never runs the pipeline inline, only enqueues it, keeping the route fast enough for Relativity's synchronous call.

Retrieval scope for Slack in this milestone is the same as the portal today (full organization scope, `client_id`-scoped exactly as `searchChunksWithTitleBoost` already enforces) — no "General"-only collection restriction was added, per §4.9/§2.5's explicit deferral.

**Idempotency:** migration `005_slack_origin_tracking.sql` adds nullable `origin`/`origin_metadata`/`idempotency_key` to `knowledge_chat_sessions` and `knowledge_gaps`, with a partial unique index on `idempotency_key` (NULLs excluded, so every existing/portal row is unaffected). `runKnowledgeQuery` checks for an existing session by `idempotencyKey` before creating a new one; if found, it replays the already-persisted assistant message instead of re-running retrieval/generation — a second guard independent of Relativity's own `slack_event_log` dedup, exactly as §4.19 asked for. `knowledge_gaps` got the same three columns for schema symmetry, but nothing writes to it automatically from either `/query` or `/ask` — gap persistence in this codebase has always been an explicit portal-only user action (`POST /gaps`), not something the shared pipeline does itself; Slack traffic never creates a second (or first) gap record, satisfying "AIKB remains responsible for knowledge-gap determination... Relativity must not create a second gap record" without inventing new gap-writing logic.

### Slack delivery behavior

`services/slackDeliverService.js` looks up the `slack_event_log` row by the echoed-back `idempotencyKey`, rejects a `clientId` mismatch (cross-tenant safety), atomically claims the row, re-verifies the connection is still `active` via the new `oauthConnectionsService.getConnectionById`, decrypts the bot token in memory immediately before use (`getDecryptedCredentialForConnection`, unchanged from Milestone 2), formats the message, and calls `services/slackDeliveryService.js#postMessage` (axios, 8s timeout, rejects non-2xx/invalid-JSON/`ok:false`, maps every failure to a safe internal code, never logs or re-throws the token). A revoked connection is never used — verified by test.

### Citation and fallback formatting

`services/slackAnswerFormatter.js`: `<answer>\n\nSources:\n• <title>\n• <title>`, deduplicated case-insensitively, capped at 5, only `fileName`/`title` ever surfaces (no document/chunk/storage ids), answers truncated at 3000 characters with an ellipsis. Knowledge-gap and temporary-failure fallbacks match the objective's exact copy, verified by test.

### Full Relativity test results

**247/247 passing** (up from 144/144 after Milestone 3) — 103 new tests across 10 new files, zero existing tests modified. `npm test` (`node --test`), no real Slack/Supabase/AIKB network call in any test.

### Full AIKB test results

**39/39 passing** (up from 8 after Milestone 1) — 31 new tests across 5 new files, zero existing tests modified (the retired-endpoint regression suite passes unmodified). `npm test` (`node --test test/*.test.js`), no real Slack/Supabase/OpenAI/Inngest network call in any test.

### Migration — exact filename and manual SQL steps

**Relativity:** `supabase/migrations/20260716_slack_event_log.sql` — additive only, one new table + two indexes, no destructive statements, does not touch `oauth_connections`/`oauth_credentials`/`oauth_states`. Manual step: run this file's contents in the Supabase SQL editor for the Global project, after `20260715_oauth_states.sql` if not already applied. Verification: `SELECT * FROM information_schema.tables WHERE table_name = 'slack_event_log';` and `\d slack_event_log` should show the `UNIQUE (provider, external_event_id)` constraint. Rollback: `DROP TABLE IF EXISTS slack_event_log;` (safe — no other table references it via FK from outside).

**AIKB:** `migrations/005_slack_origin_tracking.sql` — additive only, four `ALTER TABLE ... ADD COLUMN IF NOT EXISTS` statements plus two partial unique indexes, no destructive statements. Manual step: run in the AIKB Supabase SQL editor after `004_member_id.sql`. Verification: `SELECT column_name FROM information_schema.columns WHERE table_name = 'knowledge_chat_sessions' AND column_name IN ('origin','origin_metadata','idempotency_key');` should return all three. Rollback: `ALTER TABLE knowledge_chat_sessions DROP COLUMN IF EXISTS origin, DROP COLUMN IF EXISTS origin_metadata, DROP COLUMN IF EXISTS idempotency_key;` (and the same for `knowledge_gaps`) — safe, since nothing else depends on these columns yet.

Neither migration was executed as part of this work.

### Environment variables

**Relativity (new):** `SERVICE_REQUEST_SIGNING_SECRET` (shared with AIKB, 64 hex chars, same generation method as `INTEGRATION_CREDENTIAL_ENCRYPTION_KEY`), `SLACK_QUESTION_MAX_LENGTH` (optional, default 2000), `AIKB_ASK_TIMEOUT_MS` (optional, default 4000), `CRON_SECRET` (set automatically by Vercel when Cron is configured; the sweep route allows unauthenticated calls only when unset — i.e. local dev).

**AIKB (new):** `SERVICE_REQUEST_SIGNING_SECRET` (must be byte-identical to Relativity's), `RELATIVITY_API_BASE_URL` (e.g. `https://relativitysystems.ai`), `RELATIVITY_DELIVER_TIMEOUT_MS` (optional, default 8000).

`SLACK_SIGNING_SECRET` (Relativity, already provisioned in Milestone 3, now actually read for the first time).

### Manual Supabase steps

1. Run `Relativity/supabase/migrations/20260716_slack_event_log.sql` against the Global project. **Done.**
2. Run `aikb/migrations/005_slack_origin_tracking.sql` against the AIKB project. **Done.**
3. Confirm both with the verification queries above. **Done.**

### Manual Vercel steps (Relativity)

1. Set `SERVICE_REQUEST_SIGNING_SECRET`, `SLACK_QUESTION_MAX_LENGTH` (optional), `AIKB_ASK_TIMEOUT_MS` (optional) in the Vercel project's environment variables. **Done.**
2. ~~Vercel will set `CRON_SECRET` automatically once the `crons` entry in `vercel.json` is deployed and Cron is enabled for the project — no manual value needs to be chosen.~~ **Superseded — the `crons` entry was removed (see step 3); `CRON_SECRET` is consequently not being auto-provisioned by Vercel. See the Production deployment addendum below for the resulting operational and auth-surface implications of `GET /api/integrations/slack/sweep` no longer having a cron trigger.**
3. **Resolved, not as originally planned:** the deployed `crons` entry (`*/5 * * * *`, every 5 minutes) required a **Vercel Pro plan or higher** — this was confirmed against the actual project, which is on the **Hobby plan**, and the schedule was rejected outright (the deployment failed, it did not silently degrade to a slower cadence). The fix applied was to **remove the `crons` entry from `vercel.json` entirely** (commit `fix: remove unsupported Vercel cron schedule`), not to switch to a once-daily schedule. This was the pragmatic choice to unblock deployment; it means there is currently no automated retry sweep at any interval, daily or otherwise. See the Production deployment addendum below.
4. Deploy `feat/slack-events` to a preview/staging environment and confirm `POST /api/integrations/slack/events` answers Slack's `url_verification` challenge before enabling Event Subscriptions in the Slack dashboard (§4.16 — enabling it against a not-yet-deployed URL fails verification). **Done.**

### Manual Railway steps (AIKB)

1. Set `SERVICE_REQUEST_SIGNING_SECRET` (identical value to Relativity's), `RELATIVITY_API_BASE_URL`, `RELATIVITY_DELIVER_TIMEOUT_MS` (optional) in the Railway project's environment variables. **Done. Note: these three variables are read by `aikb/config/index.js` but are not documented in `aikb/.env.example`, which still only lists the pre-Milestone-4 Slack/API vars — a minor documentation gap found during this review, left as-is since only the report, not repository files, is in scope here.**
2. Deploy `feat/slack-ask-pipeline`. Confirm the Inngest dashboard registers the new `knowledge-slack-question-requested` function ("Answer Slack Question") after deployment (mirrors the existing `knowledge-document-ingest` registration pattern). **Done, but not on the first attempt — see "Implementation notes: manual Slack enablement and Inngest synchronization" below.**

### Manual Slack dashboard steps

1. **Do not enable Event Subscriptions until both repos are deployed and the Relativity URL below passes Slack's verification challenge.** **Followed — done after both deployments were confirmed live.**
2. Event Subscriptions Request URL: `https://relativitysystems.ai/api/integrations/slack/events`. **Set and verified — the Request URL passed Slack's `url_verification` challenge.**
3. Subscribe to exactly one bot event: `app_mention`. **Done.**
4. Bot Token Scopes: unchanged — `app_mentions:read`, `chat:write` only. Do not add `incoming-webhook`, `channels:history`, `groups:history`, `im:history`, `mpim:history`, `users:read`, or `users:read.email`. **Unchanged, confirmed.**
5. **Reinstall determination — now observed, not just predicted:** the specification's expectation (subscribing to a bot event is a separate configuration axis from OAuth scope grants, so no workspace reinstallation should be required) **held true in practice** — no reinstall was needed to start receiving `app_mention` events once the Request URL was verified and the event subscribed.

### Implementation notes: manual Slack enablement and Inngest synchronization

Two operational steps outside the application code were required to bring Milestone 4 fully online, and one production issue traced back to the second of them, not to application code:

- **Slack Event Subscriptions require manual enablement after deployment, and the Request URL must independently pass Slack's verification challenge before Slack will deliver any events.** This is not automatic on deploy — an operator must open the Slack app's configuration at api.slack.com, set the Request URL, and wait for Slack's own synchronous `url_verification` handshake (handled by `slackEventsService.handleUrlVerification`, §4.4) to succeed before the "Subscribe to bot events" section becomes usable. Until that handshake passes, Slack will not save the subscription and no `app_mention` events are delivered, regardless of whether Relativity's endpoint is deployed and healthy.
- **AIKB's Inngest functions must be (re-)synchronized after each AIKB deployment so newly added functions are registered with Inngest.** AIKB serves its functions in-process via `serve({ client: inngest, functions })` (`aikb/server.js`), and Inngest's own dashboard/cloud service must separately learn about a new function (here, `knowledge-slack-question-requested`, display name "Answer Slack Question", added in `aikb/inngest/functions.js`) before it will route events to it — deploying new code alone does not register the function.
- **The first production issue encountered after go-live was exactly this: Inngest had not yet synchronized and did not know about "Answer Slack Question," so `knowledge/slack.question.requested` events sent by `POST /api/knowledge/ask` were accepted (the route itself returned `{accepted: true}` correctly) but never picked up for processing.** This was a deployment/registration gap, not an application-code defect — `runKnowledgeQuery`, the `/ask` route, and the Inngest function body itself were all correct and passed their full test suites (§ Full AIKB test results). Once Inngest was manually resynchronized against the deployed AIKB instance, the function registered and began processing enqueued events normally. This is called out explicitly because it is easy to misdiagnose a stuck-`enqueued` `slack_event_log` row as an application bug when the actual cause is a registration step that has no automated trigger in this codebase.

### Known limitations

- The dedup-before-mapping vs. mapping-before-dedup ordering deviation from §4.3's pseudocode, documented above — no correctness impact, one extra DB read on redelivery of an event for an already-unmapped/malformed case.
- **Automated retry sweeping is currently disabled — an operational limitation introduced during production deployment, not a design change.** The Vercel Cron plan-tier constraint predicted above was confirmed for real: the project is on Vercel's Hobby plan, which rejects any cron schedule finer than daily, and the deployed `*/5 * * * *` entry caused the deployment itself to fail. The fix was to remove the `crons` entry from `vercel.json` outright (not downgrade it to a daily schedule), which unblocked deployment but means `GET /api/integrations/slack/sweep` is no longer invoked by anything on any schedule. The route and its retry logic (`slackEventsService.runDeliverySweep`) are still implemented and functional — they simply have no trigger. This is expected to remain the case until a scheduler is reintroduced (§ Security fix below covers only the endpoint's authentication, not its scheduling). Practical consequence: a `slack_event_log` row stuck at `received` because the initial synchronous `/ask` call to AIKB failed will never be automatically retried or resolved to a user-facing fallback message until a scheduler is reintroduced.
- ~~Because Vercel no longer manages a `crons` entry for this project, it also no longer auto-provisions `CRON_SECRET` — if no operator has separately set `CRON_SECRET` as a plain environment variable, the `/sweep` route's `if (config.cron.secret)` auth check (`routes/integrations/slack.js`) is skipped entirely, leaving `GET /api/integrations/slack/sweep` reachable without authentication by anyone who knows the URL.~~ **Fixed — see "Security fix: secure-by-default Slack sweep authentication" below.** The route is now fail-closed: an unset/blank `CRON_SECRET` returns `503 Service Unavailable` and runs no sweep logic, rather than skipping authentication.
- A malformed (non-JSON-parseable) request body to `POST /events` is rejected by Express's own JSON body-parser before reaching any Milestone 4 code; since `app.js` has no app-wide JSON-parse-error handler (pre-existing, not introduced by this milestone), such a request currently gets Express's default error response rather than the specified 200 ack. Slack's own Events API never sends malformed JSON in practice, so this is a theoretical gap, not an operational one — fixing it would mean adding an app-wide error handler, which is out of this milestone's scope.
- The sweep's best-effort failure-reply (`bestEffortFailureReply`) can itself fail silently (e.g. Slack is down) — the row is still marked `failed` regardless, per the objective's "no further Slack reply possible — logged internally" guidance for delivery failures. This code path is presently unreachable in production since nothing triggers the sweep (see above); it remains correct and tested, just dormant.
- No key rotation, no scheduled cleanup of old `slack_event_log`/`oauth_states` rows — same posture as Milestones 2-3, tracked as Should-Have hygiene, not a blocker.
- The full future signed `ServiceRequest` platform (multi-origin principal resolution, signed `entitledCollectionIds`, contract versioning) remains unbuilt, as stated in §4.10 and confirmed above.
- "General"-only collection scoping (§2.5 / renumbered Milestone 5) is not implemented — Slack retrieval scope is full-organization, identical to the portal today. This needs explicit confirmation before real Slack traffic is exposed if narrower scope is required. **This is the next milestone under consideration as of 2026-07-16.**
- **Resolved (verified in production):** the Inngest function's `onFailure` handler concern (below) and the function's registration were smoke-tested live — see "Implementation notes: manual Slack enablement and Inngest synchronization" above for the registration issue actually encountered (a synchronization gap, not a code defect in the handler itself). ~~The Inngest function's `onFailure` handler reads the original event via `event.data.event.data` per Inngest's documented internal `inngest/function.failed` event shape; this was not verified against a live Inngest deployment.~~
- AIKB's `.env.example` does not document `SERVICE_REQUEST_SIGNING_SECRET`, `RELATIVITY_API_BASE_URL`, or `RELATIVITY_DELIVER_TIMEOUT_MS`, even though `aikb/config/index.js` reads all three (added in this milestone). Found during this review; left uncorrected since only this report is in scope, not repository files.

### Unresolved follow-up work

- **Introduce a real scheduler for the delivery sweep** — a Vercel Pro upgrade, a once-daily Vercel Cron schedule, an external scheduler, or a Railway-side scheduled job in AIKB's already-always-on process — before Milestone 5 work begins, to restore the outage-recovery guarantee in Milestone 4's definition of done (§4.20, §12). This supersedes the original "confirm the Vercel plan tier" item below, which has now been confirmed. The authentication side of this (a scheduler needs a configured `CRON_SECRET` to call the now-fail-closed endpoint) is ready — see the security fix below; only the scheduling mechanism itself remains to be chosen.
- ~~Confirm the Vercel plan tier and adjust the sweep's cron schedule accordingly (or upgrade the plan) before relying on a 30-second sweep window in production.~~ **Done — confirmed Hobby plan, `crons` entry removed; see above.**
- ~~Confirm Slack's actual reinstall-on-event-subscription behavior against the live dashboard, and update this note with the observed result.~~ **Done — no reinstall was required; confirmed in the Manual Slack dashboard steps above.**
- ~~Close the `CRON_SECRET`/unauthenticated-`/sweep` gap identified above before reintroducing any scheduler.~~ **Done — implemented on branch `fix/secure-slack-sweep`; see "Security fix: secure-by-default Slack sweep authentication" below. Not yet committed, pushed, or deployed.**
- Document `SERVICE_REQUEST_SIGNING_SECRET`, `RELATIVITY_API_BASE_URL`, and `RELATIVITY_DELIVER_TIMEOUT_MS` in AIKB's `.env.example` (a repository change, out of scope for this report update).
- Decide whether "General"-only collection scoping (§2.5) is required before real Slack traffic goes live, given full-organization retrieval scope is what ships in this milestone. **This is the decision the next work session (Milestone 5) is expected to make.**
- Housekeeping follow-up already flagged in the Milestone 1 note: delete AIKB's now-fully-inert `routes/slack.js` (410 handler) and its `SLACK_BOT_TOKEN`/`SLACK_SIGNING_SECRET` config reads, now that Milestone 4's real endpoint is built and verified working in production. Still not done — `aikb/routes/slack.js` remains in place as the 410 stub, confirmed present as of this review.

### Production deployment (2026-07-16)

This section records what happened when Milestones 1–4 were actually taken to production, as distinct from the plan in §2.4 and the as-implemented note above (written before deployment). It exists to keep the "what was planned/built" account above intact while giving an honest, separate account of "what happened when it shipped," consistent with this report's practice elsewhere (e.g. the Phase 1 §6/§9/§10 "Historical finding" blocks) of layering status updates rather than silently rewriting prior sections.

**Timeline (from git history, both repositories):**
1. Milestones 1–3 (AIKB `chore/disable-legacy-slack`; Relativity `feat/encrypted-integration-credentials`, `feat/slack-oauth`) — implemented and merged to `main` in both repositories prior to Milestone 4.
2. AIKB `feat: add asynchronous Slack ask pipeline` (Milestone 4's AIKB half) and Relativity `feat: add Slack app mention processing` (Milestone 4's Relativity half, including the original `vercel.json` `crons` entry) — both merged to `main` and deployed.
3. The Vercel deployment carrying the `crons` entry either failed outright or deployed in a broken state, because the Hobby plan does not support a `*/5 * * * *` schedule.
4. Commit `fix: remove unsupported Vercel cron schedule` — the `crons` array was deleted from `vercel.json` in its entirety (not adjusted to a daily schedule); a follow-up `chore: trigger vercel deployment` commit confirms a redeploy was pushed to pick up the fix.
5. Slack Event Subscriptions were manually enabled against the now-live `POST /api/integrations/slack/events` URL and passed Slack's `url_verification` challenge.
6. The first real `app_mention` event was accepted by Relativity and forwarded to AIKB's `/ask` route, but the resulting `knowledge/slack.question.requested` Inngest event was not processed — Inngest had not yet synchronized against the newly deployed AIKB instance and did not know the "Answer Slack Question" function existed. This was diagnosed as an Inngest registration gap, not an application defect (see "Implementation notes" above); a manual Inngest sync resolved it.
7. Following the Inngest sync, the full flow — Slack mention → Relativity → AIKB → Relativity → Slack threaded reply — was exercised and confirmed working end-to-end in production.

**Net effect on what Milestone 4 actually delivers today, versus the spec:**
- Everything in §4.20's definition of done is met **except** the automated-retry half of "survives an AIKB outage without silently dropping the question" (§12, updated above) — that guarantee is currently provided only by AIKB's Inngest `onFailure` callback proactively notifying Relativity (§4.8's second layer), not by the Vercel Cron sweep, which is not scheduled.
- This is an **operational limitation of the current hosting tier**, not a defect in the Milestone 4 design or code — the sweep mechanism itself is implemented, tested, and would function correctly if invoked. Restoring full outage-recovery coverage is a deployment/infrastructure task (provision a scheduler), not a code change.

### Security fix: secure-by-default Slack sweep authentication (branch `fix/secure-slack-sweep`)

**Status:** Implemented on branch `fix/secure-slack-sweep` in the Relativity repository, created from `main` after the Milestone 4 production deployment above. Not committed, not pushed, no migration involved (none was needed), no packages installed, no production environment or Slack dashboard changes, no scheduler added, no Vercel Cron reintroduced, AIKB untouched.

**Root cause.** The original `GET /api/integrations/slack/sweep` handler in `routes/integrations/slack.js` only checked the caller's `Authorization` header when `config.cron.secret` was truthy:

```js
if (config.cron.secret) {
  const authHeader = req.headers['authorization'];
  if (authHeader !== `Bearer ${config.cron.secret}`) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
}
```

`config.cron.secret` reads `process.env.CRON_SECRET`, which Vercel previously auto-provisioned only when the project had a `crons` entry in `vercel.json`. Once that entry was removed (see "Production deployment" above — the Hobby plan rejected the `*/5 * * * *` schedule outright), `CRON_SECRET` is no longer set anywhere, `config.cron.secret` is `undefined`, the `if` block above is skipped entirely, and the sweep ran for **any** request with no authentication check at all. A secondary issue in the same code, of lower severity: the comparison that *did* run (`authHeader !== ...`) was a plain string `!==`, not a constant-time comparison, unlike every other secret-comparison path already established elsewhere in this codebase (`slackSignatureService.js`, `serviceRequestAuth.js`, both using `crypto.timingSafeEqual`).

**Fix implemented.** A new module, `services/cronSweepAuthService.js`, modeled directly on the existing `slackSignatureService.js` pattern already used for Slack Events signature verification (a pure, injectable-secret verification function plus a thin Express middleware wrapper, so every branch — including "not configured" — is unit-testable without mutating `process.env` or touching the cached `config` module):

- `verifyCronSweepAuth({ authorizationHeader, configuredSecret })` — pure function, returns `{ ok, reason }` where `reason` is one of `not_configured`, `missing_authorization`, `malformed_authorization`, `token_mismatch`, `ok`.
- `requireConfiguredCronSecret(req, res, next)` — Express middleware wrapping the function above; `next()` is called on exactly one path (`ok: true`), every other path returns a response and never calls `next()`, so the sweep service function cannot execute without successful authentication.

`config.cron.secret` (undefined, `null`, empty string, or whitespace-only) is now treated as **disabled**, not **open** — the endpoint returns `503 Service Unavailable` with `{ "error": "Slack event sweep is not configured." }` and runs no sweep logic. Only once a non-blank secret is configured does the endpoint require a matching `Authorization: Bearer <CRON_SECRET>` header, compared via `crypto.timingSafeEqual` with an explicit length check first (so a mismatched-length token is rejected safely rather than throwing). A missing header, a non-Bearer scheme (e.g. `Basic`), a case-mismatched scheme (`bearer`), an empty Bearer token, or an incorrect token all return `401` before the sweep service is ever called. `routes/integrations/slack.js`'s `/sweep` route now reads `router.get('/sweep', requireConfiguredCronSecret, async (req, res) => { ... })` — the route's own `config` import (previously used only for this check) was removed as no longer needed there.

**Files changed:**
- New: `services/cronSweepAuthService.js` — the verification function, the middleware, and the `REASON` enum.
- Modified: `routes/integrations/slack.js` — `/sweep` now uses `requireConfiguredCronSecret`; inline auth logic and the now-unused `config` import removed; route comment updated to reflect that no Vercel Cron currently calls this endpoint.
- Modified: `.env.example` — `CRON_SECRET` documentation rewritten to state the endpoint is secure-by-default when unset, document the required `Authorization: Bearer <CRON_SECRET>` header shape, and note it is independent of `SLACK_SIGNING_SECRET`/`SERVICE_REQUEST_SIGNING_SECRET`. No default value added.
- New: `test/cronSweepAuthService.test.js` — unit tests for the pure function and middleware (undefined/null/empty/whitespace secret, missing/malformed/wrong-scheme/wrong-token/correct-token Authorization, constant-time-safe handling of mismatched-length tokens, no secret ever present in a result or `REASON` value).
- New: `test/cronSweepDisabled.test.js` — HTTP-level test over the real Express app with `CRON_SECRET` genuinely unset in that process, proving `GET /sweep` returns exactly `503 { "error": "Slack event sweep is not configured." }` and that `slackEventsService.runDeliverySweep` is never invoked (asserted via a mock call-count of zero), regardless of what `Authorization` header (or lack thereof) is sent.
- Modified: `test/slackEventsRoutes.test.js` — extended the existing sweep-route coverage (which configures `CRON_SECRET`) to add a non-Bearer-scheme case, a token-less-`Bearer` case, and a correct-token case proving the sweep service is invoked exactly once and its result is returned in a `200` response.

**Test results:** full Relativity suite **278/278 passing** (up from 247/247 before this fix — 31 new tests across two new files plus additions to one existing file, zero existing tests modified in behavior). No real network, Supabase, Slack, or AIKB call in any new or existing test. Regression-checked: Slack Events URL verification, `app_mention` signature/filtering gating, the `/deliver` service-request envelope check, and Milestone 3's OAuth connect/status/disconnect routes all pass unmodified.

**Scope note:** this fix only changes the `/sweep` route's authentication. It does not add a scheduler, does not reintroduce Vercel Cron, does not change the Slack Events (`/events`) or delivery (`/deliver`) authentication, does not touch the retry/dedup state machine (`slack_event_log`, `slackEventLogService.js`, `slackEventsService.js`'s sweep/retry logic itself), and does not touch AIKB. Automated retry sweeping remains disabled — as documented above — until a scheduler is deliberately reintroduced; this fix only ensures that when that day comes (or in the meantime, if anyone probes the URL directly), the endpoint cannot run without a correctly configured secret and a correctly authenticated caller.

---

## Current System State (as of 2026-07-16, before Milestone 5)

This section summarizes the verified, completed state of the platform at the point Milestone 5 planning begins. It reflects direct verification against both repositories' current `main` branches (commit `5e0a607` in Relativity, `d652d37` in AIKB) — passing test suites (Relativity 247/247, AIKB 39/39), git history, and the production deployment account above — not just the design intent recorded earlier in this document.

- **Milestone 1 (neutralize the legacy AIKB Slack route) — complete.** `aikb/routes/slack.js` answers `url_verification` and returns `410 Gone` for every `event_callback`, with no channel-hash tenant derivation and no global-bot-token reply path. Deployed.
- **Milestone 2 (encrypted credential storage) — complete.** `oauth_connections`/`oauth_credentials` exist in Relativity's Global Supabase project; Slack bot tokens are stored only as AES-256-GCM ciphertext. Migrated and deployed.
- **Milestone 3 (Slack OAuth connection flow) — complete.** An organization admin can connect/disconnect exactly one Slack workspace via the portal (`/api/integrations/slack/{start,callback,status,disconnect}`), state-token/CSRF-protected, admin-only enforced. Migrated and deployed.
- **Milestone 4 (Slack app mentions and grounded AIKB answers) — complete, with one operational caveat.** `@RelativityBot` mentions are verified, mapped to an organization, answered via AIKB's shared `runKnowledgeQuery` pipeline, and delivered as threaded replies. Migrated and deployed to both Vercel (Relativity) and Railway (AIKB). **Caveat:** the automated retry sweep for a stuck event is currently unscheduled (see "Production deployment (2026-07-16)" above) — the primary path and AIKB's own `onFailure` notification both work; only the independent Vercel-Cron-driven backstop is inactive pending a scheduler decision.
- **End-to-end Slack workflow verified in production.** Slack mention → Relativity → AIKB → Relativity → Slack threaded reply was exercised against a real, connected Slack workspace and confirmed working, following the Inngest-synchronization fix described above.
- **OAuth verified.** The Milestone 3 connect/callback/status/disconnect flow was exercised against the real Slack app in production; the resulting connection and its encrypted bot token are consumed by Milestone 4's delivery path for the first time.
- **Event subscriptions verified.** Slack's Event Subscriptions Request URL (`https://relativitysystems.ai/api/integrations/slack/events`) passed Slack's `url_verification` challenge and is actively subscribed to the `app_mention` bot event only; no workspace reinstall was required to enable it.
- **Inngest processing verified.** After the manual Inngest synchronization fix, the `knowledge-slack-question-requested` ("Answer Slack Question") function registers and processes `knowledge/slack.question.requested` events end-to-end, including calling back to Relativity's `/deliver` endpoint.
- **Service authentication verified.** The additive HMAC `ServiceRequest` envelope (`SERVICE_REQUEST_SIGNING_SECRET`, `services/serviceRequestAuth.js` in both repositories) gates `POST /api/knowledge/ask` and, in reverse, `POST /api/integrations/slack/deliver` — confirmed live between the two deployed services, distinct from AIKB's unchanged, still-shared `AIKB_API_KEY` on every other route.
- **Threaded Slack replies verified.** Answers are posted via `chat.postMessage` into the originating thread (`thread_ts` or the mention's own `ts`), with citations rendered as a deduplicated, capped source list and no internal identifiers ever exposed.

**Ownership, confirmed unchanged by this review:**
- Relativity owns Slack integration end-to-end: OAuth, Events ingestion, signature verification, tenant mapping, delivery (`chat.postMessage`), the retry/dedup log, and (design-complete, currently unscheduled) retries.
- AIKB owns retrieval, reasoning, citations, conversations, and knowledge-gap detection — reached only through the shared `runKnowledgeQuery` pipeline, used identically by the portal (`origin: 'portal'`) and Slack (`origin: 'slack'`).
- Service-to-service authentication between the two repositories uses `SERVICE_REQUEST_SIGNING_SECRET`, scoped narrowly to the `/ask`/`/deliver` pair, layered on top of (not replacing) the pre-existing shared `AIKB_API_KEY`.
- Slack bot tokens are decrypted only in memory inside Relativity's `slackDeliverService`, immediately before a `chat.postMessage` call — confirmed by inspection that AIKB's codebase contains no bot-token handling outside its retired, config-only legacy references.

**What remains open, carried forward as assumptions belonging to future milestones (not resolved by this review, and intentionally not built ahead of need):**
- Milestone 5 (company-wide knowledge scope / optional "General"-only collection restriction, §2.5) — not started; this is the milestone under consideration as this section is being written.
- Milestone 6 (fuller knowledge-gap/conversation metadata polish, §2.6) — not started.
- Milestone 7 (direct messages, employee-level authorization) — explicitly deferred, no work done or expected imminently.
- The full future signed `ServiceRequest` platform (§4.10), RLS, structured `audit_log`, multi-org users, and every other Phase 2 §12 Should-Have/Future item — untouched, unaffected by Milestones 1–4.
- Restoring an automated delivery-sweep scheduler (see "Unresolved follow-up work" above) — flagged as work that should land before or alongside Milestone 5, not a Milestone 5 deliverable itself.
