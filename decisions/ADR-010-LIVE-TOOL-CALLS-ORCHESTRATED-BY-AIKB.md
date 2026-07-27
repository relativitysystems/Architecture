# ADR-010: Live external-data tool calls are orchestrated by AIKB, executed by Relativity, and never ingested

## Status
Proposed — precedes any implementation. This ADR exists to lock the invariants before CRM (or any other live-query connector) is built, per the same reasoning [ADR-004](ADR-004-SIGNED-SERVICE-REQUESTS.md) and [ADR-005](ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md) were written before their respective features shipped.

## Date
2026-07-27 (proposal)

## Context

Every connector built so far — Slack, Google Drive, Gmail — follows one shape: Relativity owns auth and the provider API call, but the *data* always ends up ingested into AIKB (`knowledge_documents`/`knowledge_chunks`) before it can be asked about. [AI_AGENTS.md](../product/AI_AGENTS.md) documents why that shape exists: AIKB's answer pipeline (`runKnowledgeQuery`) is a single-shot RAG loop with **no tool/function calling anywhere in either codebase** — the model only ever answers from chunks a prior, code-driven retrieval step already fetched. There is no path today for a live, uningested external lookup to reach that pipeline.

A CRM integration is being scoped that deliberately does not fit this shape: CRM data (deal stage, recent notes, contact status) is high-churn and privacy-sensitive enough that ingesting a copy into AIKB's vector store is undesirable — the product requirement is "look this up live, at question time" rather than "keep a synchronized copy." This is not an incremental extension of the existing connector pattern; it is the platform's first instance of tool-calling, and [AI_AGENTS.md](../product/AI_AGENTS.md) already flags that direction as "a significant architectural addition, not an incremental change." More live-query connectors (calendars, ticketing systems, a second CRM) are expected to follow the same shape once one exists, so the invariants need to be decided once, consistently, before any of them are built — the same reasoning [ADR-001](ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md) gives for why every connector should follow one rule rather than each making its own repo-split decision.

Two existing decisions constrain the design and must both still hold:
- [ADR-001](ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md): Relativity owns every external integration end to end — OAuth, credentials, provider API calls. AIKB never holds a provider credential and never branches its behavior on which provider is involved.
- [ADR-002](ADR-002-AIKB-OWNS-KNOWLEDGE-PROCESSING.md): AIKB owns knowledge processing end to end — retrieval, answer generation, citations, conversation state. Relativity never reimplements answer generation; every question-asking surface calls the same shared pipeline.

A live tool call sits exactly on the seam between these two ADRs — the credential and the API call belong on one side, the decision to call it and the resulting answer belong on the other. Getting this seam wrong reproduces the exact failure mode both ADRs already exist to prevent: two places that know how to answer a question, or a component that reaches past its owned boundary (AIKB's original Slack prototype did the latter — ran its own retrieval and derived its own tenant mapping — and was retired specifically for it).

## Decision

**AIKB decides whether to call a tool and generates the answer from the result. Relativity is the only thing that ever holds a credential or calls a provider API. Nothing a tool call returns is persisted as knowledge.**

Concretely:

1. **Orchestration ownership stays with AIKB, consistent with ADR-002.** The decision of *whether* a question needs a live lookup, *which* tool to call, and how to fold the result into an answer happens inside AIKB's existing answer-generation step (`runKnowledgeQuery`), extended with OpenAI function/tool-calling. This is the first use of `tools`/`tool_calls` in either codebase — it is a new capability, not a repurposing of the existing fixed retrieval-then-generate flow. No second, Relativity-side answer-generation path is created for this or any future live-query connector.

2. **Credentials and provider execution stay with Relativity, consistent with ADR-001.** AIKB never decrypts an OAuth credential and never calls a CRM (or any provider) API directly. When AIKB's model calls a tool, AIKB makes a callback to a single, generic, provider-agnostic Relativity endpoint (working name: `POST /api/tools/execute`) carrying only a bounded tool name, its declared parameters, and the resolved `clientId` — never a provider name. Relativity resolves which provider backs that tool for that client (via the client's active `oauth_connections` row, exactly as every existing connector resolves tenant identity), decrypts the credential, executes the provider-specific call, and returns a structured, provider-agnostic result. **AIKB never learns which CRM product answered the call**, the same way it never learns today whether a question came from Slack or the portal beyond an `origin` tag.

3. **The AIKB → Relativity tool-call boundary reuses the existing signed-envelope mechanism, not a new one.** This callback is authenticated exactly like the existing reversed Slack `/deliver` callback described in [ADR-004](ADR-004-SIGNED-SERVICE-REQUESTS.md) (AIKB signs, Relativity verifies via `requireServiceRequest`) — a new signing scheme is explicitly not warranted here; the direction and shape already exist.

4. **Authorization is fail-closed and resolved the same way every other connector resolves it.** A tool is only callable for a client if an active `oauth_connections` row for the relevant provider exists; a missing/revoked connection means the tool is unavailable for that call, never silently skipped in a way that could look like "the CRM has no data" versus "we aren't connected." This mirrors the fail-closed default [ADR-005](ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md) established for Slack's collection scoping — absence of an explicit grant means "cannot access," not "unrestricted" or "silently ignore."

5. **Nothing a tool call returns is ingested.** No `knowledge_documents`/`knowledge_chunks` rows, no embedding step, no collection assignment — this is explicitly outside the ingestion ownership [ADR-002](ADR-002-AIKB-OWNS-KNOWLEDGE-PROCESSING.md) grants AIKB over *stored* knowledge. A tool result is request-scoped context for exactly one generation call, discarded after. The generated answer text is persisted to `knowledge_chat_messages` exactly as any other answer is today (per ADR-002) — what's new is that the raw tool-call request/response is not itself a durable artifact anywhere. Whether raw tool-call payloads should be logged for debugging/audit (separately from knowledge persistence) is an open question, deliberately left to implementation-level design rather than decided here.

6. **The tool surface is a small, fixed, named set — never a freeform query the model constructs against a live, credentialed API.** Tools look like `search_crm_contacts(name)` or `get_recent_notes(contact_id)`, each with a declared parameter schema Relativity validates before execution. An LLM given the ability to construct arbitrary provider API calls against a decrypted credential is a materially different risk (and reliability) profile than an LLM selecting among a bounded set of pre-defined, parameter-validated operations — this ADR permits only the latter.

7. **The tool-call loop is bounded to a small fixed maximum (recommended: one tool call per question) for now.** An open-ended, multi-step loop where the model repeatedly calls tools until it decides it's satisfied is explicitly out of scope of this decision — [AI_AGENTS.md](../product/AI_AGENTS.md) already lists multi-step reasoning loops as unbuilt future work, and combining "first tool-calling implementation" with "first open-ended agent loop" in one change is more architectural surface than this ADR is scoping. A future ADR should raise this bound deliberately, not by default.

## Alternatives Considered

- **Ingest CRM data like every other connector** (Slack/Gmail pattern): rejected for this use case — the product requirement is answering from live state, and periodic re-sync either goes stale between syncs or requires a sync cadence tight enough to erode most of the benefit over simply calling the CRM at question time. Nothing about this ADR forecloses building an ingestion-based CRM connector later for a different use case (e.g., historical deal analytics) — the two are not mutually exclusive, but this ADR governs the live-lookup shape only.
- **Relativity performs intent detection, the provider call, and answer generation itself, bypassing AIKB entirely**: rejected — this duplicates session handling, conversation history, citation formatting, and gap-detection logic that already exists once, in AIKB, per ADR-002. It is the exact fragmentation ADR-002 was written to prevent, just recreated for a new data source instead of a new surface.
- **AIKB calls the CRM provider directly, given it's already making the generation call anyway**: rejected — this requires AIKB to hold or receive a decrypted provider credential and contain provider-specific request logic, which is precisely what ADR-001 prohibits and precisely the boundary violation that produced AIKB's original, retired Slack prototype.
- **A new, purpose-built signing scheme for the AIKB → Relativity tool callback**: rejected — the reversed signed-envelope pattern already exists (Slack's `/deliver`, per ADR-004) and covers this exactly; a second scheme would be an unjustified duplicate of existing, verified infrastructure.
- **An open-ended, model-driven multi-step tool loop from the start**: deferred, not rejected outright — reasonable future direction per AI_AGENTS.md's own roadmap section, but a larger scope than justified by the CRM use case alone, and better decided in its own ADR once a bounded single-call version is proven.

## Consequences

- A reusable pattern now exists for every future live-query connector (a second CRM, calendars, ticketing systems): define a bounded tool name and parameter schema, add a Relativity-side executor resolved via that provider's `oauth_connections` row, and register the tool with AIKB's generation call. No new architectural decision is required per connector — this is the direct payoff the same way ADR-001/002 made every ingestion-based connector a "thin adapter," not a bespoke design.
- AIKB's `services/openaiService.js`/`runKnowledgeQuery.js` must gain function/tool-calling support — a genuinely new capability, not a configuration change, and the first place either codebase gives a model the ability to trigger a side-effecting call (even a read-only one) rather than only generate text.
- Relativity gains one new generic route family (`POST /api/tools/execute` or similar) rather than one bespoke route per tool — new tools are additions to a registry, not new endpoints.
- A new failure mode is introduced — tool-call timeout or provider unavailability at question-answering time — distinct from the existing AIKB-generation-failure and Slack-delivery-failure cases ADR-007 already distinguishes. This ADR does not specify the bounded-retry/fallback mechanics for it (that is implementation-level design, likely its own follow-up ADR mirroring ADR-007's), but establishes that it must degrade gracefully (e.g., answer from other available context, or say the lookup failed) rather than hang the request or silently omit the fact that live data was unavailable.
- The existing intent-classification step in AIKB's pipeline (`classifyQueryIntent`, per [AI_AGENTS.md](../product/AI_AGENTS.md)) is the natural place tool availability/selection is decided — this ADR assumes that step is extended, not that a second, parallel routing layer is built in Relativity to guess when a tool is relevant.
- CRM (and any other live-query connector) is explicitly exempted from [CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md)'s step 4/5 (default collection assignment, embedding/storage) — that document should be updated when this pattern is implemented to reference this ADR as the alternate shape for live-query sources, rather than reading as though every connector must ingest.

## Implementation Evidence

None yet — this ADR is Proposed and intentionally precedes implementation tasks, per the request that produced it.

## Related Documents

- [ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md](ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md)
- [ADR-002-AIKB-OWNS-KNOWLEDGE-PROCESSING.md](ADR-002-AIKB-OWNS-KNOWLEDGE-PROCESSING.md)
- [ADR-004-SIGNED-SERVICE-REQUESTS.md](ADR-004-SIGNED-SERVICE-REQUESTS.md)
- [ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md](ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md)
- [ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md](ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md)
- [../architecture/CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md)
- [../product/AI_AGENTS.md](../product/AI_AGENTS.md)
- [../roadmap/CONNECTOR_ROADMAP.md](../roadmap/CONNECTOR_ROADMAP.md)
