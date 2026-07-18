# ADR-002: AIKB owns all knowledge processing

## Status
Accepted

## Date
Undated (decision made during the original architecture discovery)

## Context

Complementary to [ADR-001](ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md): given that Relativity owns identity and integrations, the platform needed an equally clear rule for where ingestion, retrieval, reasoning, conversation state, and knowledge gaps live. AIKB's legacy Slack Events handler had violated this boundary from the other direction — it ran its own retrieval (`searchChunks`) directly, bypassing the shared query pipeline's intent classification, session/member context, and gap-detection logic entirely. This meant "AIKB owns knowledge processing" was true almost everywhere except the one place a second, informal knowledge path had grown.

## Decision

**AIKB owns knowledge processing end to end:**

- Document parsing, chunking, embedding generation
- Vector retrieval (`match_knowledge_chunks`)
- Answer generation and citations
- Knowledge collections (definition, assignment, retrieval enforcement)
- Chat sessions and messages (conversation content)
- Knowledge gaps (schema and persistence)
- Background/async processing (Inngest)

**Relativity never duplicates this logic.** Every surface that needs to ask a question — the portal today, Slack today, any future connector — calls into the same shared pipeline (`runKnowledgeQuery`) rather than implementing its own retrieval or answer generation.

## Alternatives Considered

- **Let each surface implement its own lightweight retrieval** (as AIKB's original Slack prototype did): rejected — this is precisely the failure mode that produced two incompatible answer-generation paths (portal's full pipeline with intent classification and gap detection, versus Slack's direct `searchChunks` call with none of that). Every surface-specific reimplementation multiplies the places a security or correctness fix has to be applied.
- **Give AIKB its own partial identity/tenant table** (to let it resolve Slack channels independently): rejected — this is the mirror-image mistake of not fully committing to [ADR-001](ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md); AIKB should receive a resolved `clientId`, not derive one itself.

## Consequences

- Every new surface (Teams, Gmail, a future Public API) is bounded to "write a thin adapter that calls the existing pipeline," never "extend or duplicate the knowledge pipeline." This is the direct payoff realized when Slack's Milestone 4 work refactored `/query`'s handler body into a shared `runKnowledgeQuery` function called by both `/query` (`origin: 'portal'`) and `/ask` (`origin: 'slack'`), rather than writing a second retrieval implementation for Slack.
- AIKB's API surface must be broad enough to serve every current and future surface's needs (collection filtering, origin tagging, idempotency) without those surfaces needing their own logic — this is why `origin`/`originMetadata`/`idempotencyKey` columns exist on AIKB's tables even before every consumer of them is built.
- A bug or improvement in retrieval/generation/gap-detection logic is fixed once, in AIKB, and every surface benefits immediately.

## Implementation Evidence

- `services/runKnowledgeQuery.js` (AIKB) is called identically by `/query` (portal) and the Slack Inngest function (`knowledge/slack.question.requested`) — same intent classification, retrieval, generation, and gap-detection logic, verified by the pre-existing `/query` regression suite passing unmodified after the refactor.
- AIKB's legacy `routes/slack.js` — the one place this boundary was violated — is retired to a `410 Gone` stub with all retrieval logic (`searchChunks`, the channel-hash tenant derivation) deleted, not merely disabled.
- Knowledge collections (`knowledge_collections` table, migration `006_knowledge_collections.sql`) are defined, assigned, and enforced entirely inside AIKB.

## Related Documents

- [../architecture/SYSTEM_OVERVIEW.md](../architecture/SYSTEM_OVERVIEW.md)
- [../architecture/AIKB.md](../architecture/AIKB.md)
- [ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md](ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md)
- [ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md](ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md)
- [../history/ARCHITECTURE_REVIEW_PHASES.md](../history/ARCHITECTURE_REVIEW_PHASES.md)
