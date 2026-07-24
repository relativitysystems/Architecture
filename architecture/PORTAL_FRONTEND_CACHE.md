# Portal Frontend Cache (Milestone)

Source repository: `relativitysystems/Relativity`, `public/portal/portal.js`, `public/portal/portalCache.js` (new). Cross-reference [product/CLIENT_PORTAL.md](../product/CLIENT_PORTAL.md) for the overall portal architecture and [SECURITY.md](SECURITY.md) for the authentication flow this plan must not alter.

## Problem

The portal re-fetches Knowledge Collections, Documents, and Team Members from scratch on every page load and tab visit, showing a full loading/skeleton state each time even when the user was just looking at the same data seconds ago. There is no client-side caching beyond a few unrelated `localStorage` UI preferences (collection chat filter, dismissed job ids). This milestone adds a small, tab-scoped, stale-while-revalidate (SWR) cache so the portal *feels* faster after the first load, without touching the server, auth flow, or introducing new infrastructure.

## Current Loading Behavior (as inspected)

`public/portal/portal.js` is a single ~2,600-line bootstrap IIFE. Relevant state and load paths:

- `loadedCollections` (module-scoped, `null` until fetched) — populated by `initCollectionsSection()`'s `loadCollectionsTable()`, which calls `GET /api/collections` and always renders a skeleton-row loading state first.
- `GET /api/collections` is actually called from **three** independent sites: the Collections tab (`loadCollectionsTable`), the Slack "collections Slack can search" panel (`loadSlackAllowedCollections`), and the chat "search scope" filter (`loadKbCollectionsFilter`) — each does its own fetch, with no sharing today.
- Documents: `fetchDocuments()` calls `GET /api/knowledge/documents`; `loadDocuments()` (initial load) shows two skeleton rows first, then renders. `refreshDocuments()` (used after every mutation and during upload/delete polling) re-fetches without a loading skeleton. Document count is derived client-side from the fetched list (`rows.length`), not a separate endpoint.
- Team members: `GET /api/team/members` is called from **two** independent sites — the top-level `loadMembers()` (only feeds the onboarding-progress checklist) and `initTeamSection()`'s own `loadTeamMembers()` (renders the actual Team tab table, with its own skeleton rows).
- None of these three resources are cached today; every tab switch or page refresh re-fetches from the network and shows a loading state even for data that hasn't changed.
- Existing client-side storage in the portal is unrelated to this milestone and uses `localStorage`: `kbAllowedCollections:{clientId}` (chat collection filter) and `dismissedIngestionJobs:{clientId}` (locally-hidden job rows). This plan does not touch either.
- Logout (`btn-logout` handler) calls `supabase.auth.signOut()` then redirects to `/login.html`. A second, less obvious sign-out path exists at bootstrap: if `GET /auth/me` returns `{authenticated: false}` (membership disabled/invalid), the portal also calls `supabase.auth.signOut()` before redirecting with an error reason.

## Scope

Cache exactly three GET resources, as read-through-with-background-refresh, using `sessionStorage`. No Redis, no service workers, no new dependencies, no offline mode.

### Resources Cached

| Resource key | Endpoint | Rendered by |
|---|---|---|
| `collections` | `GET /api/collections` | Collections tab table (`loadCollectionsTable`) |
| `documents` | `GET /api/knowledge/documents` | Documents tab list + count (`loadDocuments` / `fetchDocuments`) |
| `teamMembers` | `GET /api/team/members` | Team tab table (`initTeamSection`'s `loadTeamMembers`) and the onboarding-progress checklist (`loadMembers`) |

The Slack-allowed-collections panel and the chat collection filter also call `GET /api/collections`; they are left as plain fetches in this milestone (not expanded to the cache) to keep the change small — see Non-Goals.

### Not Cached (explicitly excluded per requirements)

Access/refresh tokens, OAuth credentials, email bodies, document contents/uploaded file bytes, raw chat/RAG responses, chat sessions/messages, ingestion job status (changes too fast / is actively polled), analytics, import history, Slack status. The Supabase session itself continues to be managed exclusively by the Supabase JS SDK — this cache never stores or reads session/auth state.

## Storage Choice: `sessionStorage`

`sessionStorage` (not `localStorage`) is used because:
- Cached portal data should survive an in-tab refresh (so the "instant" effect works across reloads), but
- it must disappear when the tab is closed, since these are shared/kiosk-risk machines in some deployments and knowledge-base contents/team rosters are client-confidential.
- `sessionStorage` is per-tab, which also means two tabs open on two different client sessions (e.g., after switching accounts) never share a cache entry — an extra isolation property `localStorage` would not give us for free.

## Cache Key Format

```
relativity:portal:v1:{clientId}:{memberId}:{resource}
```

- `v1` is a **cache-format namespace** baked into the key itself — bumping it (e.g. to `v2`) instantly orphans every old-shaped entry without needing a migration, since old and new keys simply never collide.
- `clientId` and `memberId` come from `GET /auth/me`'s response (`me.clientId`, `me.memberId`), exactly as already resolved server-side and already in scope in `portal.js`'s bootstrap closure — never re-derived or trusted from anything else client-side.
- `resource` is one of `collections`, `documents`, `teamMembers`.
- There is **no global/shared key** at any point — every read/write goes through a key-builder that requires both ids, so it is not possible to construct a cross-tenant or cross-member key by accident.

## Cache Entry Shape

Every entry stored under a key is a JSON string of:

```json
{ "version": 1, "savedAt": 1737590000000, "data": { /* resource payload, same shape the render function expects */ } }
```

- `version` is the **entry schema version** (separate concern from the `v1` in the key namespace) — if the entry shape itself changes in the future, bumping this constant invalidates all existing entries on read without needing to touch the key format.
- `savedAt` is `Date.now()` at write time, used to compute staleness against the resource's TTL.
- `data` is stored exactly as the resource's existing fetch function already normalizes it (e.g. the array `fetchDocuments()` returns, the `collections` array, the `members` array) — so the existing render functions need no reshaping to consume cached vs. fresh data.

## TTLs (final)

| Resource | TTL | Rationale |
|---|---|---|
| `collections` | 5 minutes | Collections change rarely (owner/admin-only CRUD); matches the suggested value. |
| `documents` | 2 minutes | Most actively mutated resource (uploads/deletes/moves); matches the suggested value, short enough that a teammate's recent upload shows up soon even without a manual mutation on this tab. |
| `teamMembers` | 5 minutes | Roster changes are infrequent; matches the suggested value. |

These are read-side TTLs only (governing whether a cached entry is still eligible to be shown/refreshed-from); they do not create any server polling — the background refresh only happens when the resource's existing load function is actually invoked (tab open / page load), never on a timer.

## Stale-While-Revalidate Behavior

Implemented once, in `public/portal/portalCache.js`, as `PortalCache.staleWhileRevalidate(...)`, reused identically by all three resources instead of duplicating the control flow in `portal.js`:

1. Look up the cache entry for `(clientId, memberId, resource)`. Reject (and remove) it if malformed, wrong `version`, or older than the resource's TTL.
2. If a valid entry exists, immediately invoke the caller's render callback with the cached data (`{ fromCache: true }`) — no loading skeleton is shown in this case.
3. Always issue the real network request in the background (regardless of step 2).
4. On success: write the fresh response to cache (resetting `savedAt`), and invoke the render callback again with the fresh data — *unless* it is deep-equal (`JSON.stringify` comparison) to what was just shown from cache, to avoid a visible flicker/re-render when nothing actually changed.
5. On failure: if step 2 already rendered valid cached data, do nothing further (no error surfaced, cached view stands). If there was no valid cache (first-ever load, or expired-and-nothing-to-show), invoke the caller's existing error callback — i.e. today's existing "Failed to load…" behavior is preserved unchanged for the no-cache case.
6. When no cache exists at all, the loading skeleton is shown exactly as it is today while the single network request resolves.

## Mutation Invalidation Rules

General rule: whenever a resource's own load/reload function (`loadCollectionsTable`, `fetchDocuments`/`refreshDocuments`, `loadTeamMembers`) actually completes a real network fetch, it writes the fresh result to cache — so any code path that already re-invokes that reload function after a mutation (nearly everything below) self-heals the cache as a side effect. On top of that, `PortalCache.invalidate(clientId, memberId, resource)` is called explicitly at each mutation site immediately on success, before the reload, both for defense-in-depth and so a resource that *isn't* reloaded on that particular path (see Team role change below) still can't serve a stale cached read next time.

| Mutation | Invalidates |
|---|---|
| Document upload (single, multi-file, folder, ZIP, Google Drive) | `documents`, `collections` (a collection's `documentCount` changes) |
| Document delete | `documents`, `collections` |
| Document move to another collection | `documents`, `collections` |
| Collection create / rename / delete | `collections` |
| Team invite | `teamMembers` |
| Team role change | `teamMembers` (this is the one path with no existing automatic reload on success today — invalidation alone ensures the next load is a real fetch, not a stale cache hit) |
| Team member disable / re-enable / revoke | `teamMembers` |
| Team invite resend | none (no list data changes) |

No mutation is followed by a stale read: every success path either reloads (which rewrites the cache) or, at minimum, invalidates so the next reload cannot short-circuit on stale data.

## Logout Cleanup

- `PortalCache.clearAll()` iterates `sessionStorage` for keys prefixed with `relativity:portal:v1:` and removes only those — it never touches unrelated `sessionStorage`/`localStorage` entries (e.g. Supabase's own session storage, the portal's `localStorage`-based UI prefs).
- Called from both existing sign-out paths in `portal.js`:
  1. The `btn-logout` click handler, before/alongside `supabase.auth.signOut()`.
  2. The bootstrap guard where `GET /auth/me` returns `{authenticated: false}` (invalid/disabled membership) — cache is cleared before the redirect to `/login.html`, so a disabled member's last-seen data can't linger in `sessionStorage` under their old key past that point.
- The existing Supabase sign-out call itself is unchanged — this milestone only adds a cache-clear step alongside it.

## Tenant Isolation

- Every cache read/write goes through one key-builder (`buildKey(clientId, memberId, resource)`); there is no code path that reads or writes a cache entry without both ids, so it is structurally impossible to construct the shared/global key the requirements prohibit.
- `clientId`/`memberId` are the same server-resolved values `portal.js` already trusts for every API call (`GET /auth/me`) — this cache introduces no new trust boundary and never reads these ids from anything client-supplied.
- Two members of the same client get distinct cache entries (their `memberId` differs), so one member's cached view can never be shown to another, even within the same client.
- `sessionStorage`'s own per-tab/per-origin scoping is an additional, free backstop: a cache entry can never be read across browser tabs or origins regardless of key correctness.
- Cached data is a pure display optimization: every "fresh" fetch behind the SWR wrapper still sends the normal `Authorization: Bearer <accessToken>` header and goes through the exact same `clientAuth`/`requireRole` server-side checks as today. The server remains the sole source of truth and sole authorization boundary; the cache never grants access to anything the server wouldn't have returned anyway.

## Testing and Acceptance Criteria

**Unit tests** (`test/portalCache.test.js`, `node:test`, matching the repo's existing convention — see `test/documentsList.test.js`), with a minimal in-memory `sessionStorage` stub installed on `global`:

1. Cache keys embed `clientId`, `memberId`, and resource name in the documented format.
2. A valid, non-expired entry is returned by `get()`.
3. An expired entry is rejected by `get()` and removed from storage.
4. A malformed entry (bad JSON, wrong `version`, missing fields) fails safely (`get()` returns `null`, entry removed) rather than throwing.
5. `set()` writes include both `savedAt` and `version`.
6. `invalidate()` removes only the targeted `(clientId, memberId, resource)` entry, leaving sibling entries (other resources, other members, other clients) intact.
7. `clearAll()` removes only `relativity:portal:v1:*` keys, leaving unrelated `sessionStorage` keys untouched.
8. `staleWhileRevalidate()` invokes the render callback with cached data synchronously before the network call resolves.
9. `staleWhileRevalidate()` invokes the render callback again with fresh data once the network call resolves (when different from cached).
10. `staleWhileRevalidate()` keeps showing cached data (no error callback) when the background fetch rejects but a valid cache entry existed.
11. `staleWhileRevalidate()` invokes the error callback when the fetch rejects and no valid cache existed (preserves today's no-cache error behavior).

**In-flight deduplication tests** (`test/portalCache.test.js`, same file):

12. Two, and separately three, simultaneous `dedupedFetch()` calls for the same key run `fetchFn` exactly once; all callers receive the identical result.
13. Calls keyed by a different resource, a different `clientId`, or a different `memberId` are never merged — each runs its own `fetchFn`.
14. The in-flight entry is removed after both success and failure, so the next call for that key (even immediately after) always starts a genuinely new request.
15. `staleWhileRevalidate()`'s existing cache-first/fresh-replace/failure-preservation behavior is unchanged when two callers share a deduped request (verified by simulating `loadMembers()` and `loadTeamMembers()` racing for the same `teamMembers` key).

**Acceptance criteria:**

- Collections, Documents, and Team Members each appear immediately (no skeleton) on a page refresh when a valid cache entry exists, then silently update if the server response differs.
- Fresh data still loads in the background on every page load/tab open, even when cache is used for the immediate render.
- Every mutation listed above leaves no stale cached list visible on the next load of that resource.
- Logout, and the invalid-membership bootstrap redirect, both clear all `relativity:portal:*` entries and nothing else.
- No tokens, OAuth credentials, document contents, or chat content are ever written to the cache.
- Cache entries are scoped per client and per member; no code path can read another tenant's or another member's cached data.
- `npm test` passes in full.

## In-Flight Request Deduplication

Follow-up to the original milestone, addressing the duplicate-request limitation flagged above: `portalCache.js` now also tracks in-flight fetches in a module-level `Map` (`inFlightRequests`, in-memory only — never written to `sessionStorage`), keyed by the exact same tenant/member/resource-scoped string `buildKey()` already produces for the `sessionStorage` cache entry.

`PortalCache.dedupedFetch(key, fetchFn)` is the mechanism: the first call for a given key runs `fetchFn` and stores the resulting Promise; any additional call for that same key made before it settles gets that identical Promise back (one network request, all callers receive the same result) instead of starting its own. The entry is removed as soon as the Promise settles — success or failure — via `.finally()`, so a later call (even immediately after) always starts a genuinely new request rather than reusing a stale or failed one. `staleWhileRevalidate()` calls `dedupedFetch()` internally for its background fetch, so every resource already wired through the cache gets this for free with no call-site changes.

This only deduplicates calls that go through `staleWhileRevalidate()` with the *same* `(clientId, memberId, resource)` key — it does not, and structurally cannot, merge requests across different clients, members, or resource names (see Testing below).

**Where this actually applies today**, per an audit of every `resource:` literal passed to `staleWhileRevalidate()` in `portal.js`:

| Resource | SWR call sites | Dedup benefit today |
|---|---|---|
| `teamMembers` | `loadMembers()` (onboarding progress) and `initTeamSection()`'s `loadTeamMembers()` (the Team tab table) | **Real, exercised today.** Both fire at bootstrap for owner/admin members (`initTeamSection(); loadMembers();`) within the same tick, both hit `GET /api/team/members` and return the identical shape — they now share one request instead of firing two. |
| `documents` | `loadDocuments()` only | None today (only one call site) — the mechanism is in place and correct if a second concurrent caller is ever added. |
| `collections` | `loadCollectionsTable()` only | None today — see below. |

**Collections' other two `GET /api/collections` call sites remain intentionally outside the cache/dedup system**: `loadSlackAllowedCollections()` and `loadKbCollectionsFilter()` still issue plain, undeduped `fetch()` calls. This is *not* a response-shape mismatch — all three call sites request and destructure the exact same `{ collections: [...] }` payload — it is a deliberate scope boundary carried over from the original milestone (see Non-Goals below): pulling those two call sites onto `staleWhileRevalidate()` would mean giving them cache-driven render callbacks and TTL handling they don't currently have, which is a broader refactor than "add dedup to the existing cached path." Only `loadCollectionsTable()`, the one call site already wired through the cache, participates in dedup today.

## Non-Goals (out of scope for this milestone)

- ~~Deduplicating the three separate `GET /api/collections` call sites, or the two separate `GET /api/team/members` call sites, into a single in-flight request.~~ **Partially addressed** by the in-flight deduplication mechanism above: the two `teamMembers` call sites now share a request. The two additional `collections` call sites (`loadSlackAllowedCollections`, `loadKbCollectionsFilter`) remain separate, plain fetches, not wired onto `staleWhileRevalidate()` at all — deliberately, to avoid a broader refactor of loaders that don't otherwise use the cache.
- Caching ingestion jobs, analytics, import history, or chat/session data — all either change too fast to benefit or are explicitly excluded as sensitive/content-bearing.
- Any server-side or cross-device caching layer.

## Reported Findings (from inspection, not fixed by this milestone)

- `GET /api/collections` is still fetched from three independent call sites; only one (`loadCollectionsTable`) goes through the cache/dedup path. The other two (`loadSlackAllowedCollections`, `loadKbCollectionsFilter`) request the identical `{collections}` shape but remain plain fetches — a reasonable follow-up if it's ever worth bringing them onto the same cached/deduped path.
- Document upload/delete/move flows do not currently invalidate any collections-related state client-side (collection document counts could go stale in-memory during a long-lived tab even without this cache); this plan's mutation table now covers that going forward for the cached path, but the underlying `loadedCollections` in-memory variable itself has the same pre-existing staleness property independent of this cache.
