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
**Result**: ☐ Pass ☐ Fail ☐ Partial
**What actually happened**:
**Bugs found** (#):

### Scenario 2 — Import labeled mail
**Action**: an owner/admin configures an org allow rule; both members label a few real messages matching it, then run historical import.
**Expected**: only labeled-and-approved messages ingest, attributed to the correct member (`contributing_member_id`); each becomes a citable, retrievable chat answer showing subject/sender/date and a working "Open in Gmail" link that opens the real message.
**Result**: ☐ Pass ☐ Fail ☐ Partial
**What actually happened**:
**Bugs found** (#):

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
| | | | | |

---

## Part 3 — Documentation corrections needed

Anything EMAIL_INGESTION.md (or another doc) claimed that this staging pass proved wrong — list it here, then actually apply the correction as a follow-up doc-only change, same discipline as the EM10 documentation-audit pass this checklist follows.

-

---

## Part 4 — Readiness decision for EM11

**Decision**: ☐ Proceed to EM11 ☐ Proceed with named caveats ☐ Block on fixing specific issues first

**Reasoning**:

**Decided by**:
**Date**:
