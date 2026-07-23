# ADR-008 — Route AIKB Data Access Through a Client-Aware Database Provider

- **Status:** Proposed — not implemented
- **Date:** 2026-07-23
- **Owners:** Relativity Systems
- **Related repositories:** `relativitysystems/aikb`, `relativitysystems/Relativity`

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

- one provider owns construction and resolution of AIKB Supabase clients;
- all tenant-owned AIKB database and Storage paths use that provider;
- the shared project remains the only active target;
- existing APIs, ingestion jobs, retrieval, chat, collections, gaps, and deletion behavior remain unchanged;
- tests cover shared routing, validation, caching, and failure behavior;
- architecture documentation is updated from **Proposed** to **Accepted/Implemented** with verified source references.
