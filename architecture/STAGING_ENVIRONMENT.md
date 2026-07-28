# Staging Environment — Setup and Verification Plan

## Status

**Planned — no cloud resources have been created from this document.** This is a setup and verification plan only: a design for a dedicated staging environment, written before any Global Supabase, AIKB Supabase, Slack app, Vercel, or Railway resource for it is provisioned. Nothing in this document should be read as describing something that exists today.

## Purpose

[CONNECTOR_FRAMEWORK.md](CONNECTOR_FRAMEWORK.md)'s Slack verification checklist and the Knowledge Collections verification item in [../roadmap/MASTER_ROADMAP.md](../roadmap/MASTER_ROADMAP.md) (Track A) both require exercising live behavior — a real Slack question, a real terminal `delivery_failed` row, a real Collections-scoped query — against a running deployment and a real database. The environment currently configured in both repositories' `.env` files (`GLOBAL_SUPABASE_URL`, `AIKB_SUPABASE_URL`, `SLACK_CLIENT_ID`/`SLACK_SIGNING_SECRET`) has **not been confirmed to be a non-production project**. Nothing distinguishes it from production in either `.env` file, in `.vercel/project.json` (a single Vercel project named `relativity`, no staging/production split visible), or anywhere else inspected. [EM10_5_STAGING_CHECKLIST.md](EM10_5_STAGING_CHECKLIST.md) already flagged this exact risk for Gmail staging validation ("confirm this points to a staging or dedicated test Supabase project, not production" — its own Part 0.3). The same risk applies here: none of the Slack/Collections verification steps that write data (a real question, a terminal delivery-failure row, a redaction callback, Collections-scoped ingestion) should run against an unconfirmed environment.

This document exists to close that gap generally — not just for Gmail — so that any future live-verification pass (Slack, Collections, Gmail, a future connector) has one dedicated, confirmed-safe environment to run against, set up once rather than re-litigated per milestone.

## Why not just reuse EM10.5's plan

EM10.5's checklist ([EM10_5_STAGING_CHECKLIST.md](EM10_5_STAGING_CHECKLIST.md)) is Gmail-specific and was written assuming *some* staging environment would exist by the time it ran — it doesn't itself describe how to provision one. This document is the general-purpose environment EM10.5, this Slack/Collections pass, and any future live-verification work should all run against, so the underlying Supabase projects, Slack app, and deployments are stood up once and reused, not recreated per milestone.

## Naming Conventions

To make "which environment is this" unambiguous at a glance, everywhere:

| Resource | Production (existing, unconfirmed status) | Staging (this plan) |
|---|---|---|
| Global Supabase project | *(current `GLOBAL_SUPABASE_URL`)* | `relativity-global-staging` |
| AIKB Supabase project | *(current `AIKB_SUPABASE_URL`)* | `relativity-aikb-staging` |
| Slack app | *(current `SLACK_CLIENT_ID`)*, whatever workspace(s) it's installed in today | `Relativity Systems (Staging)`, installed **only** into a dedicated test workspace created for this purpose |
| Relativity deployment | existing Vercel project `relativity` | new Vercel project `relativity-staging`, deployed from a `staging` branch |
| AIKB deployment | existing Railway service | a separate Railway **environment** (or project) named `staging` |
| Env files | `Relativity/.env`, `aikb/.env` (unchanged) | `Relativity/.env.staging`, `aikb/.env.staging` (new, gitignored, never committed) |

Rationale for a Railway **environment** rather than a second Railway project for AIKB: Railway environments share a project but keep separate variables/deployments, which matches how Vercel's own preview/production split already works for Relativity — pick whichever Railway supports with least friction at setup time; either satisfies the isolation requirement as long as staging and production never share a database connection string.

## Component-by-Component Plan

### 1. Dedicated Global Supabase staging project (`relativity-global-staging`)

- New Supabase project, not a schema/branch inside the existing production project — full isolation, not logical separation, since this project will hold real (if synthetic) client/member/OAuth-connection rows.
- Apply every migration from `Relativity/supabase/` in order, exactly as production received them.
- Record `SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY`/`ANON_KEY` for this project — these become `GLOBAL_SUPABASE_URL`/`GLOBAL_SUPABASE_SERVICE_ROLE_KEY`/`GLOBAL_SUPABASE_ANON_KEY` in `Relativity/.env.staging`.

### 2. Dedicated AIKB Supabase staging project (`relativity-aikb-staging`)

- New Supabase project, separate from both production and from the Global staging project above (mirrors production's Global/AIKB split — see [ADR-008](../decisions/ADR-008-CLIENT-AIKB-DATABASE-ROUTING.md) for why the two databases are already separate in production).
- Apply every migration from `aikb/migrations/` in order, including `006_knowledge_collections.sql` (the `match_knowledge_chunks` fail-closed function this plan exists partly to verify).
- Create the Storage bucket matching `AIKB_STORAGE_BUCKET` (default `aikb-documents`).
- Record this project's URL/service key as `AIKB_SUPABASE_URL`/`AIKB_SUPABASE_SERVICE_KEY` (aikb) and `AIKB_SUPABASE_URL`/`AIKB_SUPABASE_SERVICE_ROLE_KEY` (Relativity) in each repo's `.env.staging` — note the two repos already use different key names for the same value (`AIKB_SUPABASE_SERVICE_KEY` vs `AIKB_SUPABASE_SERVICE_ROLE_KEY`); preserve that as-is rather than "fixing" it as a side effect of this plan.

### 3. Separate Slack app in a dedicated test workspace

- Create a **new** Slack app (`Relativity Systems (Staging)`), not a second install of the production app — a distinct app gets its own Client ID/Secret/Signing Secret, so a staging credential can never accidentally authenticate against the production app's install.
- Bot scopes: exactly `app_mentions:read`, `chat:write` — identical to production's scope list (`Relativity/.env.example`'s documented scopes), so staging exercises the same permission surface, not a superset.
- Create or reuse a dedicated Slack workspace used **only** for this kind of testing — not a real company workspace, not the workspace any production client is connected through.
- OAuth redirect URI: `https://<relativity-staging-vercel-domain>/api/integrations/slack/callback`, matching byte-for-byte per Slack's exact-match requirement (the same caveat [EM10_5_STAGING_CHECKLIST.md](EM10_5_STAGING_CHECKLIST.md) notes for Gmail's redirect URI).
- Record Client ID/Secret/Signing Secret into `Relativity/.env.staging` only.

### 4. Relativity staging deployment (Vercel)

- New Vercel project `relativity-staging`, deployed from a `staging` branch (or Vercel's own preview-deployment mechanism pointed at a long-lived branch) — not a preview deployment of the `relativity` production project, since preview deployments of a production project can inherit production environment variables unless deliberately scoped, which defeats the isolation this plan requires.
- Environment variables: every key in `Relativity/.env.staging`, entered directly into the Vercel project's environment variable settings for this project (Vercel projects don't read a committed `.env.staging` file at deploy time — this file is for local runs against staging).

### 5. AIKB staging deployment (Railway)

- A `staging` Railway environment (or separate project) distinct from AIKB's production service.
- Environment variables: every key in `aikb/.env.staging`, entered into that Railway environment's variables.
- `RELATIVITY_API_BASE_URL` (aikb → Relativity callback base) must point at the `relativity-staging` Vercel deployment, not production — getting this wrong would make staging AIKB deliver real Slack messages via a production-configured callback path, or vice versa.

### 6. Staging-only environment files

- `Relativity/.env.staging` and `aikb/.env.staging`, both gitignored (extend each repo's `.gitignore` to cover `.env.staging` explicitly if not already covered by an existing `.env*` pattern — verify before assuming).
- Contain every variable each repo's real `.env` contains, pointed at the staging resources above — a full parallel configuration, not a partial override layered on top of production values.
- Never loaded by default: `dotenv.config()` in both repos' `config/index.js` loads `.env`, not `.env.staging`, so running against staging requires an explicit, deliberate step (below), not an accidental one from forgetting to switch files.

### 7. Guard: no live-verification command runs without an explicit staging flag

**Proposed, not implemented** — described here at the level ADR-010 describes an unbuilt decision, precise enough to build from later.

The existing automated test suites (`npm test` in both repos) already never touch a real database — every test observed uses an injected/fake service object (e.g. `createFakeSlackEventLogService` in `Relativity/test/slackDeliverService.test.js`), never a real Supabase client. No guard is needed there; this section is about the *live* verification steps in the checklist below (a real question through a real deployment, a real terminal `delivery_failed` row), which by definition do touch a real database, and are the ones that must never accidentally target production.

Planned mechanism, one version per repo:

- A small helper (`scripts/assertStagingEnvironment.js` in each repo) that any live-verification script must call first.
- It requires an explicit `STAGING_ENVIRONMENT=true` variable to be set (present only in `.env.staging` / the staging Vercel/Railway environment, never in production's variables) — absence is a hard failure, not a warning, consistent with the fail-closed default this codebase already uses elsewhere ([ADR-005](../decisions/ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md)).
- Defense in depth beyond the flag alone: the same helper also checks that `GLOBAL_SUPABASE_URL`/`AIKB_SUPABASE_URL` contain the known staging project refs (recorded once, after step 1/2 above, in the helper itself or a small checked-in allowlist) — so a `.env.staging` file that was accidentally pointed back at production values fails the check even if `STAGING_ENVIRONMENT=true` was set correctly. Both checks must pass; either one failing aborts before any request is made.
- Any script or checklist step that creates, mutates, or deletes data in the staging database (seeding, the delivery-failure exercise, cleanup/reset) should call this helper as its first action.

### 8. Seed data

One minimal, consistent seed for both the Slack/Collections pass and reusable by future passes:

- **One test client** in the Global staging project (not named anything resembling a real prospect/customer).
- **One test member** on that client, owner/admin role (needed to configure Slack collection access and to exercise both `?all=true` admin views and ordinary member views if useful later).
- **Two knowledge collections** on the AIKB staging project for that client: the default `General` collection (seeded automatically per client, per [ADR-005](../decisions/ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md)) plus one additional named collection to exercise non-default filtering.
- **A small number of representative documents** (2-3 short text/PDF files) ingested into each collection, distinct content per collection so a scoped query can be verified to return only its own collection's content — this is the concrete way to observe `match_knowledge_chunks`' fail-closed empty-array behavior actually working, not just present in the migration.
- **A real Slack workspace connection** for the test client (via the staging Slack app, step 3), with `slack_collection_access` configured to allow only one of the two collections — so a Slack question can be checked against the collection it should see and, separately, confirmed blind to the one it shouldn't.

### 9. Cleanup / reset after a verification pass

- Staging is expected to accumulate real rows every time this checklist runs (chat sessions, `slack_event_log` rows including `delivery_failed` terminal rows, ingestion jobs) — unlike production, there's no redaction-sensitivity reason to scrub it immediately, but it should not be left to grow unbounded across many passes.
- Reset approach: truncate the client-scoped tables touched by a run (`knowledge_chat_sessions`, `knowledge_chat_messages`, `slack_event_log`, `knowledge_slack_request_log`, `knowledge_gaps`) for the one seeded test client between passes, rather than dropping/recreating the whole project — keeps the Slack app connection, OAuth credentials, and collection configuration intact so the next pass doesn't repeat steps 3/8 from scratch.
- A full teardown (delete both staging Supabase projects, staging Vercel project, staging Railway environment, staging Slack app) is out of scope for routine use — only warranted if the staging environment itself needs to be rebuilt (e.g. after a long period of drift from production's schema).

## Revised Assessment of the Slack/Collections Verification Checklist

Against [CONNECTOR_FRAMEWORK.md](CONNECTOR_FRAMEWORK.md)'s 7-item checklist, with this staging environment as the execution target:

| # | Item | Status without staging | Plan once staging exists |
|---|---|---|---|
| 1 | Both test suites green | ✅ **Done** — 699/699 Relativity, 135/135 AIKB, fully mocked, no live database touched. Re-run is unaffected by staging existing or not. | No change needed. |
| 2 | Ask a real question via Slack; confirm `delivered`, `attempt_count = 1` | Blocked — needs a real Slack workspace | Run against the staging Slack app/workspace (step 3) and the seeded test client (step 8), through `relativity-staging`. |
| 3 | Retry-then-success / exhaustion via automated tests | ✅ **Done** — `Relativity/test/slackDeliverService.test.js` and `slackEventsService.test.js` inject failures deterministically, already part of the green suite. Checklist item 3 explicitly says not to force a real failure by invalidating a token — this is already satisfied without staging. | No change needed. |
| 4 | Confirm `question = null` and AIKB-side redaction on a terminal `delivery_failed` row | Blocked — needs a real terminal row and a real cross-repo redaction callback | **Controlled exercise against staging data**, not a production-token invalidation: point the seeded test client's staging Slack connection at a channel the staging bot lacks `chat:write` access to (or a deleted/archived channel id), so `chat.postMessage` fails organically through Slack's own API for a bounded, known reason — 3 attempts exhaust, `delivery_failed` is reached, and both Relativity's `question = null` and AIKB's redacted session/messages can be checked directly against the two staging databases. This never touches the production Slack app or a production bot token, per the "never invalidate a production token" rule the existing checklist item 3 already states — that rule extends to this item too. |
| 5 | Resend the same `event_id` after `delivery_failed`; confirm deduped, not reprocessed | ✅ **Already satisfied** — `Relativity/test/slackEventsService.test.js`'s `'a Slack resend after the original event already reached delivery_failed is still deduped, never reprocessed'` test exercises exactly this and is part of the green suite. The checklist's own wording explicitly allows an automated replay in place of a live Slack resend ("...or replay the same `event_id` in a test") — no live action needed here, staging or not. | No change needed — already covered. |
| 6 | `GET /api/integrations/slack/sweep` returns 404; no cron/scheduled job calls it | ✅ **Done** — confirmed via static search (route not registered, only a removal comment remains; no cron in either repo's Vercel/Railway config references it) and a live local boot returning `404`. | No change needed. |
| 7 | After test runs, reconnect/verify a fresh real question still works | Blocked — needs a real Slack workspace, and specifically needs to run *after* item 4's controlled-failure exercise | Once item 4's exercise completes on the staging connection, send one more real question through the (unaffected) staging Slack workspace and confirm a normal `delivered` reply — proves the bounded-retry testing didn't leave the staging connection in a broken state, exactly as the item requires, without ever touching production. |

**Live Collections exercise** (the piece of Track A's "Knowledge Collections verified" not covered by item 1-7 above, since `match_knowledge_chunks`' fail-closed behavior lives in SQL, not application code, and has no dedicated unit test): run against the staging AIKB project only, using the two-collection seed from step 8 — ask a Slack question scoped to the allowed collection (expect an answer with citations from that collection only) and, separately, confirm the disallowed collection's content is never returned even when directly relevant to the question asked.

## What Remains Manual (Cannot Be Automated From Here)

Provisioning steps below require a human with the relevant account access (Supabase org, Slack App Management, Vercel team, Railway project) — they are listed here as the explicit handoff points, not because they're expected to be done by tooling:

- Creating the two Supabase projects and obtaining their keys.
- Creating the Slack app and the dedicated test workspace, and installing the app into it.
- Creating the `relativity-staging` Vercel project and the staging Railway environment, and entering environment variables into each.

Once those exist and their credentials are available, the remaining steps (applying migrations, seeding data, running the verification checklist above) can proceed without further manual account-creation work.

## Related Documents

- [CONNECTOR_FRAMEWORK.md](CONNECTOR_FRAMEWORK.md) — the Slack verification checklist this plan exists to safely execute
- [EM10_5_STAGING_CHECKLIST.md](EM10_5_STAGING_CHECKLIST.md) — the Gmail-specific staging validation pass that first flagged the staging-vs-production risk this document generalizes
- [../decisions/ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md](../decisions/ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md) — the fail-closed mechanism the Collections portion of this plan verifies
- [../decisions/ADR-008-CLIENT-AIKB-DATABASE-ROUTING.md](../decisions/ADR-008-CLIENT-AIKB-DATABASE-ROUTING.md) — why Global and AIKB are already separate databases in production, mirrored here for staging
- [../roadmap/MASTER_ROADMAP.md](../roadmap/MASTER_ROADMAP.md) — Track A, item 1: the staging verification this plan unblocks
- [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md) — M14: the backlog item tracking this verification work
