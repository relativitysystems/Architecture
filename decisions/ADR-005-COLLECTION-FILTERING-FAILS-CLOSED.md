# ADR-005: Collection filtering is enforced in AIKB's retrieval SQL and fails closed

## Status
Accepted — Implemented

## Date
Undated (decision); collections implemented in AIKB migration `006_knowledge_collections.sql`

## Context

The platform needed a way to restrict what a given caller (particularly a lower-trust channel like Slack) can retrieve, beyond the existing `client_id` tenant boundary. At the time of the original architecture discovery, no collections/department/per-document access model existed in either repository — retrieval was scoped only to `client_id`. This blocked any future channel-scoped access model and was flagged as a blocking prerequisite before building Slack: a channel-based surface is exactly where over-broad retrieval is most damaging, since a channel can include people outside the roles that would normally see certain documents.

A separate question, resolved alongside this one: if a caller's authorized-collections list is missing, empty, malformed, or unresolvable, should retrieval default to "search everything" or "search nothing"?

## Decision

**Collection filtering is enforced inside AIKB's retrieval SQL itself, in the same `WHERE` clause that already enforces `client_id`** (`match_knowledge_chunks`, extended with a `match_collection_ids UUID[]` parameter) — never as an application-code post-filter after chunks have already been fetched into the Node process.

**Semantics, exactly as implemented:**
- `match_collection_ids = NULL` → unrestricted (searches every collection in the client's workspace). This is the **portal's** default (`/query`'s `allowedCollectionIds` defaults to `null`) — a portal user is already scoped to their own client's full workspace, so no further restriction applies by default.
- `match_collection_ids = []` (empty array) → matches **zero rows**. This is the **Slack path's** default (`/ask`'s `allowedCollectionIds` defaults to `[]` if not explicitly supplied) — **fail closed**, meaning "search nothing" until a client explicitly allow-lists collections via `slack_collection_access`.
- A title-boost hybrid layer (`searchChunksWithTitleBoost`) is pre-filtered by the same `allowedCollectionIds` — a restricted document's filename can never force its chunks into a response it isn't entitled to see, even via the filename-matching shortcut.

Relativity resolves identity and role context (owner/admin for collection-management mutations); AIKB owns collection definition, assignment, and enforcement end to end — a simpler arrangement than the fully split Relativity-defines/AIKB-enforces model originally proposed (see [../history/ARCHITECTURE_REVIEW_PHASES.md](../history/ARCHITECTURE_REVIEW_PHASES.md) for that original design).

## Alternatives Considered

- **Application-layer post-filtering** (fetch broadly, discard disallowed chunks in Node before building the prompt): rejected — a restricted chunk would still have been retrieved into the process and could leak via a bug in the filtering code, a timing side-channel, or a future code path that forgets to apply the filter. Enforcing inside the SQL statement means there is no code path that returns a disallowed chunk in the first place.
- **Default to "search everything" for Slack too, same as the portal**: rejected — a Slack channel can include a broader or different audience than the portal's authenticated members, so an empty/unresolved allow-list must mean "nothing," not "everything." This is the single most consequential authorization default in the platform: getting it backwards would silently over-expose documents to any channel where Slack is installed.
- **A fully split Relativity-owns-definition / AIKB-owns-assignment model with a signed `entitledCollectionIds` list** (the original proposal, requiring a synchronized collection registry and a full signed-request platform): deferred — not built for the current scope. AIKB owns collection definition and assignment entirely today; see [ADR-004](ADR-004-SIGNED-SERVICE-REQUESTS.md) for the related, still-unbuilt signed-envelope platform this would have depended on.

## Consequences

- Every client is seeded with two collections at creation: **"General"** (`is_default = true`, the fallback target for newly-ingested documents) and **"Slack"** (a starter collection with no special code behavior beyond its name, for admins to scope Slack retrieval into).
- A document's `collection_id` is assigned once, at first ingest, to the client's default collection; a reindex never resets an already-moved document's collection.
- Collections currently only gate **Slack's** retrieval scope — the portal's own chat query is unaffected by collections (`allowedCollectionIds: null` always). Extending collection-based filtering to the portal's own chat is tracked as future work.
- Any future channel-based connector (Teams, a future email adapter) should default to the same fail-closed pattern Slack established, not the portal's unrestricted default, unless there is a specific reason that channel's trust model matches the portal's.

## Implementation Evidence

- `match_knowledge_chunks` SQL function (`aikb/migrations/001_knowledge_base_schema.sql`, extended in `006_knowledge_collections.sql`) — `match_collection_ids UUID[] DEFAULT NULL`, filtering `kd.collection_id = ANY(match_collection_ids)` only when non-null.
- `slack_collection_access` join table (Relativity-managed via `GET/PUT /api/integrations/slack/collections`, owner/admin only) resolves the Slack allow-list fresh at request time.
- `knowledge_collections` table, CRUD routes (`routes/knowledge.js`), and `PATCH /api/knowledge/document/:id/collection` for reassignment.
- Full detail: [../architecture/AIKB.md](../architecture/AIKB.md) — Knowledge Collections, Collection Filtering.

## Related Documents

- [../architecture/AIKB.md](../architecture/AIKB.md)
- [../architecture/CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md)
- [../architecture/SECURITY.md](../architecture/SECURITY.md)
- [ADR-002-AIKB-OWNS-KNOWLEDGE-PROCESSING.md](ADR-002-AIKB-OWNS-KNOWLEDGE-PROCESSING.md)
- [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md)
- [../history/ARCHITECTURE_REVIEW_PHASES.md](../history/ARCHITECTURE_REVIEW_PHASES.md)
