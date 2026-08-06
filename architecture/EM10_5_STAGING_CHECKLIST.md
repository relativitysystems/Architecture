# EM10.5 — Gmail Staging Validation Checklist

Working document for [EMAIL_INGESTION.md](EMAIL_INGESTION.md) §31's EM10.5 milestone. This is a **fill-in-the-blanks execution record**, not a design document — the design/rationale for why this milestone exists lives in EMAIL_INGESTION.md; this file exists to record what actually happened when someone ran it against real accounts.

**Status: Not started.** No prerequisite infrastructure exists yet (verified 2026-07-25 — `GMAIL_CLIENT_ID`/`GMAIL_CLIENT_SECRET`/`GMAIL_REDIRECT_URI` are all unset in `Relativity/.env`, no Google Cloud OAuth Client is configured, no staging deployment URL exists beyond `http://localhost:3000`). Both `Relativity` and `aikb` boot cleanly with no crash on missing Gmail config (verified 2026-07-25 — `node server.js` starts successfully in both repos with Gmail env vars empty; `GET /api/integrations/email/gmail/start` correctly returns `401` under `clientAuth`, not a crash), so there is no code-side blocker — only credentials and test accounts are missing.

---

## Part 0 — Prerequisites (must all be checked before Part 1 begins)

None of this requires code changes — EM10.5 is explicitly a "no code changes" milestone (§31). Everything below is account/credential setup.

### 0.1 Google Cloud OAuth Client

- [ ] A Google Cloud project exists (new or reused) with the **Gmail API** enabled (APIs & Services → Library → "Gmail API" → Enable).
- [ ] An OAuth consent screen is configured in **Testing** mode (not required to be verified/published for this milestone — §26 of EMAIL_INGESTION.md is explicit that dev/test-allowlisted use is sufficient here; full public verification is a separate, later, non-engineering-timeline concern).
- [ ] The two Gmail test accounts (0.2 below) are added to the consent screen's **Test users** list — without this, Google rejects the OAuth flow for any account other than the project owner's.
- [ ] An OAuth Client ID is created (type: **Web application**).
- [ ] Authorized redirect URI is added, matching `GMAIL_REDIRECT_URI` exactly (byte-for-byte — Google does exact-match, not prefix-match): for local testing, `http://localhost:3000/api/integrations/email/gmail/callback`.
- [ ] Scopes requested match `Relativity/services/gmailService.js`'s actual request exactly: `gmail.readonly`, `gmail.labels`, `openid`, `email`, `profile` — verify against the live code before assuming this list is still current.
- [ ] Client ID and Client Secret are recorded somewhere safe (password manager, not committed to any repo).

### 0.2 Test Gmail mailboxes

- [ ] Two real Gmail accounts exist, dedicated to this test (not personal/production mailboxes) — Member A and Member B.
- [ ] Both accounts are added as **Test users** on the OAuth consent screen (0.1).
- [ ] Both accounts have a small amount of realistic-looking test mail already present, covering at least: something that should match an intended org policy allow rule, something that should not, and something suitable for a deny-rule test (e.g. a payroll-looking subject line).

### 0.3 Environment configuration

- [ ] `Relativity/.env`: `GMAIL_CLIENT_ID`, `GMAIL_CLIENT_SECRET`, `GMAIL_REDIRECT_URI` set from 0.1.
- [ ] `GLOBAL_SUPABASE_URL`/`GLOBAL_SUPABASE_SERVICE_ROLE_KEY` (or equivalent) point at a **staging or dedicated test Supabase project, not production** — confirm this explicitly before starting; EM10.5 will create real, if small, real data (client, members, connections, ingested content) and this file's own scenarios are destructive in places (offboarding, disconnect-with-cleanup).
- [ ] `aikb/.env`: equivalent AIKB-side Supabase/Storage config, same staging-not-production caveat.
- [ ] `SERVICE_REQUEST_SIGNING_SECRET` matches between `Relativity` and `aikb` (required for the two repos' signed-envelope calls to verify each other — §Service Contracts).
- [ ] `EMAIL_SYNC_TICK_CRON_SCHEDULE` (aikb) / `EMAIL_SYNC_TICK_INTERVAL_MS` (Relativity) — confirm these are set to something short enough to observe within a real testing session (the documented default is 20 minutes; consider temporarily lowering it for this staging pass only, then reverting — do not ship a lowered production default as a side effect of this milestone).
- [ ] Both `node server.js` processes (Relativity, aikb) start cleanly with the above set — re-run the dry-run boot check from this milestone's kickoff before proceeding to Part 1.

### 0.4 Test client/members in the app itself

- [ ] A test client exists in the app (not a real customer) with Member A and Member B as active, non-`viewer` members.
- [ ] An owner/admin session is available for the org-policy and offboarding steps.

---

## Part 1 — Scenario execution

Run in order — later scenarios depend on earlier ones' state (e.g. Scenario 3 needs Scenario 2's ingested content to exist). For each: record **Result** (Pass / Fail / Partial), **What actually happened**, and **Bugs found** (cross-reference into Part 2's bug log by number).

### Scenario 1 — Connect two members
**Action**: Member A and Member B each connect their own real Gmail mailbox via the portal's Connect flow.
**Expected**: two independent, active connections; each member's `GET /connections` shows only their own; an owner/admin's `?all=true` shows both; the managed "Relativity/Knowledge" label is visible in both real Gmail accounts after connecting.
**Result**: ☑ Pass ☐ Fail ☐ Partial
**What actually happened**: Member A and Member B each connected separate Gmail test accounts through the production portal (note: run against production, not a dedicated staging Supabase project — see Part 0.3 caveat). Both connections completed successfully. The managed Relativity/Knowledge label was created in both Gmail accounts. Clicking "Go to email" opened Gmail at the label:relativity-knowledge view for the correct connected mailbox. Each member could see only their own connection, and the owner/admin view showed both active connections.
**Bugs found** (#): None

### Scenario 2 — Import labeled mail
**Action**: an owner/admin configures an org allow rule; both members label a few real messages matching it, then run historical import.
**Expected**: only labeled-and-approved messages ingest, attributed to the correct member (`contributing_member_id`); each becomes a citable, retrievable chat answer showing subject/sender/date and a working "Open in Gmail" link that opens the real message.
**Result**: ☑ Pass
**What actually happened**: an owner/admin configured an Allow rule for the managed Relativity/Knowledge Gmail label. Member A labeled three test emails (`Project Phoenix Onboarding SOP`, `Weekly Sales Meeting Agenda`, `Customer Refund Policy`) and ran a full historical scan (completed ~2026-08-04 19:42 UTC: 3 imported, 0 skipped, 0 failed). All three emails became retrievable in chat and produced accurate answers (onboarding steps, Thursday 9:00 AM meeting time, 30-day refund policy). The relevant Gmail email appeared as the primary source and a working "Open in Gmail" link was generated — the scenario's literal acceptance criteria (import correctness, attribution, citability, working deep link) were met. The chat UI also initially displayed a separate "Sources" bubble containing unrelated retrieval candidates (e.g. `test_b_long.txt`, `Accreited investors TO 2026.pdf.pdf`, `slack-test.txt`, the other unlabeled Gmail email), and the same cited email showed two different dates depending on whether it was read from the answer's inline `Source:` line or the structured sources list. Root-caused and fixed same-day (see Bugs 1-3 below); fix is unit-tested in both repos (aikb: 41/41 `runKnowledgeQuery.test.js`, 18/18 `openaiService.test.js`; Relativity: 29/29 `portalCitations.test.js`, 840/840 full suite) and **confirmed 2026-08-05 by re-running this scenario's live chat queries against production**: source filtering is correct (only cited documents shown), dates agree between the inline and structured citations, and the sources render inside the single answer bubble rather than a second card. Three additional client-portal UI improvements shipped alongside this retest, all covered by their own unit tests and the full 857/857 Relativity suite: the manual "Email search" mode selector (`Automatic`/`Company knowledge only`/`Live email` dropdown) was removed from the Knowledge Base chat UI, with every query now silently defaulting to `automatic` (the value `routes/api.js` already fell back to); and the static "Searching connected email…" text was removed from the loading bubble, leaving only the animated three-dot indicator. See `Relativity` commits `a6cbdd4`, `c557178`, and `LIVE_EMAIL_LOOKUP.md` §2.2's confirmed-2026-08-05 update note for detail.
**Bugs found** (#): 1, 2, 3 — all closed, live-verified

### Scenario 3 — Remove a label
**Action**: Member A removes the label from a previously-ingested message (Scenario 2), then syncs again.
**Expected**: that message's content is tombstoned (no longer retrievable in chat) on the next sync; Member B's content is untouched.
**Result**: ☐ Pass ☑ Fail ☐ Partial
**What actually happened**: Member A removed the Relativity/Knowledge label only from the Weekly Sales Meeting Agenda email and ran an incremental Quick Check. The intended email became non-retrievable, but the Customer Refund Policy and Project Phoenix Onboarding SOP emails were also tombstoned. Production `email_ingestion_events` confirmed that all three messages received `tombstoned_label_removed` outcomes, although only one had lost the managed label. The incremental Gmail history path treated unrelated `labelsRemoved` events, including likely UNREAD removals caused by opening messages, as managed-label removals. The sync summary misleadingly reported 0 imported, 0 skipped, and 0 failed despite three document deletions.

After the Gmail label-removal fix, a full scan reported two emails imported, but both remained non-searchable. Inngest Cloud traces showed both ingest runs completed with `skipped: true`, `reason: content hash unchanged`, and zero chunks/pages. Earlier delete runs had removed those same documents and their chunks. The ingest deduplication path trusted the unchanged content hash without verifying that the document was still active and indexed.

After Bug 7's fix was implemented (not yet deployed), a manual production Full Scan was run to retest it. That scan reported **0 imported, 2 failed** — worse than the pre-fix result, and for a different reason. No AIKB HTTP requests, ingestion jobs, or Inngest runs were created for either message: Relativity `email_ingestion_events` recorded `AIKB storage upload failed: The resource already exists` for both. The collision happened at Relativity's storage-write step, before the request ever reached AIKB, so it did not exercise Bug 7's reindex fix at all — that fix remains unverified live. Root cause traced to a distinct bug, logged as Bug 8 below.
**Bugs found** (#): 4, 5, 6, 7, 8 — see Part 2; 4 and 5 fixed and unit-tested same day (see their Status column), 6 (observability) not yet addressed, 7 and 8 fixed and unit-tested, not yet deployed/manually retested

### Scenario 4 — Change policy
**Action**: an owner/admin edits organization policy so a previously-approved category of mail no longer matches, then a sync runs for each connection.
**Expected**: previously-ingested, now-excluded content is tombstoned for every affected connection, not just whichever one happened to sync first.
**Result**: ☐ Pass ☐ Fail ☐ Partial
**What actually happened**:
**Bugs found** (#):

### Scenario 5 — Disable a member
**Action**: an owner/admin disables Member A.
**Expected**: Member A's connection stops syncing immediately — a manual sync attempt is rejected, and if in automatic mode, the next real tick excludes it; Member A's already-ingested content remains in place (cleanup wasn't requested).
**Result**: ☐ Pass ☐ Fail ☐ Partial
**What actually happened**:
**Bugs found** (#):

### Scenario 6 — Reconnect
**Action**: re-enable Member A, then have them go through OAuth again (or disconnect/reconnect).
**Expected**: a new, working, active connection; no leftover state from the prior connection blocks the new one.
**Result**: ☐ Pass ☐ Fail ☐ Partial
**What actually happened**:
**Bugs found** (#):

### Scenario 7 — Automatic sync
**Action**: Member B switches to `automatic` mode (org-wide toggle on) and sends themselves new policy-matching mail with no label.
**Expected**: the mail appears as searchable content within one real tick interval, with no manual "Sync now" click.
**Result**: ☐ Pass ☐ Fail ☐ Partial
**What actually happened**:
**Bugs found** (#):

### Scenario 8 — Disconnect with cleanup
**Action**: Member A disconnects with `cleanupIngestedContent: true`.
**Expected**: every document `contributing_member_id`-attributed to Member A is removed; Member B's content is entirely unaffected.
**Result**: ☐ Pass ☐ Fail ☐ Partial
**What actually happened**:
**Bugs found** (#):

### Scenario 9 — Disconnect without cleanup
**Action**: Member B disconnects with no cleanup flag.
**Expected**: Member B's connection is revoked and stops syncing, but their previously-ingested content remains searchable.
**Result**: ☐ Pass ☐ Fail ☐ Partial
**What actually happened**:
**Bugs found** (#):

---

## Part 2 — Bug log

| # | Scenario | Description | Severity | Status |
|---|---|---|---|---|
| 1 | Scenario 2 | Unrelated retrieval candidates (documents the model never actually cited) displayed as answer sources in the portal's Sources box — e.g. asking about the weekly sales meeting also surfaced `test_b_long.txt`, an accredited-investors PDF, `slack-test.txt`, and the unrelated Project Phoenix email. Root cause: `runKnowledgeQuery.js` returned every retrieved chunk's document as a "source," regardless of whether the model's answer relied on it. Fix: the model now reports which numbered context items it actually used (`Cited: [n, n]`), validated against the retrieved set and used to filter `sources[]`. | Medium | Fix implemented, unit-tested (aikb `runKnowledgeQuery.test.js` — new filtering/fallback tests), **verified live in production, 2026-08-05** |
| 2 | Scenario 2 | Sources rendered in a visually disconnected second bubble/card ("SOURCES") below the answer bubble, duplicating the model's own inline "Source:" citation line already present in the answer text. Root cause: `portal.js` appended the sources box as a DOM sibling of the answer bubble, and `portal.css` gave it the same background/border-radius as the bubble itself — visually indistinguishable from a second message. Fix: sources box is now nested inside the answer bubble (CSS divider, not a second card); the model's redundant inline "Source:" line is stripped from the displayed text whenever the structured box will show. | Medium | Fix implemented, unit-tested (Relativity `portalCitations.test.js` — new `stripInlineSourceLine` tests), **verified live in production, 2026-08-05** |
| 3 | Scenario 2 | The same cited email showed two different dates: the answer's inline `Source:` line (server-formatted) vs. the structured sources list (browser-formatted). Root cause: `formatCitationDate` was duplicated in `aikb/services/openaiService.js` (server) and `Relativity/public/portal/portalCitations.js` (browser), both calling `toLocaleDateString` without pinning a timezone — each defaulted to its own runtime's local timezone, disagreeing near a timezone boundary. Fix: both now pass `timeZone: 'UTC'` explicitly. | Minor | Fix implemented, unit-tested (new UTC boundary-case tests in both repos' citation test files), **verified live in production, 2026-08-05** |
| 4 | Scenario 3 | Removing the managed Relativity/Knowledge label from ONE message (Weekly Sales Meeting Agenda) tombstoned all three previously-ingested messages during an incremental "Quick check" sync. Root cause: `gmailService.js`'s `listHistory` discarded Gmail's `labelIds` field on `labelsRemoved` history entries, and `emailSyncService.js`'s `runIncrementalPage` treated ANY `labelRemoved` event as "the managed label was removed," with no re-verification. Gmail's labelId-scoped history feed still surfaced unrelated `labelsRemoved` events (most likely `UNREAD`, removed when Customer Refund Policy and Project Phoenix were opened in Gmail during Scenario 2's own "Open in Gmail" link verification) for messages that also carried the managed label, and those got misclassified identically. Fix: `listHistory` now preserves `labelIds` on each `labelRemoved` change; `runIncrementalPage` only treats it as a real removal when `labelIds` actually names the managed label. | High | Fix implemented, unit-tested (`gmailService.test.js`, `emailSyncService.test.js` — new regression tests reproducing this exact 3-messages-from-1-label-removal scenario), **not yet manually retested live** |
| 5 | Scenario 3 | The same message (Weekly Sales Meeting Agenda) was tombstoned twice in the same sync run: once via the label-removal diff (`tombstoned_label_removed`) and again 1.65s later via policy-change reconciliation (`tombstoned_policy_change`) — a redundant delete call against an already-deleted document. Root cause: `syncConnection`'s `excludeMessageIds` guard against double-tombstoning only included `reconcileRemovedLabelsFullList`'s output (historical-only), never `runIncrementalPage`'s own per-page `reconciled` tombstones — so an incremental run's label-removal pass and its policy-change pass could each independently tombstone the same message. Fix: the exclusion set now also includes `pageOutcome.reconciled` (a no-op for historical runs, where it's always empty). | Medium | Fix implemented, unit-tested (`emailSyncService.test.js` — new double-tombstone regression test), **not yet manually retested live** |
| 6 | Scenario 3 | The portal's sync-run summary (and `email_sync_runs.messages_*` columns) only track ingest-side counts, never reconciliation/tombstone activity — the Quick check that tombstoned 3 documents displayed "0 imported, 0 skipped, 0 failed," giving no visible indication anything happened. Actual evidence only existed in `email_ingestion_events`, which isn't surfaced anywhere in the portal UI. | Low (observability gap, not a data-correctness bug) | Not yet fixed — out of scope for Bugs 4/5's narrow fix; flagged for a follow-up |
| 7 | Scenario 3 | Deleted document re-ingestion with unchanged content is skipped because stale content-hash state survives deletion or is trusted without verifying chunks/index state. After the Gmail label-removal fix (Bug 4) was live, a full scan re-imported two previously-deleted emails; both Inngest `knowledge-document-ingest` runs completed with `skipped: true, reason: "content hash unchanged", chunkCount: 0` because `knowledge_documents.content_hash` survived the earlier soft-delete and the dedup check never verified the document was still `status: 'indexed'` or that any chunks existed. Both emails were reported as imported by the Gmail sync UI (which counts successful event submission to AIKB, not successful indexing) but were never searchable. Root cause and fix: `aikb/inngest/functions.js`'s unchanged-hash skip now requires `existing.status === 'indexed'` AND a real chunk count > 0 (`services/ingestDedup.js#canSkipUnchangedHash`, backed by a new `supabaseService.getChunkCountForDocument`), and `markDocumentDeleted` now also clears `content_hash` as defense in depth. | High | Fix implemented, unit-tested (aikb `test/ingestDedup.test.js`, full suite 187/187), **Open — not yet deployed to production, and the two affected emails have not yet been manually re-scanned/retested** |
| 8 | Scenario 3 | Distinct root cause discovered while retesting Bug 7 in production: `Relativity/services/aikbService.js`'s `uploadToStorage` writes to a deterministic path (`uploads/{clientId}/{sourceFileId}`) with `upsert: false`, and Relativity always uploads to that path *before* AIKB gets a chance to run its content-hash/status dedup — the two are not ordered so that dedup can prevent the write. This is a safe assumption for portal upload, ZIP import, and Google Drive import, whose `sourceFileId` is a fresh `crypto.randomUUID()` per call (routes/api.js), so the path can never legitimately pre-exist. Gmail ingestion (EM6) reuses the same function with `sourceFileId` = the stable Gmail message id, which is legitimately re-submitted on every re-scan — historical Full Scan revisits every currently-matching message unconditionally, with no local "already ingested" pre-filter (`emailSyncService.js`'s `runHistoricalPage`). Any repeat Full Scan of already-ingested mail, or any delete-then-reimport cycle (including the exact orphan Bug 7 left behind), therefore fails at the storage write with `AIKB storage upload failed: The resource already exists` — before AIKB's request is ever made, so Bug 7's reindex fix never even gets exercised. Confirmed in production: the post-Bug-7-fix Full Scan reported 0 imported/2 failed with no AIKB HTTP requests, ingestion jobs, or Inngest runs, and `email_ingestion_events` recorded exactly this message. Fix: `uploadToStorage`/`uploadAndIngest` now take the already-threaded `sourceProvider` and pass `upsert: allowsStorageOverwrite(sourceProvider)` — a small, explicit, provider-keyed allowlist (`STABLE_SOURCE_ID_PROVIDERS = new Set(['gmail'])`) — direct atomic upsert, not delete-then-upload. `portal_upload`/`google_drive`/`dropbox` keep `upsert: false` unchanged; `microsoft` is deliberately excluded (no Relativity code path ingests it yet). | High | Fix implemented, unit-tested (`Relativity/test/aikbService.test.js`, new; full Relativity suite 870/870), **Open — not yet deployed to production, and the two affected emails have not yet been manually re-scanned/retested** |

---

## Part 3 — Documentation corrections needed

Anything EMAIL_INGESTION.md (or another doc) claimed that this staging pass proved wrong — list it here, then actually apply the correction as a follow-up doc-only change, same discipline as the EM10 documentation-audit pass this checklist follows.

- §23 didn't make a false claim (it never asserted `sources[]` was filtered to only cited documents), but it was silent on the question this scenario exposed — applied as an addendum, not a correction, documenting the EM10.5-driven fix: `sources[]` is now filtered to model-cited documents only, the portal renders citations as part of the single answer bubble rather than a second box, and both repos' citation-date formatting is pinned to UTC. See §23's new closing paragraph.

---

## Part 4 — Readiness decision for EM11

**Decision**: ☐ Proceed to EM11 ☐ Proceed with named caveats ☐ Block on fixing specific issues first

**Reasoning**:

**Decided by**:
**Date**:
