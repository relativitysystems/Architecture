# Google OAuth App Verification — Prerequisite Checklist

Working document for [EMAIL_INGESTION.md](EMAIL_INGESTION.md) §26/§29's "provider app-verification" dependency. This is a **fill-in-the-blanks execution record for a non-engineering, external process**, not a design document — same discipline as [EM10_5_STAGING_CHECKLIST.md](EM10_5_STAGING_CHECKLIST.md). It tracks getting the Gmail OAuth Client off the "unverified app" warning screen and off the ≤100-test-user cap, for real customer use.

**Status: Not started.** No privacy policy page exists yet (verified 2026-08-01 — `Relativity/public/` has no `privacy.html`/`terms.html`, only `marketing/index.html`, `login.html`, `portal.html`, etc.). OAuth Client, Client Secret, and redirect URI are already configured correctly in both local `.env` and Vercel (Production + Preview) — that groundwork is done; this checklist is what's still missing.

**Does not block EM10.5.** Testing-mode + a Test users allowlist (already what EM10.5 uses) works without verification. This checklist matters for opening the app to real customers beyond that allowlist — run it in parallel with EM10.5, not sequenced after it, per EMAIL_INGESTION.md §29's explicit guidance that verification lead time should start early.

**One thing to internalize before starting**: `gmail.readonly` and `gmail.labels` are Google-classified **restricted** scopes (§26), not just sensitive ones. That means two separate approvals, not one — the standard OAuth consent-screen review, *and* a CASA security assessment. Confirm current requirements/tiers/cost on Google's own verification docs before estimating timeline or budget; don't rely on a remembered number here, Google changes this program periodically.

---

## Part 0 — Legal / content prerequisites (blocks submission entirely without these)

- [ ] **Privacy policy page live at a public URL** (e.g. `https://relativitysystems.ai/privacy`). Must accurately describe, at minimum: what Gmail data is accessed (`gmail.readonly` read-only mail content/metadata, `gmail.labels` for the managed "Relativity/Knowledge" label), why, how it's processed (ingested into AIKB's pipeline, sent to OpenAI for embeddings/generation, stored in Supabase — see EMAIL_INGESTION.md §26's data-processing-chain note), retention, and how a user/member revokes access or requests deletion (§24.1 covers the functional right-to-delete mechanism this page should describe in plain language).
- [ ] **Terms of service page** (recommended, sometimes requested during review even if not strictly required for every scope tier).
- [ ] Both pages linked from the app's public homepage footer/nav — Google's reviewer checks that the links are actually reachable from the app, not just present as bare URLs in the console form.
- [ ] Support email address that's actually monitored (used on the OAuth consent screen and possibly contacted by Google's review team with questions).

**This is the concrete, buildable gap right now** — nothing else in this checklist can be submitted until the privacy policy exists. Good candidate to build next.

## Part 1 — OAuth consent screen configuration (Google Cloud Console)

- [ ] App name, logo (120×120px png), and app homepage URL set and accurate.
- [ ] User support email and developer contact email(s) set.
- [ ] **Authorized domain** (`relativitysystems.ai`) added and **verified via Google Search Console** — this is a separate verification step from the OAuth review itself and can be done immediately, no dependency on anything else here.
- [ ] Privacy policy URL and terms of service URL fields point at the Part 0 pages.
- [ ] Scopes list matches exactly what the code requests today — cross-check against `Relativity/services/gmailService.js`'s actual scope request before submitting, the same discipline EM10.5's own checklist already applies to this (`gmail.readonly`, `gmail.labels`, `openid`, `email`, `profile`).
- [ ] Test users list still has the EM10.5 staging accounts (or is expanded) — keep this current independent of verification progress, since it's what unblocks all pre-verification testing.

## Part 2 — Scope justification

For each *sensitive/restricted* scope, Google's review form asks for a written justification plus (for restricted scopes) evidence in the demo video that the scope is used narrowly. Draft these in advance rather than writing them live in the console form:

- [ ] **`gmail.readonly` justification** — narrow framing: used only to read messages a member has explicitly labeled AND that match an admin-configured organization policy, never blanket mailbox access; no send/modify/compose/delete scope is requested (this narrowness is already a documented design property, EMAIL_INGESTION.md §25 table). *(Updated 2026-08-26, EM10.6: the original draft also justified reading policy-matching-but-unlabeled mail under a since-removed "automatic mode" — Automatic Email Ingestion no longer exists, so this justification is narrower and stronger than the original draft, not weaker.)*
- [ ] **`gmail.labels` justification** — used only to create/read the single managed "Relativity/Knowledge" label used for the label-driven consent mechanism, not general label management.
- [ ] Internal legal/founder review of both justification drafts before submission — these are the sentences Google's reviewer weighs most heavily for restricted scopes.

## Part 3 — Demo video

Google requires a screen recording (not a marketing video — a literal walkthrough) showing the actual OAuth consent screen exactly as an end user sees it, followed by how the granted data is used in-product.

- [ ] Reuse [../go-to-market/DEMO_VIDEO_STRATEGY.md](../go-to-market/DEMO_VIDEO_STRATEGY.md)'s existing honesty guardrails and script framework as the starting point — same "show real product behavior, don't dramatize" discipline applies here, this is a stricter compliance audience than a sales prospect.
- [ ] Record: connect flow start → Google's real OAuth consent screen (showing the exact scopes requested) → landing back in the portal → the label/policy-gated ingestion happening → a citation in chat pointing back to the ingested email.
- [ ] Video is unlisted/accessible to Google's reviewers per their submission instructions (check current submission format — YouTube unlisted link is the common pattern, confirm still current).

## Part 4 — CASA security assessment (restricted-scope tier)

- [ ] Confirm current tier/process/cost on Google's own docs (App Defense Alliance / CASA) — do not plan against a remembered figure, this program's specifics change.
- [ ] Identify and engage an approved assessor if the current process requires a paid third party for this app's tier.
- [ ] Scope the assessment to what's actually exposed: the signed service-request boundary (ADR-004), encrypted OAuth credential storage (ADR-006), fail-closed collection filtering (ADR-005) are all existing, documented controls worth having ready to describe/evidence to an assessor rather than re-discovering during the assessment.
- [ ] Budget real calendar time for this step specifically — it is very likely the longest single item in this whole checklist.

## Part 5 — Submit and track

- [ ] Submit for verification once Parts 0–3 are complete (Part 4's assessment may run in parallel with or after initial submission, depending on Google's current process — confirm sequencing when you reach this step, it has changed over time).
- [ ] Record submission date here: __________
- [ ] Track any reviewer follow-up questions/requested changes in the bug-log style below, so a second reviewer round doesn't start from zero.
- [ ] Record approval date here: __________

| # | Date | Reviewer request / question | Response / fix | Status |
|---|---|---|---|---|
| | | | | |

---

## Related documents

- [EMAIL_INGESTION.md §26](EMAIL_INGESTION.md#26-privacy-and-compliance-considerations) — where this dependency was first flagged, plus the DPA/HIPAA/privilege considerations adjacent to it (not solved by this checklist — legal/business follow-ups, noted there).
- [EM10_5_STAGING_CHECKLIST.md](EM10_5_STAGING_CHECKLIST.md) — the parallel, non-blocking real-account validation track; both can run at the same time.
- [../go-to-market/DEMO_VIDEO_STRATEGY.md](../go-to-market/DEMO_VIDEO_STRATEGY.md) — reused for Part 3's video discipline.
- [../decisions/ADR-006-OAUTH-CREDENTIAL-ENCRYPTION.md](../decisions/ADR-006-OAUTH-CREDENTIAL-ENCRYPTION.md) — evidence to have ready for Part 4.
