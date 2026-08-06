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
**Result**: ☑ Pass, with named caveats
**What actually happened**: an owner/admin configured an Allow rule for the managed Relativity/Knowledge Gmail label. Member A labeled three test emails (`Project Phoenix Onboarding SOP`, `Weekly Sales Meeting Agenda`, `Customer Refund Policy`) and ran a full historical scan (completed ~2026-08-04 19:42 UTC: 3 imported, 0 skipped, 0 failed). All three emails became retrievable in chat and produced accurate answers (onboarding steps, Thursday 9:00 AM meeting time, 30-day refund policy). The relevant Gmail email appeared as the primary source and a working "Open in Gmail" link was generated — the scenario's literal acceptance criteria (import correctness, attribution, citability, working deep link) were met. However, the chat UI also displayed a separate "Sources" bubble containing unrelated retrieval candidates (e.g. `test_b_long.txt`, `Accreited investors TO 2026.pdf.pdf`, `slack-test.txt`, the other unlabeled Gmail email), and the same cited email showed two different dates depending on whether it was read from the answer's inline `Source:` line or the structured sources list. Root-caused and fixed same-day (see Bugs 1-3 below); fix is unit-tested in both repos (aikb: 41/41 `runKnowledgeQuery.test.js`, 18/18 `openaiService.test.js`; Relativity: 29/29 `portalCitations.test.js`, 840/840 full suite) but **not yet manually retested against the live hosted environment** — re-run this scenario's live chat queries once the fix is deployed before considering it fully closed.
**Bugs found** (#): 1, 2, 3

### Scenario 3 — Remove a label
**Action**: Member A removes the label from a previously-ingested message (Scenario 2), then syncs again.
**Expected**: that message's content is tombstoned (no longer retrievable in chat) on the next sync; Member B's content is untouched.
**Result**: ☐ Pass ☐ Fail ☐ Partial
**What actually happened**:
**Bugs found** (#):

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
| 1 | Scenario 2 | Unrelated retrieval candidates (documents the model never actually cited) displayed as answer sources in the portal's Sources box — e.g. asking about the weekly sales meeting also surfaced `test_b_long.txt`, an accredited-investors PDF, `slack-test.txt`, and the unrelated Project Phoenix email. Root cause: `runKnowledgeQuery.js` returned every retrieved chunk's document as a "source," regardless of whether the model's answer relied on it. Fix: the model now reports which numbered context items it actually used (`Cited: [n, n]`), validated against the retrieved set and used to filter `sources[]`. | Medium | Fix implemented, unit-tested (aikb `runKnowledgeQuery.test.js` — new filtering/fallback tests), **not yet manually retested live** |
| 2 | Scenario 2 | Sources rendered in a visually disconnected second bubble/card ("SOURCES") below the answer bubble, duplicating the model's own inline "Source:" citation line already present in the answer text. Root cause: `portal.js` appended the sources box as a DOM sibling of the answer bubble, and `portal.css` gave it the same background/border-radius as the bubble itself — visually indistinguishable from a second message. Fix: sources box is now nested inside the answer bubble (CSS divider, not a second card); the model's redundant inline "Source:" line is stripped from the displayed text whenever the structured box will show. | Medium | Fix implemented, unit-tested (Relativity `portalCitations.test.js` — new `stripInlineSourceLine` tests), **not yet manually retested live** |
| 3 | Scenario 2 | The same cited email showed two different dates: the answer's inline `Source:` line (server-formatted) vs. the structured sources list (browser-formatted). Root cause: `formatCitationDate` was duplicated in `aikb/services/openaiService.js` (server) and `Relativity/public/portal/portalCitations.js` (browser), both calling `toLocaleDateString` without pinning a timezone — each defaulted to its own runtime's local timezone, disagreeing near a timezone boundary. Fix: both now pass `timeZone: 'UTC'` explicitly. | Minor | Fix implemented, unit-tested (new UTC boundary-case tests in both repos' citation test files), **not yet manually retested live** |

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
