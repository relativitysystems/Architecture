# Knowledge Gap Detection

Source repository: `relativitysystems/AIKB` (`services/runKnowledgeQuery.js`, `services/knowledgeGapKey.js`, `services/supabaseService.js`, `routes/knowledge.js`, `migrations/003_chat_history.sql`, `migrations/005_slack_origin_tracking.sql`, `migrations/008_gap_reported_by.sql`) and `relativitysystems/Relativity` (`public/portal/portal.js`, `public/admin/admin.js`, `public/admin/admin.html`, `routes/api.js`, `routes/admin.js`, `services/aikbService.js`).

## Overview

Knowledge-gap **detection** exists and runs automatically on every query. As of Backlog M4/M5/M12, **persistence, deduplication, and admin review are also real and automatic** — this used to be a design recommendation in this document; it is now implemented and covered by tests. This document reflects the current implementation, not a proposal.

## Current State

### Detection (automatic, heuristic — not LLM-judged)

Detection happens in two places inside `runKnowledgeQuery.js`, both triggered on every query, with no separate "is this a gap" model call:

1. **No chunks retrieved.** If vector search plus title-boost search return zero chunks above the similarity threshold, the response is flagged `isKnowledgeGap: true`, `gapReason: 'no_chunks_found'`, and a canned "couldn't find any relevant information" answer is returned without an LLM call.
2. **Answer-text phrase matching.** After the LLM generates an answer, `isKnowledgeGapAnswer()` lower-cases the text and checks it against a fixed list of substrings (`'not documented in'`, `'not found in'`, `"couldn't find"`, `'could not find'`, `'no information in'`, `'not in the knowledge base'`, `'not available in the knowledge base'`, `'not provided in the documentation'`, `'there is no information'`). A match sets `gapReason: 'answer_indicated_not_found'`.

The system prompt instructs the model to use one of these exact phrasings when it cannot answer from the retrieved context, and the heuristic then greps for that phrasing in the model's own output — this is pattern matching over LLM output, not a structured classification call.

When a gap is detected, the response's `sources[]` is forced empty and any model-hallucinated `Source:` line is rewritten to `Source: N/A`, so a gap answer can never appear to cite a document.

### Persistence (automatic, server-side, deduplicated — Backlog M4)

`runKnowledgeQuery()` now calls `createKnowledgeGap` itself on both gap-detection branches, for both `origin: 'portal'` and `origin: 'slack'` — a deliberate reversal of the prior "only an explicit portal click persists a gap" behavior. This is best-effort: a `createKnowledgeGap` failure is logged and swallowed, and never breaks the user-facing answer/Slack reply, since gap logging is review-workflow/analytics data, not core to answering the user.

Deduplication uses a new shared helper, `services/knowledgeGapKey.js#buildGapIdempotencyKey`, which derives `gap:v1:<clientId>:<sha256(normalized question)>:<ISO week>` — the question is lower-cased, whitespace-collapsed, and trailing punctuation stripped before hashing, and the key is bucketed by ISO week (not day): a recurring unresolved question resurfaces as a fresh row roughly weekly rather than staying silently open forever, while repeats of the same question within the same week collapse onto a single row. `supabaseService.js#createKnowledgeGap` upserts on this key — insert-on-conflict-do-nothing, then (on conflict) an update of `reason`/`session_id`/`message_id`/`updated_at` only. `status` and `reported_by` are deliberately never touched by the conflict-path update, so a recurring question never resets an admin's review state or misattributes a system-detected gap to a manual save (or vice versa).

The portal's explicit "Save gap" card (still present — `POST /api/knowledge/gaps`) now derives the *same* idempotency key from `(clientId, question)`, so in the common case a manual save converges onto an already-auto-detected row (refreshing `reason`) rather than duplicating it. It only performs a true first-insert (with `reported_by: 'user'`) when no matching auto-detected row exists yet.

### Origin attribution (Backlog M4/M12)

Both the auto-persist path and the manual-save path now populate `origin`/`origin_metadata` (columns added in migration 005, previously unused by any write path) and a new `reported_by` column (migration 008): `'system'` for pipeline-auto-detected gaps, `'user'` for an explicit manual save, `NULL` for the ~280 gap rows that predate this distinction (never backfilled — guessing a value for those rows would misattribute origin this migration has no way to actually know). Concretely:

- Slack-originated gaps carry `origin_metadata: {teamId, channelId, threadTs, eventId}` (unchanged from what the Slack flow already attached to chat sessions).
- Portal-originated gaps now carry `origin_metadata: {route: '/api/knowledge/query', memberRole}` — previously sent nothing.

### Admin review workflow (Backlog M5)

`aikb/routes/knowledge.js` adds `GET /api/knowledge/gaps/:clientId` (status-filterable list, `requireServiceRequest`-signed) and `PATCH /api/knowledge/gaps/:id` (status transition among `open`/`reviewed`/`resolved`/`ignored`), backed by `supabaseService.js#listKnowledgeGapsByClient`/`getKnowledgeGapById`/`updateKnowledgeGapStatus`. Relativity's admin console has a "Knowledge Gaps" tab (`public/admin/admin.html`/`admin.js`) fanning these out across every client (mirroring the existing `/analytics`/aikb-health fan-out pattern), showing client, question, reason, origin, reported-by, date, and an inline status `<select>` per row.

### What still does not exist

- **Client-facing gap visibility** — a client/member has no way to see which of their own questions were flagged as gaps; this remains admin-only. Not designed.
- **Richer conversation-level polish beyond the minimal M4/M12 slice** — e.g. surfacing gap counts/trends over time in the admin UI, or bulk status actions — was not part of M4/M5/M12's scope and is not built.

**Summary**: gap *detection*, *persistence*, *deduplication*, *origin attribution*, and *admin review* are all real, automatic (where "automatic" applies), and covered by tests (`aikb/test/runKnowledgeQuery.test.js`, `aikb/test/knowledgeGapKey.test.js`). Client-facing visibility is the one piece from the original recommendation that remains unbuilt.

## Current Limitations

- Gap detection quality depends entirely on the LLM using one of the expected trigger phrases in its own output — a model response that fails to answer the question without using one of those phrases would not be flagged, even though it should be.
- Migration `008_gap_reported_by.sql` (the `reported_by` column) has not yet been applied to either Supabase project as of this writing — see [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md)'s M12 entry. The corresponding code must not be deployed ahead of that migration.
- The ~280 gap rows created before this feature existed have `origin`/`origin_metadata`/`idempotency_key`/`reported_by` all `NULL` and are rendered as "legacy"/"—" in the admin UI, not retroactively attributed.
- Week-bucketed dedup means the *same* unresolved question can still appear as a new row every week it recurs — this is intentional (so a genuinely still-open gap doesn't go stale forever), not a bug, but it does mean the review queue is not a strict one-row-per-distinct-question model over long time horizons.

## Future Extension Points

- Client-facing gap visibility (a member seeing their own flagged questions) would reuse `listKnowledgeGapsByClient` with a member-scoped filter — the schema and core list function already support this; only a new member-facing endpoint/route and portal UI would be needed.
