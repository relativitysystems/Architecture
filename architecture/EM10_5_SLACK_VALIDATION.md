# EM10.5 — Slack Surface Validation

**Companion to [EM10_5_STAGING_CHECKLIST.md](EM10_5_STAGING_CHECKLIST.md). Not a new milestone.** EM10.5 remains the milestone; this is a second execution record under it, covering a surface the original checklist never touched.

The division of labour is exact:

| | Question it answers | Owns |
|---|---|---|
| [EM10_5_STAGING_CHECKLIST.md](EM10_5_STAGING_CHECKLIST.md) | **Did Gmail ingestion work correctly?** | Gmail label → policy → sync → normalize → chunk → embed → index. Every backend assertion about whether a document and its chunks exist. |
| **This document** | **Can Slack correctly retrieve and present the knowledge that Gmail ingestion produced?** | Slack event → tenant/collection resolution → AIKB retrieval → answer/citation formatting → delivery back to Slack. |

This document does **not** own Gmail ingestion. Where a scenario needs indexed content to change (a label removed, a policy edited, a member disconnected), the change is made through the portal/Gmail exactly as EM10.5 already specifies, EM10.5 proves the document and chunk state actually changed, and this document only asks: **does Slack now behave accordingly?** Backend assertions are not duplicated here unless they are needed to diagnose a Slack-side failure.

**Status: In progress — Part A complete, Part B underway (B5 run early, out of order, alongside the A.6 fix; B1, B2, B3, B4 passed 2026-08-30; B6 passed 2026-08-30 on weaker evidence than this document's own standard — see its own section; B7 passed 2026-09-02, surfacing and fixing Bug 5 along the way). B8–B9 not yet run. C1 and all of Part D are N/A/Removed (EL7C, 2026-08-30) — Slack identity linking and live-email lookup were removed entirely; see each section. C2–C6 not yet run.**

---

## Slack is a question surface, not an admin surface

Slack has no verb for connecting a mailbox, editing organization policy, disabling a member, changing sync mode, triggering a sync, or cleaning up content. Those are portal/admin-console actions and appear in this document only as **setup steps**. The only user-initiated Slack inputs that exist are:

- `@mention` of the bot in a channel (`app_mention`),
- a direct message to the bot (`message` with `channel_type: 'im'`).

~~the DM link command `link CODE` (EL7A, `slackEventsService.js` `LINK_COMMAND_PATTERN`)~~ — **removed (EL7C, 2026-08-30)**. Slack identity linking no longer exists; a `link CODE`-shaped message is now just an ordinary question like any other.

Group DMs (`mpim`) are explicitly refused (`OUTCOME.MPIM_UNSUPPORTED`). Every other event type is a silent HTTP 200 no-op.

---

## Stored Gmail knowledge vs. live Gmail lookup — historical framing, superseded by EL7C

**Superseded (EL7C, 2026-08-30).** This section originally distinguished stored Gmail knowledge (this document's subject) from live Gmail lookup through Slack (Part D, supplemental), because a DM from a linked user could contaminate a Part B result with a live tool call. **Slack's live-lookup path (EL7B) and the identity linking it depended on (EL7A) have been removed entirely** — see the [EL7C Implementation Record](LIVE_EMAIL_LOOKUP.md#el7c--slack-live-email-access-removal). `slackEventsService.js` now unconditionally sends `memberId: null, emailLookupAvailable: false` for every Slack request, channel or DM alike — there is no code path left by which a live tool call can occur through Slack, so the contamination risk this section warned about no longer exists, and neither does Part D (see its own N/A/Removed note).

**Practical effect: every Part B scenario, whether run as a channel `@mention` or a DM, is now equally safe** — no evidence-rules ceremony is required to prove a stored-knowledge answer wasn't contaminated by a live lookup, because no live lookup is reachable at all. The historical detail below is retained for context, not as a live procedure.

~~These are two different features that can produce superficially similar Slack answers. Confusing them will silently invalidate results.

| | **Stored Gmail knowledge** (this document's primary subject) | **Live Gmail lookup** (supplemental, Part D) |
|---|---|---|
| What it is | Normal AIKB vector retrieval over `knowledge_chunks` — Gmail messages that were ingested, chunked, embedded, and indexed by EM1–EM10 | EL-series tool calling: the model invokes `search_email_messages` / `get_email_content` against a live mailbox at question time |
| Owning design | [EMAIL_INGESTION.md](EMAIL_INGESTION.md) | [LIVE_EMAIL_LOOKUP.md](LIVE_EMAIL_LOOKUP.md), [ADR-010](../decisions/ADR-010-LIVE-TOOL-CALLS-ORCHESTRATED-BY-AIKB.md) |
| Requires identity linking | No — client-scoped | Yes — resolves `requestingMemberId` from `slack_user_links` |
| Citation rendering | Plain source title | Suffixed `(Live)` (`slackAnswerFormatter.js`, `source.live === true`) |
| Validated by | **Parts A–C of this document** | Part D here (surface smoke check only) and [EL10](LIVE_EMAIL_LOOKUP.md#el10--staging-validation-and-security-testing) (backend, authoritative) |

**A live lookup must never be allowed to make a stored-ingestion test pass.** For every Part B scenario, the answer must be provably sourced from indexed AIKB content.

### The structural control that makes this easy

`services/slackEventsService.js` resolves a member — and therefore offers email tools — **only for direct messages**:

```js
let requestingMemberId = null;
let emailLookupAvailable = false;
if (isDirectMessage) { … }
```

with the code's own comment stating the intent outright: *"channel @mentions never offer the tools regardless of link status" is a hard MVP policy, not an oversight*.

**Therefore: run every Part B stored-knowledge scenario as a channel `@mention` by default.** A live tool call is structurally unreachable on that path, so a correct answer is proof of indexed retrieval without needing to inspect anything. B3 deliberately re-runs the same questions over DM, where contamination *is* possible, and that is exactly the scenario where the evidence rules below must be applied in full.

### Evidence rules — proving an answer came from `knowledge_chunks`

For any Part B scenario run over DM, or any result you doubt, capture at least two of:

1. **Channel-vs-DM control.** The same question answered identically via channel `@mention` (tools structurally unavailable) is strong evidence the DM answer did not need them.
2. **`email_ingestion_events` audit rows.** EL7B/EL9 audit writes extend this table rather than adding a new one (migration `20260731_email_live_lookup_el7b_audit.sql`, per LIVE_EMAIL_LOOKUP.md §10.1) with `origin`, `origin_metadata`, `tool_name`, `result_count`, `provider_latency_ms`. **A live tool call produces a row with a non-null `tool_name`. No such row in the request window = no live call occurred.** This is the definitive check.
3. **Citation shape.** A `(Live)` suffix on any source means a live source was used. Its absence means every cited source was stored.
4. **`emailLookupAvailable`.** Logged/derivable per request; `false` means no tools were offered at all.
5. **Unlinked-user control.** An unlinked Slack user can never trigger a live lookup (`getLinkedMember` returns nothing → `requestingMemberId` stays null).

**Expected result for every pure ingestion test: the indexed Gmail document is retrieved, and no live Gmail tool call was required or made.**~~

---

## Known constraints to account for before running

**Slack citations carry no deep link, by design.** `services/slackAnswerFormatter.js` emits human-readable title text only — `formatCitations` pushes `title` (or `"subject" from sender`), never a URL, per the module's explicit contract that *"document IDs, chunk IDs, storage paths, signed URLs … must never reach this module's output."* EM10.5 Scenario 2's "a working 'Open in Gmail' link" **is not an applicable acceptance criterion on this surface.** Do not fail a Gmail ingestion scenario over it. If the absence is judged a product gap, log it in Part E's documentation/product follow-up list.

**EM10.5 Bug 6 is still open.** The portal's sync-run summary reports only ingest-side counts, so a sync that tombstones documents still displays `0 imported, 0 skipped, 0 failed`. **Do not use the sync summary alone as evidence that a state change did or did not happen.** Use `email_ingestion_events`, actual document/chunk state, and Slack's own retrieval behavior. This applies to B7, B8, and B9.

**`slack_collection_access` fails closed.** See A.6 — this is the single most likely cause of a false failure in this document.

---

# Part A — Prerequisites and surface readiness

Everything in Part A is account/credential/configuration setup. No code changes are in scope.

## Part A execution record — 2026-08-09

**Original result: FAIL at A.6. Part B was blocked and must not be started.**

**Update — 2026-08-24: A.6 fixed, A.7 passed. Part B is unblocked.** See the A.6/A.7 sections below for detail.

Verified by direct inspection of both production Supabase projects, the two repositories' source, and `Relativity/.env`. Not verified from the Slack admin UI (no access from the execution environment) — where a Slack-side fact was needed, live database evidence was used instead and is noted as such.

| Item | Result | Summary |
|---|---|---|
| A.1 Environment declaration | ☐ **Open — needs sign-off** | Everything below points at **production**. No staging exists. See A.1. |
| A.2 Slack app | ⚠ **Pass with a blocking caveat** | Live install has every needed scope, but the code's reconnect path would downgrade it — Bug 1. |
| A.3 Workspace and users | ⚠ **Partial — needs confirmation** | Workspace `T0B7BDF7J3E` connected. Both "member" mailboxes belong to one person. |
| A.4 Environment configuration | ☑ **Pass** | All values present and consistent; recorded in A.4. |
| A.5 Test client and known content | ⚠ **Pass with two divergences** | All 3 Gmail docs indexed with 1 chunk each, but state contradicts EM10.5's record — Bugs 2 and 3. |
| A.6 Slack collection access | ☑ **Fixed — 2026-08-24** | `c033f615` ("General") added to the allow-list alongside `3b15ddc1` ("Slack"). Bug 4 closed. |
| A.7 Baseline sanity check | ☑ **Pass — 2026-08-24** | `@mention`, question about a `portal_upload` document in General, answered correctly with a sources block. |

**The single blocking issue**: this client's Slack allow-list permits collection `3b15ddc1` ("Slack"), which contains 0 indexed documents. All 10 of the client's indexed documents — including all 3 Gmail test documents — are in `c033f615` ("General"), which is **not** allow-listed. Per `aikbAskClient.js`'s fail-closed contract this is correct, intended behavior, not a defect in retrieval; it simply means every Part B scenario would return a knowledge gap for a configuration reason. This is precisely the false-failure mode A.6 exists to catch, and it was caught before a single content scenario ran.

**Resolved — 2026-08-24.** `c033f615` was added to the allow-list (kept alongside `3b15ddc1`, not replacing it — see A.6's updated section below for the product-decision note this leaves open). A.7 was then run and passed. Part B is now unblocked.

---

### A.1 Environment declaration (do this first, in writing)

EM10.5's own Part 0.3 required a staging Supabase project, and its Scenario 1 records that it ran against production anyway. Do not repeat that silently.

- [ ] Environment used for this pass: **Production.** No staging environment exists ([STAGING_ENVIRONMENT.md](STAGING_ENVIRONMENT.md) — no cloud resources created). Evidence: `SLACK_REDIRECT_URI = https://relativitysystems.ai/api/integrations/slack/callback`; `AIKB_API_BASE_URL = https://aiknowledgebaseinngest-production.up.railway.app`.
- [ ] Supabase projects (Relativity / AIKB) in use: **both production.** Verified 2026-08-09 by direct query.
- [ ] Slack workspace in use: **`T0B7BDF7J3E`**, connected 2026-07-16 to client `72e78cfe-218d-4927-8e26-a392e43846f4` ("Relativity Systems"), `status = active`, never revoked.
- [ ] Client under test: **`72e78cfe-218d-4927-8e26-a392e43846f4` — "Relativity Systems", `is_active = true`.** This is the company's own live client record, not a synthetic test tenant. Three other real clients exist in the same database (Scribed.ai, Dawna, tat) and are **not** in scope; B6 can use one of them as out-of-scope content.

**Destructive scenarios in this document — B5 (collection allow-list removal), B7 (label removal), B8 (policy change), B9 (member disable, disconnect with cleanup) — must either run against a non-production test client/environment, or be explicitly acknowledged below as running against production data that is understood to be isolated test data.**

- [ ] Blast radius understood and recorded: ______________________
- [ ] Signed off by: ______________________

### A.2 Slack app

- [x] A Slack app exists and is installed — `oauth_connections` holds an `active`, never-revoked `slack` row for team `T0B7BDF7J3E`.
- [x] **Bot token scopes on the live install are sufficient.** `scopes_granted` records all eleven of: `app_mentions:read`, `chat:write`, `im:history`, `im:read`, `im:write`, `channels:read`, `channels:history`, `groups:read`, `groups:history`, `users:read`, `users:read.email`. Every scope Part B, C, and D need is present **on the current token**.
- [x] **Event Subscriptions are working** — proven by delivery, not by inspecting the Slack UI: `slack_event_log` holds 12 `app_mention` and 3 `message` events at terminal status `delivered`, most recently 2026-07-21. Both `app_mention` and `message.im` are therefore subscribed and reaching the deployed endpoint. One row remains stuck at `enqueued` from 2026-07-16, predating [ADR-007](../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md)'s bounded-retry work; it is stale, not evidence of a current defect.
- [x] Signing secret set and matching (`SLACK_SIGNING_SECRET`, 32 chars).
- [ ] ⚠ **Do not disconnect and reconnect Slack during this pass — see Bug 1.** `services/slackService.js:17` requests only `REQUIRED_SCOPES = ['app_mentions:read', 'chat:write']`. The live token's broader grant predates that constant or came from the app's own manifest; a reconnect through the current code path would request two scopes and drop `im:history`/`im:read`/`im:write`, silently killing DM delivery and with it C1 identity linking, B3, and all of Part D.

### A.3 Test workspace and users

- [ ] ⚠ **Not dedicated** — `T0B7BDF7J3E` is the company's live workspace, not a throwaway. Combined with A.1's production finding, every destructive scenario needs the A.1 sign-off before it runs.
- [ ] ⚠ **Two Slack users not yet confirmed.** The client has exactly two `client_members`, both `active` with `search_enabled = true`: owner `65900664-…` (mailbox `tenzin@relativitysystems.ai`) and member `4c8acdee-…` (mailbox `10zinsteel@gmail.com`). **Both mailboxes belong to the same person.** That is workable for retrieval and delivery scenarios, but it weakens B6 and D3 — a cross-member isolation result proved with two accounts owned by one operator is softer evidence than two genuinely separate users. Record which real Slack user ids will act as A and B: ______________________
- [x] `slack_user_links` is **empty** — no Slack user is linked to any member yet. C1 has never been run. Consequences: Part D cannot run until C1 succeeds, and B3's DM leg will run unlinked (which is a *clean* stored-knowledge control, so run it that way first deliberately).
- [ ] At least one public channel the bot is a member of, plus DM access for both users — the 12 delivered `app_mention` events prove a channel existed as of 2026-07-20; confirm it still does.

### A.4 Environment configuration

**Verified 2026-08-09 — A.4 passes.**

- [x] `Relativity/.env`: `SLACK_CLIENT_ID` (29 ch), `SLACK_CLIENT_SECRET` (32 ch), `SLACK_SIGNING_SECRET` (32 ch), `SLACK_TOKEN_ENCRYPTION_KEY` (64 ch) all set; `SLACK_REDIRECT_URI = https://relativitysystems.ai/api/integrations/slack/callback`. The redirect URI's host matches the workspace that has been delivering events, so the app is the live one, not a stale entry.
- [x] `SLACK_QUESTION_MAX_LENGTH` = **2000** (explicitly set; matches `config/index.js`'s own default). C6(c) needs a question longer than 2000 characters.
- [x] `SLACK_DELIVERY_MAX_ATTEMPTS` / `SLACK_DELIVERY_BACKOFF_MS` are **both unset**, so `config/index.js:167` defaults apply: **3 total attempts, backoff `2000,5000` ms**. C5 asserts against these values.
- [x] `SERVICE_REQUEST_SIGNING_SECRET` — **matches** byte-for-byte between `Relativity/.env` and `aikb/.env` (64 chars each). The `/deliver` callback will verify ([ADR-004](../decisions/ADR-004-SIGNED-SERVICE-REQUESTS.md)).
- [x] `AIKB_API_BASE_URL = https://aiknowledgebaseinngest-production.up.railway.app` (the config key is `AIKB_API_BASE_URL`, not `AIKB_BASE_URL`).
- [ ] Both `node server.js` processes start cleanly — **not re-checked in this pass**; the deployed services are what Slack actually talks to, and their liveness is better evidenced by A.2's delivered events than by a local boot.

### A.5 Test client, members, and known Gmail content

- [x] Same client and members as EM10.5 (client `72e78cfe-…`; members `65900664-…` owner and `4c8acdee-…`).
- [x] The Slack workspace is connected to this client — `oauth_connections` row `c112faea-…`, `status = active`.
- [x] **All three Gmail documents are indexed with real chunks** (AIKB, verified 2026-08-09):

| Document | AIKB document id | Status | Chunks | Collection | `updated_at` |
|---|---|---|---|---|---|
| `Project Phoenix Onboarding SOP.txt` | `02f6506c-…` | `indexed` | 1 | `c033f615` General | 2026-08-08 15:03:04 |
| `Customer Refund Policy.txt` | `eef87a0b-…` | `indexed` | 1 | `c033f615` General | 2026-08-08 15:03:04 |
| `Weekly Sales Meeting Agenda.txt` | `83039afe-…` | `indexed` | 1 | `c033f615` General | **2026-08-08 15:20:30** |

  Reference questions for Part B, with EM10.5's known-good answers: Project Phoenix onboarding steps; weekly sales meeting → **Thursday 9:00 AM**; refund policy → **30 days**.

- [ ] ⚠ **Divergence 1 — `Weekly Sales Meeting Agenda` is live when EM10.5 says it should not be (Bug 2).** EM10.5's final 2026-08-08 retest records the managed label removed from this message and the Full Scan correctly *not* re-importing it. The document is nevertheless `indexed` with 1 chunk, `updated_at` **17 minutes after** the other two documents' 15:03:04 import. Something re-indexed it at 15:20:30 that EM10.5's record does not account for. **B7 depends on a known starting label state and cannot be trusted until this is reconciled.** Confirm the actual current Gmail label state by hand before running B7: ______________________
- [ ] ⚠ **Divergence 2 — nine active `email_connections` rows for two mailboxes (Bug 3).** Owner `65900664-…` has **6** rows for `tenzin@relativitysystems.ai`; member `4c8acdee-…` has **3** for `10zinsteel@gmail.com`. Every one has `sync_enabled = true`. Two of the member's three have `managed_label_id = null`. B9 (and EM10.5's own Scenarios 5, 8, 9) assume one connection per member — a disable or disconnect exercised against one row may leave five others syncing, which would read as a correctness failure when it is really connection-row duplication.

### A.6 Slack collection access — the fail-closed prerequisite

`slack_collection_access` gates which AIKB `knowledge_collections` a Slack question may search, client-wide (`services/slackCollectionAccessService.js`). The contract is explicit in `services/aikbAskClient.js`:

> `allowedCollectionIds` — *"Always an explicit array (possibly empty — empty means "search nothing", not "search everything"), never omitted, so a caller can never accidentally fall back to an unrestricted search."*

**`allowedCollectionIds = []` means Slack searches nothing.** If the collection holding the Gmail documents is not allow-listed, every scenario in Part B returns a knowledge gap and will look like an ingestion or RAG failure when it is purely a configuration state. This is [ADR-005](../decisions/ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md) working as designed.

### ☒ A.6 FAILED — verified 2026-08-09

- [x] AIKB collection id holding the Gmail test documents: **`c033f615-57ea-42a0-956a-3d9ba5875bf5`** — named "General", `is_default = true`, client `72e78cfe-…`.
- [x] The full current Slack allow-list, recorded so B5 can restore it exactly: **exactly one row — `3b15ddc1-5453-438d-a342-f9296924e66c`**, named "Slack", `is_default = false`.
- [x] ☒ **The Gmail collection is NOT allow-listed.** `c033f615` ≠ `3b15ddc1`.

**Scope of the failure is wider than Gmail.** Every one of this client's **10** indexed documents — 3 Gmail plus 7 `portal_upload` — lives in `c033f615`. The allow-listed `3b15ddc1` holds **zero** indexed documents.

> **Slack can currently retrieve nothing at all for this client.** Not "no Gmail content" — no content.

This is `slack_collection_access` behaving exactly as [ADR-005](../decisions/ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md) and `aikbAskClient.js` specify. It is a **configuration state, not a defect** in retrieval, ingestion, or RAG. It is logged as Bug 4 to track the fix, not to attribute blame to code.

**Consequence**: A.7 would fail, and every Part B scenario would return a knowledge gap for a reason that has nothing to do with what they are testing. Had A.6 not been run first, B1 through B4 would have produced four convincing, entirely false ingestion failures.

**Fix required before proceeding** (owner/admin, portal Slack panel → `PUT /api/integrations/slack/collections`): add `c033f615-57ea-42a0-956a-3d9ba5875bf5` to the allow-list. Whether `3b15ddc1` stays alongside it is a product decision — see E.2. Re-run A.7 immediately after.

### ☑ A.6 FIXED — 2026-08-24

- [x] `c033f615-57ea-42a0-956a-3d9ba5875bf5` ("General") added to the allow-list via the portal's Slack panel.
- [x] **Product decision (see E.2's open candidate): both collections are kept allow-listed** — `3b15ddc1` ("Slack") remains alongside `c033f615` ("General"), rather than General replacing it. `3b15ddc1` still holds 0 documents as of this update, so it currently contributes nothing to retrieval either way; it was left in place rather than removed. The underlying naming-split question E.2 raises (should Gmail-ingested mail route to a Slack-specific collection instead of the client default?) is **still open** — this fix unblocks validation, it does not resolve that product question.
- [x] Verified via the fail-closed round-trip below (logged formally as B5, run here alongside the fix): removing General from the allow-list produced a clean knowledge gap; re-adding it restored the correct answer. This is direct proof the allow-list gate itself works, not just that content exists.

### A.7 Baseline sanity check — **gate: do not start Part B until this passes**

**Setup**: A.1–A.6 complete.
**Action**: in the test channel, `@mention` the bot with a question about a **known indexed document that is not Gmail-sourced** (any portal-uploaded document already in an allow-listed collection).
**Expected**: Slack returns the known correct answer, with a `Sources:` block naming that document.
**Evidence to capture**: the answer text, the sources block, and the round-trip latency.
**Why this gate exists**: it proves the entire Slack → AIKB → retrieval → delivery path is live and the allow-list is non-empty, *before* any Gmail-specific result becomes ambiguous. A knowledge gap in B1 is unattributable without it.
**Result**: ☑ **Pass — 2026-08-24.** Run by hand after Bug 4's allow-list fix, using one of the `portal_upload` documents in `c033f615` (a non-Gmail document, per the scenario's own requirement). The bot returned the correct answer with a `Sources:` block naming that document. Exact question/answer/sources text and round-trip latency were not captured verbatim for this record.
**Notes**: this passing means the entire Slack → AIKB → retrieval → delivery path is confirmed live and the allow-list is non-empty. Part B is unblocked as of this result.
**Bugs found** (#): none — closes out Bug 4

---

# Part B — Stored Gmail knowledge through Slack

**Run every scenario in this Part as a channel `@mention` unless the scenario says otherwise** — see "The structural control" above. Live email tools are structurally unavailable on that path, so any correct answer is proof of indexed AIKB retrieval.

### B1 — A known ingested Gmail document is retrievable in Slack

**Setup**: A.5's three documents indexed; A.6 allow-list correct; A.7 passed.
**Action**: `@mention` the bot in the test channel with each of A.5's three reference questions, one at a time.
**Expected**: each returns the known-good answer from the table in A.5, followed by a `Sources:` block naming the correct Gmail-backed document (rendered as its `title`, or as `"Subject" from Sender`).
**Evidence to capture**: answer text and sources block per question; confirmation of **no** `email_ingestion_events` row with a non-null `tool_name` in the request window; absence of any `(Live)` suffix.
**Result**: ☑ Pass ☐ Fail ☐ Partial
**Notes**: All three questions asked as channel `@mention`s in `C0B75NXAMGW`, 2026-08-30 ~04:33 UTC. Each answered correctly with the right fact and cited exactly the right document: Project Phoenix onboarding steps → "Project Phoenix Onboarding SOP"; weekly sales meeting → Thursday 9:00 AM, "Weekly Sales Meeting Agenda"; refund policy → 30 days (plus genuine extra detail from the source email: fully-delivered digital services excluded, 5-business-day processing, order number required), "Customer Refund Policy". No `(Live)` suffix on any source. `slack_event_log` shows exactly 3 `app_mention` events, all `status: delivered`, same channel, no thread, each completing in ~6-7s. `email_ingestion_events` has zero rows in the 04:33:15–04:34:06 UTC request window — no live tool call occurred (a live call would leave a row with a non-null `tool_name`).
**Bugs found** (#): None

### B2 — Portal vs. Slack parity

*The central scenario of this document: proving both surfaces consume the same indexed knowledge correctly.*

**Setup**: B1 passed.
**Action**: for each of A.5's three reference questions, ask the **exact same question text** in the portal chat and in Slack, and compare.

| Question | Portal answer | Slack answer | Same source doc? | Unrelated citations? | Consistent? |
|---|---|---|---|---|---|
| Project Phoenix onboarding steps | 5-step process: create client account → upload onboarding docs → schedule kickoff within 2 business days → assign implementation specialist → send welcome email | Identical 5 steps, same order | ☑ "Project Phoenix Onboarding SOP" | ☑ None | ☑ |
| Weekly sales meeting time | Thursday 9:00 AM | Thursday 9:00 AM | ☑ "Weekly Sales Meeting Agenda" | ☑ None | ☑ |
| Refund policy | 30 calendar days; fully-delivered digital services excluded; order number required; 5-business-day processing | Same four facts, same wording | ☑ "Customer Refund Policy" | ☑ None | ☑ |

**Expected**:
- materially consistent answers (wording may differ; facts, and any figure such as "Thursday 9:00 AM" or "30 days", must not);
- the **same underlying Gmail-backed knowledge document** cited on both surfaces;
- no unrelated citations on either surface;
- **no knowledge gap in Slack for anything the portal can retrieve** — if Slack gaps where the portal answers, suspect A.6's allow-list first, then log it;
- collection restrictions behave as expected on both.

**Evidence to capture**: the completed table; both surfaces' full source lists side by side; for any divergence, the allow-list state at the time.
**Result**: ☑ Pass ☐ Fail ☐ Partial
**Notes**: All three questions run with identical text on both surfaces, 2026-08-30. Facts matched exactly on every question (no wording-vs-fact ambiguity to resolve). The only divergence is expected and by design: the portal renders an "Open in Gmail" deep link per source, Slack does not (`slackAnswerFormatter.js` never emits one — see this document's "Known constraints" section; EM10.5 Scenario 2's deep-link acceptance criterion is portal-only). No knowledge gap on either surface; both correctly retrieved all three Gmail-backed documents from the allow-listed `c033f615` collection.
**Bugs found** (#): None

### B3 — Channel `@mention` vs. DM parity

**Setup**: B1 passed. ~~Slack User A linked (run C1 first if not).~~ **Superseded (EL7C, 2026-08-30)**: Slack identity linking no longer exists — see C1's own N/A/Removed note. No setup beyond B1 is needed.
**Action**: ask each of A.5's three reference questions **again as a DM** to the bot. Compare to B1's channel results.
**Expected**: both surfaces retrieve the same stored AIKB knowledge, subject to the same collection access. DM answers match the channel answers on facts and cited document.
~~**This is the one Part B scenario where live-lookup contamination is possible** — a DM from a linked user with an active mailbox may have tools offered. Apply the full evidence rules.~~ **Superseded (EL7C)**: live-lookup contamination from a DM is now structurally impossible — `slackEventsService.js` always sends `memberId: null, emailLookupAvailable: false` to AIKB, with no linking mechanism left to resolve a member from. This scenario now exists purely to confirm DM and channel `@mention` share the identical stored-knowledge retrieval path, nothing more.
**Evidence to capture**: per DM question — the answer and sources block, and confirmation it matches the corresponding B1 channel result. The old live-lookup checks (`(Live)` suffix, `email_ingestion_events` tool-call rows) are no longer meaningful evidence here post-EL7C — retained only as a sanity check, not as a required pass condition.
**Result**: ☑ Pass ☐ Fail ☐ Partial
**Notes**: All three reference questions DM'd to the bot in `D0B7BF7N4JG`, 2026-08-30 19:46:40–19:47:18 UTC. `slack_event_log` shows exactly 3 `message` events, all `status: delivered`, single attempt each, completing in 6–11s. Facts and cited documents matched B1's channel answers exactly (Thursday 9:00 AM, 30 days, the onboarding steps). The one observed difference — no "Open in Gmail" link on the DM answers — is expected, not a defect: per this document's "Known constraints" section, Slack citations never carry a deep link on either surface, channel or DM. Zero `email_ingestion_events` rows in the request window, consistent with EL7C (no live tool call is reachable from Slack at all anymore).
**Bugs found** (#): None

### B4 — Citation integrity (EM10.5 Bug 1 regression, through the Slack path)

**Setup**: B1 passed.
**Background**: EM10.5 Bug 1 found that unrelated retrieval candidates the model never cited were being returned as sources — asking about the weekly sales meeting also surfaced `test_b_long.txt`, an accredited-investors PDF, `slack-test.txt`, and the unrelated Project Phoenix email. The fix (the model reports `Cited: [n, n]`, validated and used to filter `sources[]`) lives in aikb's `runKnowledgeQuery.js` and is therefore **shared with Slack — but has only ever been verified live through the portal** (EM10.5, 2026-08-05).
**Action**: `@mention` the bot with a question whose answer should come from exactly one known Gmail document (the weekly sales meeting question is the strongest test, since it is the one that originally reproduced the bug).
**Expected**:
- only the genuinely cited source(s) appear;
- the previously-observed unrelated candidates do **not** appear;
- source count is plausible for the answer (not "every retrieved chunk's document");
- no internal identifiers, storage paths, signed URLs, or database IDs anywhere in the message;
- formatting is readable — `Sources:` header present, one `• ` line per source, nothing truncated mid-token.

**Evidence to capture**: the verbatim Slack message, including the complete sources block.
**Result**: ☑ Pass ☐ Fail ☐ Partial
**Notes**: Asked as a channel `@mention` in `C0B75NXAMGW`, 2026-08-30 19:50:33 UTC (`slack_event_log` event `2493e675…`, `status: delivered`, completed in ~6.7s). The sources block cited only "Weekly Sales Meeting Agenda" — none of the previously-observed unrelated candidates (`test_b_long.txt`, the accredited-investors PDF, `slack-test.txt`, the unrelated Project Phoenix email) appeared. Confirms EM10.5 Bug 1's citation-filtering fix holds through the Slack path, not just the portal (where it was originally verified 2026-08-05). Zero `email_ingestion_events` rows in the request window.
**Bugs found** (#): None

### B5 — Collection access fails closed

*Proves fail-closed behavior. A knowledge gap here is a pass, not a bug.*

**Setup**: B1 passed. A.6's full allow-list recorded so it can be restored exactly.
**Action**:
1. Confirm the Gmail document is indexed and answerable **in the portal** (portal retrieval is not collection-gated the way Slack is).
2. Remove its collection from the Slack allow-list.
3. Ask the same question in Slack.
4. Restore the allow-list to A.6's recorded state.
5. Ask again.

**Expected**: at step 3, Slack returns a clean knowledge gap ("I couldn't find that information in your organization's knowledge base.") — **not** an error, **not** an unrestricted search, and with no leaked content, title, or metadata from the excluded collection. At step 5 the original answer returns.
**Evidence to capture**: the step-3 message verbatim; confirmation the portal still answered at step 1; the restored allow-list at step 5.
**Result**: ☑ **Pass — 2026-08-24.** Run out of order, alongside the A.6/Bug 4 fix, rather than after B1 as this document's original sequencing intended — noted here as a process deviation, not a defect. Portal control (step 1) was confirmed answerable before the Slack-side removal. General was removed from the allow-list, the question returned a clean knowledge gap (no error, no leaked metadata), General was restored, and the same question then returned the correct answer.
**Notes**: because this ran before B1, it used a `portal_upload` document (same one as A.7), not a confirmed Gmail document — the Gmail-specific instance of this same fail-closed behavior is still open to verify once B1 runs. Re-run B5 with a Gmail document if a Gmail-specific confirmation is wanted, though the mechanism (`slack_collection_access` gating by collection, not by source provider) is document-agnostic and already proven here.
**Bugs found** (#): none

### B6 — Tenant isolation

**Setup**: identify content that exists only for a different client/tenant, or otherwise outside this client's allowed scope. Record what it is and how you know it is out of scope: ______________________
**Action**: from the test workspace, ask Slack a question that would only be answerable from that out-of-scope content — both as a channel `@mention` and as a DM. Ask the same question in the portal as the control.
**Expected**: Slack never retrieves it. No cross-client source metadata, titles, subjects, or senders appear. Portal and Slack isolation behavior are consistent — neither surface answers.
**Evidence to capture**: both Slack messages verbatim; the portal control result; the resolved `client_id` on the Slack request (workspace → client mapping is re-resolved per request from `team_id`, never trusted from the payload).
**Result**: ☑ **Pass — 2026-08-30, on weaker evidence than this document's own stated standard.** Not run as a fresh, controlled query from the Relativity Systems test workspace against a specific identified out-of-scope document (the originally proposed check, using Scribed.ai's `Capital-Raise-Pipeline-SOP.pdf`, was not carried out — no `slack_event_log`/`email_ingestion_events` trail exists for it). Marked Pass instead on the reporter's own firsthand account: while a different real client (Dawna) used the product, she was observed to only see outputs sourced from her own client's documents, never another tenant's.
**Notes**: this is real evidence of tenant isolation holding in practice, but it is a different, softer kind than every other scenario in this document — an incidental observation of one client's own session, not a targeted cross-tenant probe from the test client with a captured verbatim reply and a resolved `client_id`. It doesn't specifically exercise the Slack `team_id → client_id` resolution path this scenario was written to test (Dawna's observation was, per context, about the product generally, not necessarily the Slack surface specifically). If a Slack-specific regression in that resolution path is ever suspected, this result should not be treated as having ruled it out — the original controlled check (ask about `Capital-Raise-Pipeline-SOP.pdf` from the Relativity Systems workspace, confirm a clean knowledge gap and the correct resolved `client_id`) remains a quick, cheap way to get first-party evidence.
**Bugs found** (#): none

### B7 — Label removal propagates to Slack

*Verification-only. The action and the backend proof belong to EM10.5 Scenario 3, which already passed in production on 2026-08-08.*

**Setup**: one known ingested Gmail document that is currently answerable in Slack (confirm via B1 before starting). Record which: **"Customer Refund Policy"** (one of A.5's three reference documents; also used as B1/B2's refund-policy question).
**Action**:
1. Confirm the document is answerable in Slack. ☑ **Done — 2026-08-31.**
2. Remove the `Relativity/Knowledge` label in Gmail.
3. Run a sync through the existing portal workflow; confirm it completed.
4. Ask the same question in Slack.
5. Re-add the label.
6. Re-sync.
7. Ask the same question in Slack again.

**Expected**: after step 4, the document is no longer answerable in Slack (knowledge gap, or an answer that no longer cites it). After step 7, it is answerable again. Slack tracks the same indexed-state lifecycle as the portal, because both consume the same AIKB retrieval.
**Evidence to capture**: Slack messages at steps 1, 4, and 7. Per Bug 6, **do not** rely on the sync summary — capture `email_ingestion_events` outcomes for steps 3 and 6, and the document's status/chunk count if a step fails. Note that steps 5–7 exercise the deleted-document re-ingest path that produced EM10.5 Bugs 7, 8, and 9; if step 7 fails, check whether the document actually returned to `indexed` before logging it as a Slack bug.

**Step 1 evidence (2026-08-31, 7:26 AM)** — `@mention` in channel, question "what is our refund policy?":
> TL;DR: Customers can request a refund within 30 calendar days of purchase, provided they include the original order number. Refunds are processed within 5 business days after approval. Refunds are not available for digital services that have already been fully delivered.
> Source: Email — "Customer Refund Policy" from 10zinsteel@gmail.com, Aug 5, 2026
> Sources: "Customer Refund Policy" from 10zinsteel@gmail.com — via tenzin@relativitysystems.ai's mailbox

**Steps 2–3 (2026-08-31)**: label removed in Gmail; synced via portal. Confirmed directly against AIKB (per Bug 6, not the sync summary): `knowledge_documents.status` for this document went `indexed` → `deleted`, `updated_at` bumped to the sync time, chunk count `1` → `0`. No corresponding row appeared in `knowledge_ingestion_jobs` for this sync — the delete path updates the document row directly rather than logging a job; noted as a minor evidence-trail gap, not a defect (state change is independently confirmed via status + chunk count).

**Step 4 evidence (2026-08-31, 7:32 AM)** — same question, same channel:
> TL;DR: There is no documented refund policy in the provided knowledge base.
> Source: N/A
> Sources: test_b_long.txt · restricted-test.txt · "Accreited investors TO 2026 .pdf.pdf" · "QA Test — Northstar Vendor Policy" · "Weekly Sales Meeting Agenda"

Answer text is a correct gap (no citation of the removed document) — matches this step's core expectation. But see **Bug 5**: the response still rendered a `Sources:` block of unrelated retrieved-but-unused documents alongside a `Source: N/A` gap answer, because the gap-phrase heuristic in `aikb/services/runKnowledgeQuery.js` didn't recognize this phrasing.

**Bug 5 fixed and re-verified (2026-08-31)**: `isKnowledgeGapAnswer()` patched to also check for the `Source: N/A` line (deployed to production, Railway `AI_Knowledge_Base_Inngest` service, commit `777a118`). Re-asked the same question live in Slack, 7:43 AM: clean `"I couldn't find that information in your organization's knowledge base."`, no Sources block. Fix confirmed working end-to-end.

**Steps 5–6 (2026-09-02)**: `Relativity/Knowledge` label re-added in Gmail; re-synced via portal. Confirmed directly against AIKB (per Bug 6, not the sync summary): `email_ingestion_events` recorded outcome `ingested` at 05:40:28 UTC, reason "Matched allow rule 4b806d48-371e-49ed-8220-314a737f7e9a" (the same message id, `19fcfcc40e0d66c1`, that was tombstoned at step 3). `knowledge_ingestion_jobs` shows a `completed` job for this document created 05:40:28, updated 05:40:33. `knowledge_documents.status` is back to `indexed`, chunk count `0` → `1`, `last_indexed_at` 05:40:32. Minor evidence-trail note (mirror image of step 3's): this `ingested` event's own `ingested_document_id` column is null even though the document row and job both resolve cleanly to `eef87a0b-c6db-45eb-956f-1243406f159f` — not treated as a defect, same as step 3's analogous gap.

**Step 7 evidence (2026-09-02, ~05:40:50 UTC)**: `knowledge_slack_request_log` shows one `slack`-origin request at 05:40:50 (18s after the step 6 sync completed), `status: delivered`, `attempt_count: 1`, no `error_category`. Reporter (user) confirmed the answer itself was clean — the document was answerable again, matching step 1. Answer text was not captured verbatim in this pass (the request-log table stores delivery metadata only, not message content) — flagged as weaker evidence than steps 1 and 4, which do have verbatim quotes; if a verbatim re-confirmation is ever needed, re-ask the same question in the same channel.

**Result**: ☑ **Pass — 2026-09-02.** Full label-removal → tombstone → re-add → re-ingest → retrievable-again lifecycle confirmed through Slack, backed by direct AIKB/backend evidence at every step (per Bug 6, not the sync summary). Bug 5 found and fixed along the way (step 4); no further bugs found in steps 5–7.
**Notes**: steps 5–7 exercise the deleted-document re-ingest path that produced EM10.5 Bugs 7, 8, and 9 — no recurrence observed here; the document cleanly returned to `indexed` with the same `document_id` reused (no duplicate row).
**Bugs found** (#): 5 (found in step 4; none new in steps 5–7)

### B8 — Policy change propagates to Slack

*Gated on EM10.5 Scenario 4 — passed 2026-08-06 (see `EM10_5_STAGING_CHECKLIST.md`).*

**Setup**: Scenario 4's own original evidence (a single tombstoned test message) had since reverted — the org policy narrowing from that run was undone afterward, and the message re-matched the standard allow rule on 2026-08-08, leaving nothing currently policy-excluded. Rather than reuse stale evidence, a fresh deny rule was added 2026-09-02: `Deny`, Sender `tenzin@relativitysystems.ai`, no other criteria. That sender backs two currently-indexed test documents ingested into Member B's (`10zinsteel@gmail.com`) mailbox — "QA Test — Northstar Vendor Policy" and "QA Test — Project Aurora Deployment" — while leaving Customer Refund Policy / Project Phoenix Onboarding SOP / Weekly Sales Meeting Agenda (different sender, `10zinsteel@gmail.com`, into Member A's mailbox) untouched as the still-allowed control.

**Action**:
1. Confirmed "QA Test — Northstar Vendor Policy" answerable in the portal before any change (baseline).
2. Saved the deny rule above via the portal's policy builder.
3. Synced the `10zinsteel@gmail.com` connection.
4. **Blocked here — see Bug 10.** The sync produced `excluded_deny_listed` events for both target messages, but neither already-indexed document was tombstoned (`knowledge_documents.status` stayed `indexed`). Root-caused to a real bug: `getPreviouslyIngestedMessageIds` scoped its "what has this mailbox previously ingested" lookup to a single `email_connections.id`, and the `10zinsteel@gmail.com` connection had been silently replaced by a new connection row (reconnect) shortly before this sync — orphaning both documents' original `ingested` events from reconciliation. Logged as **Bug 10** in `EM10_5_STAGING_CHECKLIST.md`'s Part 2 bug log; fixed same-day (`emailSyncService.js` now resolves every connection id sharing the mailbox's durable identity — client + member + provider + address — not just the current row), full 826/826 suite green, deployed to production (Vercel `dpl_5KvmSRceGnbCR9DVMpZRpN7MeLna`, commit `b33d494`).
5. Re-synced the same connection post-fix. `email_ingestion_events` recorded fresh `tombstoned_policy_change` outcomes for both messages with `ingested_document_id` correctly populated; both documents confirmed `status: 'deleted'` in `knowledge_documents` (2026-09-03, ~04:51 UTC).
6. Asked the tombstoned content's question in Slack, then a control question against untouched content.

**Step 6 evidence (2026-09-03)** — `@mention` in channel, "Who is our approved reporting vendor for Project Aurora, and what's the retention requirement for their reports?":
> I couldn't find that information in your organization's knowledge base.

Clean gap, consistent with the portal's own re-ask (same question, same clean `Source: N/A` gap format, no stray Sources block — Bug 5's fix holding).

**Control evidence (2026-09-03)** — Slack, refund policy question (content unaffected by the deny rule): came back correct, consistent with B1/B7's established baseline for that document.

**Expected**: the portal stops finding the tombstoned content and so does Slack. Still-allowed documents remain answerable from both surfaces.
**Evidence to capture**: both Slack messages; the corresponding portal results as controls; `email_ingestion_events` tombstone outcomes (Bug 6 — the sync summary will misreport).
**Result**: ☑ **Pass — 2026-09-03, after finding and fixing Bug 10 along the way.** Both halves of the expected behavior confirmed in Slack: previously-answerable content that now fails org policy returns a clean knowledge gap; still-allowed content remains answerable, unaffected.
**Notes**: Scenario 4's original gate evidence had gone stale by the time this ran (see Setup) — B8 effectively re-validated the policy-change tombstone path end-to-end itself, on fresh evidence, rather than purely relying on Scenario 4's record. The real finding here wasn't about Slack at all — Bug 10 is a backend reconciliation gap (reconnecting a mailbox orphans its ingestion history from both label-removal and policy-change tombstoning) that would have equally affected the portal; Slack simply inherited it because it consumes the same AIKB retrieval.
**Bugs found** (#): 10 — found and fixed in step 4; none new in step 6

### B9 — Member lifecycle changes propagate to Slack

*Gated on EM10.5 Scenarios 5, 8, and 9, all still unrun. Destructive — see A.1.*

**Setup**: EM10.5's corresponding scenario completed, with its own backend record of what content was removed or retained.

| EM10.5 scenario (owns the action + backend proof) | Slack verification (owned here) | Expected in Slack | Result |
|---|---|---|---|
| S5 — Member A disabled | Ask a question Member A's ingested content answers | Still answerable — cleanup was not requested, so content stays in place | ☐ Pass ☐ Fail ☐ Partial |
| S8 — Member A disconnects with `cleanupIngestedContent: true` | Ask the same question again | No longer answerable; Member B's content still answerable | ☐ Pass ☐ Fail ☐ Partial |
| S9 — Member B disconnects with no cleanup | Ask a question Member B's content answers | Still answerable — connection revoked, content retained | ☐ Pass ☐ Fail ☐ Partial |

**Evidence to capture**: the Slack message per row; the matching EM10.5 backend record it is being checked against.
**Notes**:
**Bugs found** (#):

---

# Part C — Slack platform behavior

Slack-specific mechanics with no portal analog. Keep these separate from Part B: a failure here is a delivery/platform defect, not a knowledge-retrieval defect.

### ~~C1 — Identity linking (EL7A)~~

**N/A / Removed (EL7C, 2026-08-30).** Slack identity linking (EL7A) and Slack's live-email-lookup path (EL7B) were removed entirely before this scenario was ever run — `slack_user_links` had been empty since EL7A shipped (2026-07-31), confirming the linking flow was never actually used in practice. Live email lookup remains portal-only. See the [EL7C Implementation Record](LIVE_EMAIL_LOOKUP.md#el7c--slack-live-email-access-removal) for the removal rationale and scope.

~~**Setup**: Slack User A unlinked; portal Email panel accessible.
**Action**: generate a link code in the portal (`POST /api/integrations/slack/link/generate-code`) and DM the bot `link CODE`. Then exercise each failure mode: a reused code, an expired code, an unknown/malformed code, and a code generated under a different client.
**Expected**: the valid code links and replies *"Your Slack account is now linked to your Relativity mailbox."* Each failure mode returns its own non-leaking reply (`linkAttemptReplyText` — `not_found` and `client_mismatch` deliberately share wording so a wrong-client code is indistinguishable from an invalid one; `reused` and `expired` are distinct). Codes are single-use and hash-stored (`slack_link_codes`). No link is ever resolved from a Slack-reported email address alone.
**Evidence to capture**: each reply verbatim; the `slack_user_links` row after success; confirmation a re-link to a different member cleanly replaces the prior mapping.
**Result**: ☐ Pass ☐ Fail ☐ Partial
**Notes**:
**Bugs found** (#):~~

### C2 — Asynchronous delivery lifecycle

**Setup**: A.7 passed.
**Action**: ask a question in Slack and trace it end to end.
**Expected**, each step observable:
1. the event is accepted (HTTP 200 to Slack, promptly);
2. the request is enqueued to AIKB (`OUTCOME.ENQUEUED`, an `aikbEventId` returned);
3. AIKB generates an answer;
4. the `/deliver` callback succeeds (signed service request, `claimForDelivery` claims the row);
5. `chat.postMessage` produces **exactly one** response in Slack.

**Evidence to capture**: the `slack_event_log` row's status transitions; observed end-to-end latency (record it — this is the number a user actually feels, and no fixture measures it); confirmation of a single posted message.
**Result**: ☐ Pass ☐ Fail ☐ Partial
**Notes**:
**Bugs found** (#):

### C3 — Duplicate suppression and idempotency

**Setup**: C2 passed.
**Action**: provoke a redelivery of the same Slack event (Slack re-sends when the endpoint is slow to 200). Separately, provoke a second `/deliver` callback for the same idempotency key.
**Expected**: the redelivered event is recognized (`OUTCOME.DUPLICATE`) and produces **no second answer**. A concurrent or repeated `/deliver` callback loses the `claimForDelivery` race and is a safe no-op. A transient delivery retry does not create duplicate messages.
**Evidence to capture**: the Slack channel/DM showing exactly one answer; the `slack_event_log` row showing the duplicate/claim outcome.
**Result**: ☐ Pass ☐ Fail ☐ Partial
**Notes**:
**Bugs found** (#):

### C4 — Thread placement

**Setup**: C2 passed.
**Action**: `@mention` the bot (a) at top level in a channel, and (b) inside an existing thread. Then provoke a retry on a threaded mention.
**Expected**: the response lands in the expected thread — a threaded mention is answered in that thread, not as a new top-level message. No duplicate top-level replies are created. Retries preserve correct thread placement (`threadTs` is carried through `originMetadata` from the original event, not re-derived at delivery time).
**Evidence to capture**: screenshots or permalinks showing placement for each case, including after the retry.
**Result**: ☐ Pass ☐ Fail ☐ Partial
**Notes**:
**Bugs found** (#):

### C5 — Failure paths

*Bounded, fail-safe behavior — deliberately separated from content correctness.*

**Setup**: A.4's retry/backoff values recorded.
**Action**: provoke each failure mode reachable without code changes.

| Failure injected | Expected behavior | Result |
|---|---|---|
| AIKB unavailable / unreachable | Retries bounded by `SLACK_DELIVERY_MAX_ATTEMPTS`, then a single best-effort *"I couldn't complete that request right now. Please try again shortly."*, row marked `delivery_failed` and redacted ([ADR-007](../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md)) | ☐ Pass ☐ Fail ☐ Partial |
| Slack bot token revoked mid-flight | Finalizes as `CONNECTION_REVOKED`, no infinite retry loop, no duplicate replies | ☐ Pass ☐ Fail ☐ Partial |
| `/deliver` callback fails | Bounded retry; no duplicate message on eventual success | ☐ Pass ☐ Fail ☐ Partial |
| Tampered / absent Slack signature | Rejected 401 before reaching the handler | ☐ Pass ☐ Fail ☐ Partial |
| Timestamp older than the 5-minute replay window | Rejected 400 | ☐ Pass ☐ Fail ☐ Partial |
| Malformed event payload | Silent HTTP 200 no-op (`OUTCOME.MALFORMED`), no reply, no crash | ☐ Pass ☐ Fail ☐ Partial |
| Empty question (bare `@mention`) | *"Please include a question after mentioning me."*, no AIKB round trip | ☐ Pass ☐ Fail ☐ Partial |
| Group DM (`mpim`) | Silently ignored (`OUTCOME.MPIM_UNSUPPORTED`), HTTP 200, no reply posted | ☐ Pass ☐ Fail ☐ Partial |
| Missing collection permission | Covered by B5 — cross-reference, do not re-run here | — |

**Expected across all rows**: bounded behavior, no duplicate replies, and **no message text, answer text, question text, or token in any log** (`slackEventLogService`'s metadata-only contract).
**Evidence to capture**: per row, the observed reply (or absence), the log line, and confirmation no sensitive text was logged.
**Notes**:
**Bugs found** (#):

### C6 — Formatting limits

**Setup**: A.4's `SLACK_QUESTION_MAX_LENGTH` recorded.
**Action**: (a) ask something whose answer exceeds `MAX_ANSWER_CHARS` (3000); (b) ask something that retrieves more than `MAX_CITATIONS` (5) distinct sources; (c) send a question exceeding `SLACK_QUESTION_MAX_LENGTH`.
**Expected**: (a) the answer truncates with a trailing `…`, no broken mrkdwn, `Sources:` header intact; (b) citations cap at 5, deduplicated case-insensitively; (c) the over-length question is rejected (`too_long`) with no AIKB round trip.
**Evidence to capture**: the rendered messages; the rejection reply for (c).
**Result**: ☐ Pass ☐ Fail ☐ Partial
**Notes**:
**Bugs found** (#):

---

# ~~Part D — Live Gmail lookup (supplemental)~~

**N/A / Removed (EL7C, 2026-08-30).** Slack's live-email-lookup path (EL7B), and the identity linking (EL7A) it depended on, were removed entirely before any scenario in this Part was ever run. Live email lookup remains portal-only — there is no Slack surface left for D1–D4 to validate. [EL10](LIVE_EMAIL_LOOKUP.md#el10--staging-validation-and-security-testing) remains the authoritative validation for the live-lookup backend itself, unaffected by this removal. See the [EL7C Implementation Record](LIVE_EMAIL_LOOKUP.md#el7c--slack-live-email-access-removal) for the removal rationale and scope.

~~**Clearly separate from Parts B and C.** This is a **surface smoke check only** — confirming that the already-built live-lookup backend surfaces correctly through Slack. [EL10](LIVE_EMAIL_LOOKUP.md#el10--staging-validation-and-security-testing) remains the authoritative validation for live email lookup, and its scope is unchanged: real-account and prompt-injection security validation of the live-lookup **backend**, via direct signed `POST /api/tools/execute` / `runKnowledgeQuery` calls. Nothing here discharges EL10, and a failure here should be triaged as Slack-surface or backend before being logged against either.

**Do not let any result in this Part be used as evidence for a Part B scenario.**

### D1 — Linked DM user performs a live lookup

**Setup**: Slack User A linked (C1); Member A's Gmail connection active; a real message in Member A's mailbox that is **not** ingested (no managed label, not policy-matched). Record which: ______________________
**Action**: Slack User A DMs a live-lookup-shaped question about that message.
**Expected**: `requestingMemberId` resolves from `slack_user_links` and only Member A's mailbox is searched. Live sources render with the `(Live)` suffix. The two-call search-then-fetch bound holds — a third call attempt is rejected. A deny-listed sender's message never surfaces even when asked for directly.
**Evidence to capture**: the answer and sources; the `email_ingestion_events` audit row(s) with `tool_name`, `result_count`, `origin`, `origin_metadata`, `provider_latency_ms` populated.
**Result**: ☐ Pass ☐ Fail ☐ Partial
**Notes**:
**Bugs found** (#):

### D2 — Unlinked user gets the prompt, never a silent failure

**Setup**: Slack User B deliberately left unlinked.
**Action**: Slack User B DMs the same live-lookup-shaped question from D1.
**Expected**: no live lookup occurs; no error is shown. The reply carries the appended hint *"I can search your email for questions like this once you link your Slack account or connect email search — see the Relativity portal's Email panel."* (`FALLBACK.EMAIL_LOOKUP_SUGGESTED`). Per `formatSlackMessage`, the hint appears **even on a knowledge-gap answer** — the "never a silent failure" rule takes priority. User B sees nothing from User A's mailbox.
**Evidence to capture**: the reply verbatim; confirmation of no audit row with a `tool_name`.
**Result**: ☐ Pass ☐ Fail ☐ Partial
**Notes**:
**Bugs found** (#):

### D3 — Live lookup privacy and authorization

**Setup**: both Slack users linked to their own respective members.
**Action**: Slack User B DMs a question whose answer exists only in Member A's mailbox.
**Expected**: Member A's mailbox is never searched on User B's behalf. No subject, sender, snippet, or metadata from Member A's mailbox appears in User B's reply. Every live interaction produces an audit row naming the Slack user, workspace, member, mailbox, tool, and result count (EL7B acceptance criteria).
**Evidence to capture**: the reply; the audit rows for both users' requests, showing distinct member/mailbox attribution.
**Result**: ☐ Pass ☐ Fail ☐ Partial
**Notes**:
**Bugs found** (#):

### D4 — Channel mentions never offer live tools

**Setup**: Slack User A linked with an active mailbox — i.e. the case where tools *would* be offered over DM.
**Action**: ask D1's live-lookup-shaped question as a channel `@mention` instead of a DM.
**Expected**: no live lookup occurs and no mailbox content appears, **regardless of link status** — this is the hard MVP policy stated in `slackEventsService.js` ("channel `@mention`s never offer the tools regardless of link status"), not an incidental behavior. The answer falls back to stored knowledge, with the link/unavailable hint if applicable.
**Evidence to capture**: the reply; confirmation of no audit row with a `tool_name` for that request.
**Why this matters beyond Part D**: this scenario is what licenses Part B's use of channel `@mention`s as a contamination-free control. If it fails, Part B's evidence model needs revisiting.
**Result**: ☐ Pass ☐ Fail ☐ Partial
**Notes**:
**Bugs found** (#):~~

---

# Part E — Bug log, corrections, and readiness

## E.1 Bug log

| # | Scenario | Description | Severity | Status |
|---|----------|-------------|----------|--------|
| 1 | A.2 | **Slack reconnect would downgrade scopes and silently break DMs.** `services/slackService.js:17` sets `REQUIRED_SCOPES = ['app_mentions:read', 'chat:write']` and `:51` sends exactly that as the OAuth `scope` parameter. The live token (`oauth_connections.scopes_granted`, granted 2026-07-16) carries eleven scopes including `im:history`, `im:read`, and `im:write` — so the working install predates this constant or was granted from the app manifest. Any disconnect/reconnect through the current code path requests two scopes, and Slack issues a token with two. DM events would stop arriving, which silently disables EL7A identity linking (`link CODE` is DM-only), B3's DM leg, and all of Part D. Nothing in the code detects or warns about the downgrade. | **High** — latent; no impact until someone reconnects, total DM loss the moment they do | **Open.** Slack-surface. Workaround for this pass: do not reconnect. Fix: `REQUIRED_SCOPES` must include `im:history`, `im:read`, `im:write` to match what the feature set actually needs. |
| 2 | A.5 | **`Weekly Sales Meeting Agenda` is indexed and searchable, contradicting EM10.5's own closing record.** EM10.5 Scenario 3's final 2026-08-08 retest states the managed label was removed from this message and the Full Scan "correctly not re-imported" it. AIKB shows document `83039afe-…` at `status = indexed` with 1 chunk, `updated_at` 2026-08-08 15:20:30 — 17 minutes *after* the two documents that scan did import (15:03:04). Some event re-indexed it that EM10.5's record does not describe. Either the label was re-applied out of band, or a tombstone did not hold. | **Medium** — invalidates B7's starting state; possible unclosed edge of Bugs 4/5/7 in EM10.5 | **Open.** Backend/ingestion — belongs against EM10.5, not this document. Reconcile the actual Gmail label state before running B7. |
| 3 | A.5 | **Nine `sync_enabled` `email_connections` rows for two distinct mailboxes.** Owner `65900664-…` has 6 rows for `tenzin@relativitysystems.ai` (created 2026-07-27 through 2026-08-08); member `4c8acdee-…` has 3 for `10zinsteel@gmail.com`. All nine are `sync_enabled = true`. Two of the member's three have `managed_label_id = null`, so those connections never created or reused a managed label. EM1 added provider-partitioned partial unique indexes to `oauth_connections`, but `email_connections` appears to have no equivalent one-active-per-member-per-mailbox constraint, so each reconnect accreted a row instead of replacing one. | **Medium** — duplicate syncing today; makes B9 and EM10.5's Scenarios 5/8/9 unreliable, since disabling or disconnecting one row may leave five syncing | **Open.** Backend/ingestion — belongs against EM10.5/EM9. |
| 4 | A.6 | **Slack's collection allow-list grants a collection that holds no documents, so Slack can retrieve nothing for this client.** `slack_collection_access` has exactly one row for client `72e78cfe-…`: collection `3b15ddc1` ("Slack"), which contains 0 indexed documents. All 10 of the client's indexed documents — 3 `gmail`, 7 `portal_upload` — are in `c033f615` ("General", default), which is not allow-listed. `aikbAskClient.js` passes the allow-list through explicitly and an empty-of-content grant retrieves nothing, per [ADR-005](../decisions/ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md). **This is fail-closed behavior working correctly, not a code defect** — logged to track the configuration fix and because it fully blocks A.7 and Part B. | **High** — blocks the entire validation; in production, means the Slack bot answers nothing for this client today | **Closed — 2026-08-24.** Configuration. Fix applied: `c033f615-57ea-42a0-956a-3d9ba5875bf5` added to the allow-list via the portal's Slack panel, alongside (not replacing) `3b15ddc1`. Verified via A.7 (pass) and B5's fail-closed round-trip (pass) — see both sections above. The naming-split product question (should Slack-visible content route to its own collection rather than the client default?) remains open, tracked in E.2, and is not part of what this bug closes. |
| 5 | B7 | **A genuinely-gap answer slips past `isKnowledgeGapAnswer()`'s phrase list and gets irrelevant citations attached.** After the "Customer Refund Policy" document was tombstoned (step 4), the model's answer was "There is no documented refund policy in the provided knowledge base." — a correct gap, but phrased differently from every fixed substring `isKnowledgeGapAnswer()` checks for (`aikb/services/runKnowledgeQuery.js:58-71`: `'not documented in'`, `'not found in'`, `"couldn't find"`, `'could not find'`, `'no information in'`, `'not in the knowledge base'`, `'not available in the knowledge base'`, `'not provided in the documentation'`, `'there is no information'`). None match "no documented refund policy in the provided knowledge base", so `isGap` evaluates `false` and the response falls through to the normal-answer path, attaching `allSources` — the retrieved-but-unused chunks (`test_b_long.txt`, `restricted-test.txt`, an accredited-investors PDF, "QA Test — Northstar Vendor Policy", "Weekly Sales Meeting Agenda") — as a rendered `Sources:` block, even though the answer text itself says `Source: N/A`. `slackAnswerFormatter.js` is not at fault — it correctly omits a Sources block whenever `isKnowledgeGap` is true; it was simply never told this was a gap. | **Medium** — no data leak (all sources are within the same client's own workspace) but misleading: a user reading "no documented policy" alongside five cited-looking document names may reasonably assume one of them is relevant | **Fixed — 2026-08-31.** `isKnowledgeGapAnswer()` now also checks for a `Source: N/A` line (`SOURCE_NA_RE`) — the structured signal `RAG_SYSTEM_PROMPT` already mandates whenever the model can't answer, independent of prose phrasing. More robust than the phrase list alone, which would have missed even the system prompt's own canonical phrase ("This is not fully documented in our knowledge base." doesn't contain the fixed substring `'not documented in'` — "fully" breaks it). Regression test added in `aikb/test/runKnowledgeQuery.test.js` reproducing this exact answer text; full `runKnowledgeQuery.test.js` suite (44 tests) and full aikb suite (196 tests, excluding 3 pre-existing/unrelated live-trigger scripts requiring real env/network) pass. Re-verified live in Slack, 2026-08-31 7:43 AM — same question now returns the clean `FALLBACK.KNOWLEDGE_GAP` text ("I couldn't find that information in your organization's knowledge base.") with no Sources block. |

When logging, state whether the defect is **Slack-surface** (event handling, formatting, delivery, collection resolution) or **backend/ingestion** (retrieval, indexing, tool orchestration). A backend defect found here belongs against EM10.5 or the relevant EL milestone, not against this document.

## E.2 Documentation and product corrections

Anything [EMAIL_INGESTION.md](EMAIL_INGESTION.md), [LIVE_EMAIL_LOOKUP.md](LIVE_EMAIL_LOOKUP.md), [ADR-003](../decisions/ADR-003-SLACK-EVENTS-LIVE-IN-RELATIVITY.md), [ADR-007](../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md), or another doc claimed that this pass proved wrong — list it here, then apply the correction as a follow-up doc-only change, the same discipline EM10.5's own Part 3 follows.

**Open candidate, identified before the pass begins:**

- **Slack citations carry no "Open in Gmail" deep link.** EM10.5's Scenario 2 acceptance language is portal-specific; `slackAnswerFormatter.js` deliberately emits titles only. Whether this is a documentation gap (the criterion should be scoped to the portal) or a product gap (Slack should carry deep links) is a decision for this pass to record — **not** a reason to fail a Gmail ingestion scenario.

**Raised by Part A, 2026-08-09:**

- **Should Gmail-ingested mail land in the default "General" collection at all?** Every collection allow-listed for Slack is named "Slack"; every document, including Gmail's, lands in "General". Two collections named this way imply an intended split — Slack-visible vs. everything — that nothing in the ingestion path enforces, and `resolveDocumentCollectionId` sends a first-insert to the client's *default* collection with no notion of Slack visibility. Either the naming is vestigial and the allow-list should simply name the collections that hold real content, or ingestion should route by intended audience. Decide which; today's state gives a Slack bot that answers nothing, with no error anywhere to explain why.
- **Should a client with an allow-list that grants zero indexed documents be surfaced anywhere?** Bug 4 was invisible from every UI — the portal shows an allow-list, `slack_collection_access` shows a row, and the bot silently gaps on every question. `CONNECTOR_FRAMEWORK.md`'s Slack verification checklist would not catch it either. A "this allow-list grants no content" warning in the portal's Slack panel would have prevented it.
- **Should `REQUIRED_SCOPES` be derived from the feature set rather than hardcoded?** Bug 1 exists because the constant lists two scopes while the product needs at least five. Any future scope-dependent feature repeats the failure.

## E.3 Readiness decision — Slack as a knowledge-search surface

Answer each question explicitly. A blank is not a pass.

| Question | Answer | Evidence |
|---|---|---|
| Can Slack reliably retrieve Gmail-ingested AIKB knowledge? | | B1, B7 |
| Do the portal and Slack give materially consistent answers? | | B2 |
| Are collection restrictions fail-closed? | | A.6, B5 |
| Are tenant boundaries preserved? | | B6 |
| Are citations trustworthy (only genuinely cited sources, no internal identifiers)? | | B4 |
| Are channel and DM behaviors understood and documented? | | B3, D4 |
| Are retries and idempotency safe (no duplicate answers, correct thread placement)? | | C3, C4 |
| Are failure paths bounded and non-leaking? | | C5 |
| Any known issues blocking Slack being declared ready as a knowledge-search surface? | | E.1 |

**Decision**: ☐ Slack is ready as a knowledge-search surface ☐ Ready with named caveats ☐ Block on fixing specific issues first

**Named caveats / blocking issues**:

**Reasoning**:

**Decided by**:
**Date**:

---

## Related documents

- [EM10_5_STAGING_CHECKLIST.md](EM10_5_STAGING_CHECKLIST.md) — the Gmail ingestion half of EM10.5; owns every backend assertion this document defers to.
- [EMAIL_INGESTION.md](EMAIL_INGESTION.md) — EM1–EM13 design, including [EM10.5](EMAIL_INGESTION.md#em105--gmail-staging-validation) itself.
- [LIVE_EMAIL_LOOKUP.md](LIVE_EMAIL_LOOKUP.md) — EL-series design; [EL10](LIVE_EMAIL_LOOKUP.md#el10--staging-validation-and-security-testing) remains the authoritative live-lookup validation.
- [ADR-003](../decisions/ADR-003-SLACK-EVENTS-LIVE-IN-RELATIVITY.md), [ADR-004](../decisions/ADR-004-SIGNED-SERVICE-REQUESTS.md), [ADR-005](../decisions/ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md), [ADR-007](../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md), [ADR-010](../decisions/ADR-010-LIVE-TOOL-CALLS-ORCHESTRATED-BY-AIKB.md).
- [GOOGLE_OAUTH_VERIFICATION_CHECKLIST.md](GOOGLE_OAUTH_VERIFICATION_CHECKLIST.md) — parallel non-blocking track for Gmail app verification.
