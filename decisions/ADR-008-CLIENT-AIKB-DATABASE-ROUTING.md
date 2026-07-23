# ADR-008 — Route AIKB Data Access Through a Client-Aware Database Provider

- **Status:** Accepted — Implemented (shared-routing phase only; dedicated per-client databases remain future work)
- **Date:** 2026-07-23
- **Owners:** Relativity Systems
- **Related repositories:** `relativitysystems/aikb`, `relativitysystems/Relativity`

## Implementation Record

Implemented 2026-07-23, in `relativitysystems/aikb` only. Every acceptance criterion below is met for the shared-project phase; dedicated databases, the `client_database_assignments` table, and any Global routing lookup are still not built — exactly as scoped.

**Source references:**

- `aikb/services/aikbDatabaseProvider.js` — the provider. Exports `getAikbDatabase(clientId)` (async; validates `clientId`, fails closed, returns `{ supabase, storageBucket, mode: 'shared' }` from a single lazily-constructed, process-cached Supabase client) and `getSharedAikbDatabaseForAdminOperation()` (a narrow, explicitly-documented exception used only by the one genuinely cross-client function — see below).
- `aikb/services/supabaseService.js` — every exported function that touches AIKB's Supabase project or Storage bucket now resolves its client via `getAikbDatabase(clientId)` (the top-of-file module-scope `aikbSupabase` constant is gone). The separate `globalSupabase` client remains constructed inline in this same file, unchanged, per this ADR's requirement not to touch it. Functions that previously operated on an id alone with no `clientId` parameter (`getKnowledgeDocumentById`, `getCollectionById`, `renameCollection`, `deleteCollection`, `moveDocumentCollection`, `deleteChunksForDocument`, `insertKnowledgeChunks`, `getKnowledgeGapById`, `updateKnowledgeGapStatus`, `updateIngestionJob`, `logIngestionError`, `markDocumentIndexed`, `markDocumentDeleted`, `markDocumentError`, `downloadFromStorage`, `deleteFromStorage`, `getSlackRequestByIdempotencyKey`, `markSlackRequestDelivered`, `markSlackRequestFailed`) now take `clientId` as an explicit parameter, threaded from the caller's already-authorized context. `getDistinctClientIds` (unused cross-client admin enumeration — no route calls it) is the one documented exception: it resolves via `getSharedAikbDatabaseForAdminOperation()` instead of `getAikbDatabase(clientId)`, since it has no single client to scope to.
- `aikb/routes/knowledge.js` and `aikb/inngest/functions.js` — every call site updated to pass the already-authorized `clientId` (from `req.serviceRequest`, `req.context`, or `event.data.clientId`) into the newly-parameterized `supabaseService.js` functions. No route or job reads an unvalidated `clientId` from a request body to resolve a database; every existing `.eq('client_id', clientId)` filter and `match_client_id` RPC parameter is unchanged.
- `aikb/test/aikbDatabaseProvider.test.js` — new. Covers: two different valid `clientId`s resolving to the same shared client instance; the client being constructed exactly once across repeated/different-client calls (cache reuse, not per-request construction); missing/empty/malformed `clientId` failing closed; and missing `AIKB_SUPABASE_URL` configuration producing a clear startup error.
- `aikb/test/chatSessionOriginFilter.test.js`, `aikb/test/redactChatSession.test.js`, `aikb/test/slackRequestLogService.test.js` — updated to also clear the require-cache entry for `aikbDatabaseProvider.js` (in addition to `config` and `supabaseService.js`) when substituting a fake `@supabase/supabase-js`, and to pass `clientId` to the now-reparameterized `markSlackRequestDelivered`/`markSlackRequestFailed`.

**Not touched, and why:** `relativitysystems/Relativity/services/aikbService.js#getAikbSupabase()` constructs a separate AIKB Supabase Storage client directly (used only by `uploadToStorage`, for the portal's direct-to-bucket upload leg prior to calling AIKB's `/ingest` HTTP route). This is a different deployable that cannot import AIKB's in-process provider module, so it is out of this ADR's implementation scope, which targets AIKB-owned access from within AIKB's own codebase. It already independently satisfies most of this ADR's spirit (single construction site, cached, fails closed on missing config) but will need its own routing awareness — or a change to route the upload itself through AIKB's API instead of writing directly to Storage — if and when dedicated per-client projects are introduced. Tracked as follow-up work, not a gap in this implementation.

## Context

AIKB currently uses one shared Supabase project for every client's knowledge data. Tenant-owned rows are separated by `client_id`, and the service constructs one static AIKB Supabase client from `AIKB_SUPABASE_URL` and `AIKB_SUPABASE_SERVICE_KEY`.

This design is appropriate for the current product stage and should remain the default. A future enterprise customer may nevertheless require a dedicated AIKB Supabase project for contractual isolation, regional placement, compliance, customer-specific retention, or performance guarantees.

If application code continues importing and using one process-wide AIKB Supabase client directly, adding dedicated projects later would require changing database access throughout ingestion, retrieval, storage, chat history, collections, gap detection, deletion, and background jobs.

## Decision

All AIKB-owned database and Storage access should be routed through one client-aware provider abstraction.

The application-facing contract should be conceptually equivalent to:

```js
const aikbDatabase = await getAikbDatabase(clientId);
```

The returned object should expose the Supabase database client and the Storage client or methods required by the caller. During the first implementation, every valid `clientId` must resolve to the existing shared AIKB Supabase project. No client should be migrated, no new Supabase project should be provisioned, and no runtime behavior should change.

The provider becomes the only module permitted to construct or select an AIKB Supabase client. The separate Global Supabase client remains static because identity, client membership, and control-plane data continue to live in `Relativity_Global`.

## Initial Implementation Requirements

1. Add a dedicated AIKB database-provider module, for example `services/aikbDatabaseProvider.js`.
2. Export a client-aware function such as `getAikbDatabase(clientId)`.
3. Validate that `clientId` is present before resolving a database.
4. Return the existing shared AIKB Supabase client and existing Storage bucket configuration for every client.
5. Cache constructed clients so requests do not create a new Supabase client repeatedly.
6. Refactor AIKB services, routes, and Inngest functions so tenant-owned database and Storage operations obtain their client through the provider rather than importing a static AIKB client directly.
7. Do not alter the Global Supabase client or authorization model.
8. Preserve every current `client_id` filter and RPC tenant parameter. Database routing supplements tenant filtering; it does not replace it.
9. Add tests proving two different client IDs resolve to the shared project today and that missing client IDs fail closed.
10. Keep all current environment variables working. Do not introduce dedicated-project credentials yet.

## Future Control-Plane Model

When dedicated databases are introduced, `Relativity_Global` should own a routing record similar to:

```text
client_database_assignments
---------------------------
client_id                  UUID PRIMARY KEY
mode                       shared | dedicated
project_reference          TEXT
credential_reference       TEXT
storage_bucket             TEXT
schema_version             TEXT
status                     provisioning | active | suspended | failed
created_at                 TIMESTAMPTZ
updated_at                 TIMESTAMPTZ
```

Raw service-role keys should not be stored directly in ordinary database columns. `credential_reference` should point to an approved secrets-management mechanism. Routing records are control-plane data and therefore belong in Global, not inside a client's AIKB project.

## Future Resolution Behavior

The provider may later resolve clients as follows:

```js
if (assignment.mode === 'dedicated') {
  return getCachedDedicatedClient(assignment);
}

return sharedAikbDatabase;
```

A production implementation must define cache expiration, credential rotation, assignment-change propagation, connection health checks, and behavior when Global cannot be reached. It must fail closed rather than silently routing a dedicated client into the shared project.

## Migration Strategy for a Client Moving to Dedicated Storage

A future migration should be explicit and resumable:

1. Provision the dedicated Supabase project.
2. Apply the complete AIKB migration set and verify `schema_version`.
3. Copy the client's rows, preserving IDs and relationships.
4. Copy original files under the client's Storage prefix when applicable.
5. Verify row counts, chunk counts, checksums, vector dimensions, collections, and retrieval behavior.
6. Temporarily pause or dual-write tenant mutations during cutover.
7. Change the Global routing assignment atomically.
8. Run read and write smoke tests.
9. Retain the source data for a defined rollback window.
10. Delete shared copies only after approval and expiry of the rollback window.

Dual-write is not part of the initial abstraction and should not be added prematurely.

## Consequences

### Positive

- The current shared-database model remains simple and inexpensive.
- Future dedicated databases can be added without rewriting every AIKB feature.
- Enterprise isolation becomes a routing and migration concern rather than a full application redesign.
- Tests can exercise routing independently from business logic.

### Costs

- AIKB data-access functions must consistently receive or retain `clientId`.
- A provider abstraction adds a small amount of indirection.
- Background jobs must resolve the client database inside the job execution path.
- Future dedicated projects require migration orchestration, schema fleet management, monitoring, and secret rotation.

## Rejected Alternatives

### Create one Supabase project per client immediately

Rejected because present scale does not justify the operational overhead. The shared Pro project should support the expected early customer volume, and database-per-client would multiply migrations, secrets, monitoring, and provisioning work.

### Keep the static client until the first enterprise request

Rejected because direct static-client imports would spread further as email and call-transcript ingestion are added, making the eventual refactor significantly riskier.

### Remove `client_id` filtering after dedicated databases exist

Rejected. Dedicated projects reduce blast radius but do not replace defense-in-depth, auditability, or protection against accidental cross-client routing.

## Acceptance Criteria

This ADR is considered implemented when:

- [x] one provider owns construction and resolution of AIKB Supabase clients — `aikb/services/aikbDatabaseProvider.js`;
- [x] all tenant-owned AIKB database and Storage paths use that provider — every function in `aikb/services/supabaseService.js`, called from `aikb/routes/knowledge.js` and `aikb/inngest/functions.js`;
- [x] the shared project remains the only active target — `mode: 'shared'` always, no `client_database_assignments` table, no dedicated-project credentials configured;
- [x] existing APIs, ingestion jobs, retrieval, chat, collections, gaps, and deletion behavior remain unchanged — full AIKB test suite passes (74/74, `npm test` in `aikb/`), no request/response contract or schema changes;
- [x] tests cover shared routing, validation, caching, and failure behavior — `aikb/test/aikbDatabaseProvider.test.js`;
- [x] architecture documentation is updated from **Proposed** to **Accepted/Implemented** with verified source references — this document, plus `Architecture/architecture/AIKB.md`, `Architecture/docs/repositories/AIKB_REPO.md`, and `Architecture/docs/supabase/aikb/SECURITY.md`.

Not implemented (deliberately, per scope): dedicated per-client Supabase projects, the `client_database_assignments` Global routing table, any Global routing lookup inside the provider, credential rotation, and tenant cutover/migration tooling. See "Future Control-Plane Model" and "Migration Strategy" above for the design a later ADR/implementation would follow.
