# EM10.5 — Slack Surface Validation

**Companion to [EM10_5_STAGING_CHECKLIST.md](EM10_5_STAGING_CHECKLIST.md). Not a new milestone.** EM10.5 remains the milestone; this is a second execution record under it, covering a surface the original checklist never touched.

The division of labour is exact:

| | Question it answers | Owns |
|---|---|---|
| [EM10_5_STAGING_CHECKLIST.md](EM10_5_STAGING_CHECKLIST.md) | **Did Gmail ingestion work correctly?** | Gmail label → policy → sync → normalize → chunk → embed → index. Every backend assertion about whether a document and its chunks exist. |
| **This document** | **Can Slack correctly retrieve and present the knowledge that Gmail ingestion produced?** | Slack event → tenant/collection resolution → AIKB retrieval → answer/citation formatting → delivery back to Slack. |

This document does **not** own Gmail ingestion. Where a scenario needs indexed content to change (a label removed, a policy edited, a member disconnected), the change is made through the portal/Gmail exactly as EM10.5 already specifies, EM10.5 proves the document and chunk state actually changed, and this document only asks: **does Slack now behave accordingly?** Backend assertions are not duplicated here unless they are needed to diagnose a Slack-side failure.

**Status: Not started.**

---

## Slack is a question surface, not an admin surface

Slack has no verb for connecting a mailbox, editing organization policy, disabling a member, changing sync mode, triggering a sync, or cleaning up content. Those are portal/admin-console actions and appear in this document only as **setup steps**. The only user-initiated Slack inputs that exist are:

- `@mention` of the bot in a channel (`app_mention`),
- a direct message to the bot (`message` with `channel_type: 'im'`),
- the DM link command `link CODE` ([EL7A](LIVE_EMAIL_LOOKUP.md#el7a--slack-identity-linking), `slackEventsService.js` `LINK_COMMAND_PATTERN`).

Group DMs (`mpim`) are explicitly refused (`OUTCOME.MPIM_UNSUPPORTED`). Every other event type is a silent HTTP 200 no-op.

---

## Stored Gmail knowledge vs. live Gmail lookup — read this before running anything

These are two different features that can produce superficially similar Slack answers. Confusing them will silently invalidate results.

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

**Expected result for every pure ingestion test: the indexed Gmail document is retrieved, and no live Gmail tool call was required or made.**

---

## Known constraints to account for before running

**Slack citations carry no deep link, by design.** `services/slackAnswerFormatter.js` emits human-readable title text only — `formatCitations` pushes `title` (or `"subject" from sender`), never a URL, per the module's explicit contract that *"document IDs, chunk IDs, storage paths, signed URLs … must never reach this module's output."* EM10.5 Scenario 2's "a working 'Open in Gmail' link" **is not an applicable acceptance criterion on this surface.** Do not fail a Gmail ingestion scenario over it. If the absence is judged a product gap, log it in Part E's documentation/product follow-up list.

**EM10.5 Bug 6 is still open.** The portal's sync-run summary reports only ingest-side counts, so a sync that tombstones documents still displays `0 imported, 0 skipped, 0 failed`. **Do not use the sync summary alone as evidence that a state change did or did not happen.** Use `email_ingestion_events`, actual document/chunk state, and Slack's own retrieval behavior. This applies to B7, B8, and B9.

**`slack_collection_access` fails closed.** See A.6 — this is the single most likely cause of a false failure in this document.

---

# Part A — Prerequisites and surface readiness

Everything in Part A is account/credential/configuration setup. No code changes are in scope.

### A.1 Environment declaration (do this first, in writing)

EM10.5's own Part 0.3 required a staging Supabase project, and its Scenario 1 records that it ran against production anyway. Do not repeat that silently.

- [ ] Environment used for this pass: ______________________
- [ ] Supabase projects (Relativity / AIKB) in use: ______________________
- [ ] Slack workspace in use: ______________________

**Destructive scenarios in this document — B5 (collection allow-list removal), B7 (label removal), B8 (policy change), B9 (member disable, disconnect with cleanup) — must either run against a non-production test client/environment, or be explicitly acknowledged below as running against production data that is understood to be isolated test data.**

- [ ] Blast radius understood and recorded: ______________________
- [ ] Signed off by: ______________________

### A.2 Slack app

- [ ] A Slack app exists, installable to the test workspace, with an OAuth redirect URI matching `SLACK_REDIRECT_URI` byte-for-byte (Slack does exact-match, not prefix-match).
- [ ] Bot token scopes cover at minimum `app_mentions:read`, `im:history`, `im:read`, `im:write`, `chat:write` — verify against `services/slackIntegrationService.js`'s actual requested scope string rather than trusting this list.
- [ ] Event Subscriptions enabled; Request URL points at the deployed `POST /api/integrations/slack/events` and Slack's `url_verification` challenge passes (`handleUrlVerification`).
- [ ] Subscribed bot events include `app_mention` and `message.im`.
- [ ] Signing secret recorded and matching `SLACK_SIGNING_SECRET`.

### A.3 Test workspace and users

- [ ] A real Slack workspace dedicated to this test.
- [ ] Two real Slack users — **Slack User A** and **Slack User B** — corresponding to EM10.5's Member A and Member B.
- [ ] At least one public channel the bot is a member of, plus DM access for both users.

### A.4 Environment configuration

- [ ] `Relativity/.env`: `SLACK_CLIENT_ID`, `SLACK_CLIENT_SECRET`, `SLACK_SIGNING_SECRET`, `SLACK_REDIRECT_URI`, `SLACK_TOKEN_ENCRYPTION_KEY` all set. *(All five verified non-empty 2026-08-09; confirm they point at the intended app, not a stale one.)*
- [ ] `SLACK_QUESTION_MAX_LENGTH` value recorded — C6 needs it: ______
- [ ] `SLACK_DELIVERY_MAX_ATTEMPTS` / `SLACK_DELIVERY_BACKOFF_MS` recorded — C5 asserts against them ([ADR-007](../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md); default 3 attempts): ______
- [ ] `SERVICE_REQUEST_SIGNING_SECRET` matches between `Relativity` and `aikb` — the `/deliver` callback is a signed service request and fails without it ([ADR-004](../decisions/ADR-004-SIGNED-SERVICE-REQUESTS.md)).
- [ ] Both `node server.js` processes (Relativity, aikb) start cleanly.

### A.5 Test client, members, and known Gmail content

- [ ] The same test client and Member A / Member B rows from EM10.5 are reused — Slack live lookup depends on real `email_connections`, and reuse avoids provisioning a second Gmail OAuth Client (the same reasoning EL10 gives).
- [ ] The Slack workspace is connected to this client via the portal's Slack panel (`GET /api/integrations/slack/start`, owner/admin only); `GET /status` reports an active connection.
- [ ] **EM10.5's known-good Gmail documents are confirmed indexed and answerable in the portal before Slack is tested at all.** These are the fixed reference set for Part B:

| Document | Reference question | Known-good answer (per EM10.5) | Indexed? |
|---|---|---|---|
| `Project Phoenix Onboarding SOP` | What are the Project Phoenix onboarding steps? | (onboarding steps) | ☐ |
| `Weekly Sales Meeting Agenda` | When is the weekly sales meeting? | Thursday 9:00 AM | ☐ |
| `Customer Refund Policy` | What is the refund policy? | 30 days | ☐ |

- [ ] Current Gmail label state for each of the three, recorded: ______________________
  *(EM10.5's final 2026-08-08 retest left the managed label on Project Phoenix and Customer Refund Policy, and removed from Weekly Sales Meeting Agenda. B7 depends on knowing the actual starting state.)*

### A.6 Slack collection access — the fail-closed prerequisite

`slack_collection_access` gates which AIKB `knowledge_collections` a Slack question may search, client-wide (`services/slackCollectionAccessService.js`). The contract is explicit in `services/aikbAskClient.js`:

> `allowedCollectionIds` — *"Always an explicit array (possibly empty — empty means "search nothing", not "search everything"), never omitted, so a caller can never accidentally fall back to an unrestricted search."*

**`allowedCollectionIds = []` means Slack searches nothing.** If the collection holding the Gmail documents is not allow-listed, every scenario in Part B returns a knowledge gap and will look like an ingestion or RAG failure when it is purely a configuration state. This is [ADR-005](../decisions/ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md) working as designed.

- [ ] AIKB collection id holding the Gmail test documents identified: ______________________
- [ ] That id is present in `slack_collection_access` for this client (portal `PUT /api/integrations/slack/collections`, owner/admin only).
- [ ] The full current allow-list is recorded here, so B5 can restore it exactly: ______________________

### A.7 Baseline sanity check — **gate: do not start Part B until this passes**

**Setup**: A.1–A.6 complete.
**Action**: in the test channel, `@mention` the bot with a question about a **known indexed document that is not Gmail-sourced** (any portal-uploaded document already in an allow-listed collection).
**Expected**: Slack returns the known correct answer, with a `Sources:` block naming that document.
**Evidence to capture**: the answer text, the sources block, and the round-trip latency.
**Why this gate exists**: it proves the entire Slack → AIKB → retrieval → delivery path is live and the allow-list is non-empty, *before* any Gmail-specific result becomes ambiguous. A knowledge gap in B1 is unattributable without it.
**Result**: ☐ Pass ☐ Fail ☐ Partial
**Notes**:
**Bugs found** (#):

---

# Part B — Stored Gmail knowledge through Slack

**Run every scenario in this Part as a channel `@mention` unless the scenario says otherwise** — see "The structural control" above. Live email tools are structurally unavailable on that path, so any correct answer is proof of indexed AIKB retrieval.

### B1 — A known ingested Gmail document is retrievable in Slack

**Setup**: A.5's three documents indexed; A.6 allow-list correct; A.7 passed.
**Action**: `@mention` the bot in the test channel with each of A.5's three reference questions, one at a time.
**Expected**: each returns the known-good answer from the table in A.5, followed by a `Sources:` block naming the correct Gmail-backed document (rendered as its `title`, or as `"Subject" from Sender`).
**Evidence to capture**: answer text and sources block per question; confirmation of **no** `email_ingestion_events` row with a non-null `tool_name` in the request window; absence of any `(Live)` suffix.
**Result**: ☐ Pass ☐ Fail ☐ Partial
**Notes**:
**Bugs found** (#):

### B2 — Portal vs. Slack parity

*The central scenario of this document: proving both surfaces consume the same indexed knowledge correctly.*

**Setup**: B1 passed.
**Action**: for each of A.5's three reference questions, ask the **exact same question text** in the portal chat and in Slack, and compare.

| Question | Portal answer | Slack answer | Same source doc? | Unrelated citations? | Consistent? |
|---|---|---|---|---|---|
| Project Phoenix onboarding steps | | | ☐ | ☐ | ☐ |
| Weekly sales meeting time | | | ☐ | ☐ | ☐ |
| Refund policy | | | ☐ | ☐ | ☐ |

**Expected**:
- materially consistent answers (wording may differ; facts, and any figure such as "Thursday 9:00 AM" or "30 days", must not);
- the **same underlying Gmail-backed knowledge document** cited on both surfaces;
- no unrelated citations on either surface;
- **no knowledge gap in Slack for anything the portal can retrieve** — if Slack gaps where the portal answers, suspect A.6's allow-list first, then log it;
- collection restrictions behave as expected on both.

**Evidence to capture**: the completed table; both surfaces' full source lists side by side; for any divergence, the allow-list state at the time.
**Result**: ☐ Pass ☐ Fail ☐ Partial
**Notes**:
**Bugs found** (#):

### B3 — Channel `@mention` vs. DM parity

**Setup**: B1 passed. Slack User A linked (run C1 first if not).
**Action**: ask each of A.5's three reference questions **again as a DM** to the bot. Compare to B1's channel results.
**Expected**: both surfaces retrieve the same stored AIKB knowledge, subject to the same collection access. DM answers match the channel answers on facts and cited document.
**This is the one Part B scenario where live-lookup contamination is possible** — a DM from a linked user with an active mailbox may have tools offered. Apply the full evidence rules.
**Evidence to capture**: per DM question — the answer, the sources block, whether any source carried `(Live)`, and whether any `email_ingestion_events` row with a non-null `tool_name` was written. For a stored-knowledge question the expected answer is **no tool call**; if a tool *was* called and the answer still matched, record that explicitly rather than passing silently — it means the stored path was not exclusively exercised.
**Result**: ☐ Pass ☐ Fail ☐ Partial
**Notes**:
**Bugs found** (#):

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
**Result**: ☐ Pass ☐ Fail ☐ Partial
**Notes**:
**Bugs found** (#):

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
**Result**: ☐ Pass ☐ Fail ☐ Partial
**Notes**:
**Bugs found** (#):

### B6 — Tenant isolation

**Setup**: identify content that exists only for a different client/tenant, or otherwise outside this client's allowed scope. Record what it is and how you know it is out of scope: ______________________
**Action**: from the test workspace, ask Slack a question that would only be answerable from that out-of-scope content — both as a channel `@mention` and as a DM. Ask the same question in the portal as the control.
**Expected**: Slack never retrieves it. No cross-client source metadata, titles, subjects, or senders appear. Portal and Slack isolation behavior are consistent — neither surface answers.
**Evidence to capture**: both Slack messages verbatim; the portal control result; the resolved `client_id` on the Slack request (workspace → client mapping is re-resolved per request from `team_id`, never trusted from the payload).
**Result**: ☐ Pass ☐ Fail ☐ Partial
**Notes**:
**Bugs found** (#):

### B7 — Label removal propagates to Slack

*Verification-only. The action and the backend proof belong to EM10.5 Scenario 3, which already passed in production on 2026-08-08.*

**Setup**: one known ingested Gmail document that is currently answerable in Slack (confirm via B1 before starting). Record which: ______________________
**Action**:
1. Confirm the document is answerable in Slack.
2. Remove the `Relativity/Knowledge` label in Gmail.
3. Run a sync through the existing portal workflow; confirm it completed.
4. Ask the same question in Slack.
5. Re-add the label.
6. Re-sync.
7. Ask the same question in Slack again.

**Expected**: after step 4, the document is no longer answerable in Slack (knowledge gap, or an answer that no longer cites it). After step 7, it is answerable again. Slack tracks the same indexed-state lifecycle as the portal, because both consume the same AIKB retrieval.
**Evidence to capture**: Slack messages at steps 1, 4, and 7. Per Bug 6, **do not** rely on the sync summary — capture `email_ingestion_events` outcomes for steps 3 and 6, and the document's status/chunk count if a step fails. Note that steps 5–7 exercise the deleted-document re-ingest path that produced EM10.5 Bugs 7, 8, and 9; if step 7 fails, check whether the document actually returned to `indexed` before logging it as a Slack bug.
**Result**: ☐ Pass ☐ Fail ☐ Partial
**Notes**:
**Bugs found** (#):

### B8 — Policy change propagates to Slack

*Gated on EM10.5 Scenario 4, which is still unrun. Do not run this before it.*

**Setup**: EM10.5 Scenario 4 completed, with its own record of which documents were tombstoned and which remained.
**Action**: ask Slack a question that the tombstoned content previously answered, and a question that still-allowed content answers.
**Expected**: the portal stops finding the tombstoned content and so does Slack. Still-allowed documents remain answerable from both surfaces.
**Evidence to capture**: both Slack messages; the corresponding portal results as controls; `email_ingestion_events` tombstone outcomes from EM10.5 Scenario 4's own record (Bug 6 — the sync summary will misreport).
**Result**: ☐ Pass ☐ Fail ☐ Partial
**Notes**:
**Bugs found** (#):

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

### C1 — Identity linking (EL7A)

**Setup**: Slack User A unlinked; portal Email panel accessible.
**Action**: generate a link code in the portal (`POST /api/integrations/slack/link/generate-code`) and DM the bot `link CODE`. Then exercise each failure mode: a reused code, an expired code, an unknown/malformed code, and a code generated under a different client.
**Expected**: the valid code links and replies *"Your Slack account is now linked to your Relativity mailbox."* Each failure mode returns its own non-leaking reply (`linkAttemptReplyText` — `not_found` and `client_mismatch` deliberately share wording so a wrong-client code is indistinguishable from an invalid one; `reused` and `expired` are distinct). Codes are single-use and hash-stored (`slack_link_codes`). No link is ever resolved from a Slack-reported email address alone.
**Evidence to capture**: each reply verbatim; the `slack_user_links` row after success; confirmation a re-link to a different member cleanly replaces the prior mapping.
**Result**: ☐ Pass ☐ Fail ☐ Partial
**Notes**:
**Bugs found** (#):

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

# Part D — Live Gmail lookup (supplemental)

**Clearly separate from Parts B and C.** This is a **surface smoke check only** — confirming that the already-built live-lookup backend surfaces correctly through Slack. [EL10](LIVE_EMAIL_LOOKUP.md#el10--staging-validation-and-security-testing) remains the authoritative validation for live email lookup, and its scope is unchanged: real-account and prompt-injection security validation of the live-lookup **backend**, via direct signed `POST /api/tools/execute` / `runKnowledgeQuery` calls. Nothing here discharges EL10, and a failure here should be triaged as Slack-surface or backend before being logged against either.

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
**Bugs found** (#):

---

# Part E — Bug log, corrections, and readiness

## E.1 Bug log

| # | Scenario | Description | Severity | Status |
|---|----------|-------------|----------|--------|
|   |          |             |          |        |

When logging, state whether the defect is **Slack-surface** (event handling, formatting, delivery, collection resolution) or **backend/ingestion** (retrieval, indexing, tool orchestration). A backend defect found here belongs against EM10.5 or the relevant EL milestone, not against this document.

## E.2 Documentation and product corrections

Anything [EMAIL_INGESTION.md](EMAIL_INGESTION.md), [LIVE_EMAIL_LOOKUP.md](LIVE_EMAIL_LOOKUP.md), [ADR-003](../decisions/ADR-003-SLACK-EVENTS-LIVE-IN-RELATIVITY.md), [ADR-007](../decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md), or another doc claimed that this pass proved wrong — list it here, then apply the correction as a follow-up doc-only change, the same discipline EM10.5's own Part 3 follows.

**Open candidate, identified before the pass begins:**

- **Slack citations carry no "Open in Gmail" deep link.** EM10.5's Scenario 2 acceptance language is portal-specific; `slackAnswerFormatter.js` deliberately emits titles only. Whether this is a documentation gap (the criterion should be scoped to the portal) or a product gap (Slack should carry deep links) is a decision for this pass to record — **not** a reason to fail a Gmail ingestion scenario.

-

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
