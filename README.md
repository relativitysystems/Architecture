# Relativity Systems — Architecture Repository

## Overview

This repository is the canonical source of technical documentation for Relativity Systems. It documents the **actual, verified implementation** of the platform's two production codebases — `relativitysystems/AIKB` (the knowledge engine) and `relativitysystems/Relativity` (the client-facing gateway, portal, and integration layer) — plus the product-level and roadmap context that sits on top of them.

This repository contains **no code**. It is a documentation-only project; nothing here is executed, imported, or deployed. Its purpose is to give any engineer, without reading both codebases end to end, an accurate picture of what exists today, how it works, where its limitations are, and where deliberate extension points already exist for future work.

## Current Implementation

As of this review, the documentation set covers:

- The full AIKB architecture: multi-tenant design, database schema, knowledge collections, embedding pipeline, chunking, vector search, retrieval, security model, and citation generation.
- The complete document ingestion pipeline, from upload through background job processing to indexed, searchable chunks.
- The connector framework underlying Google Drive, Dropbox, and Slack, with Slack documented as the reference implementation for how a future connector (Gmail, Outlook, Teams, CRM) should be built.
- The cross-repository security model: authentication mechanisms, tenant isolation, secrets management, and currently-known risks.
- The client portal product surface: authentication, knowledge management, chat, collections, team management, and integrations.
- What currently exists for analytics and knowledge-gap detection, explicitly separated from what is planned but not built.
- What currently exists in the way of AI capabilities (RAG chat — not autonomous agents), and how that architecture could evolve.
- A technical backlog derived directly from evidence found in both codebases (dead code, documented gaps, and existing extension points).

## Documentation Structure

```
Architecture/
├── README.md                          — this file
├── architecture/
│   ├── AIKB.md                        — AIKB engine: schema, retrieval, security, citations
│   ├── INGESTION_PIPELINE.md          — document upload → chunk → embed → index, end to end
│   ├── CONNECTOR_FRAMEWORK.md         — Slack, Google Drive, Dropbox; pattern for future connectors
│   └── SECURITY.md                    — auth, tenant isolation, secrets, current risks
├── product/
│   ├── CLIENT_PORTAL.md               — portal UX and feature surface
│   ├── KNOWLEDGE_ANALYTICS.md         — what's measured today vs. planned
│   ├── KNOWLEDGE_GAP_DETECTION.md     — current heuristic detection vs. a full gap-review system
│   └── AI_AGENTS.md                   — current RAG chat vs. a future agentic roadmap
├── roadmap/
│   ├── FEATURE_BACKLOG.md             — evidence-based technical backlog, prioritized
│   ├── CONNECTOR_ROADMAP.md           — (not yet populated by this review)
│   └── MASTER_ROADMAP.md              — (not yet populated by this review)
├── vision/
│   ├── COMPANY_MISSION.md             — (not yet populated by this review)
│   ├── FOUNDING_PRINCIPLES.md         — (not yet populated by this review)
│   └── PRODUCT_VISION.md              — (not yet populated by this review)
└── ideas/
    └── IDEAS_VAULT.md                 — (not yet populated by this review)
```

## How Developers Should Use This Repository

- **Before making a backend change**, read the relevant `architecture/` document first. `AIKB.md` and `INGESTION_PIPELINE.md` cover the knowledge engine; `CONNECTOR_FRAMEWORK.md` covers anything touching Slack, Google Drive, Dropbox, or a new integration; `SECURITY.md` covers auth and tenant isolation and should be read before touching any auth-adjacent code.
- **Before building a new integration**, read `CONNECTOR_FRAMEWORK.md` in full — it documents the pattern Slack already established (encrypted OAuth, signed service requests, fail-closed collection scoping) so new connectors are consistent rather than reinventing tenant resolution or retrieval logic per channel.
- **Before starting new product work**, check `roadmap/FEATURE_BACKLOG.md` — it is derived directly from dead code, documented TODOs, and known gaps found in the current codebases, and should be checked to avoid duplicating already-identified work.
- **Every document distinguishes current implementation from future work.** Content under a "Future Roadmap" or "Future Extension Points" heading is **not implemented** — treat it as design intent, not as a description of running code. If a document doesn't explicitly say a capability exists, verify against the codebase before relying on it; this documentation set reflects the state of both repositories at the time it was written and will drift as the code changes.
- **Verify before you build.** Every claim in the `architecture/` and `product/` documents is sourced from a specific file and, where practical, a line reference in one of the two source repositories. If you find a discrepancy between this documentation and the code, the code is authoritative — please update the relevant document in the same change.

## Folder Descriptions

| Folder | Contents |
|---|---|
| `architecture/` | System-level technical architecture of the AIKB engine and the Relativity gateway: data model, pipelines, integration patterns, and security. Written for engineers making backend or infrastructure changes. |
| `product/` | Product- and feature-level documentation of user-facing capability: the client portal, analytics, knowledge-gap handling, and AI capabilities. Written for anyone scoping product work, not just backend changes. |
| `roadmap/` | Prioritized, evidence-based backlog and longer-range roadmap documents. `FEATURE_BACKLOG.md` is populated by this review; `CONNECTOR_ROADMAP.md` and `MASTER_ROADMAP.md` exist as stubs for future population. |
| `vision/` | Company mission, founding principles, and product vision. These are stubs today — outside the scope of a codebase-verification review, since they describe intent rather than implementation. |
| `ideas/` | An open idea vault for unvetted future concepts. A stub today, intentionally not populated by this review since its content is by definition not yet evidenced in either codebase. |

## Current Limitations

- `vision/`, `ideas/`, and two of the three `roadmap/` documents remain empty stubs — this review populated only the documents explicitly in scope (the four `architecture/` documents, four `product/` documents, and `roadmap/FEATURE_BACKLOG.md`).
- Documentation here reflects a point-in-time review of both source repositories. Neither repository is version-pinned against this documentation set, so drift is possible, especially in the connector and security areas, which were the most recently changed at the time of writing.
- There is no automated check tying this repository's claims back to the source repositories — accuracy currently depends on manual review at documentation-update time.

## Future Extension Points

- Populate `vision/` and `ideas/` once company-level (not codebase-level) source material is available to verify against.
- Populate `roadmap/CONNECTOR_ROADMAP.md` and `roadmap/MASTER_ROADMAP.md` as longer-range planning decisions are made on top of the current-state documentation here.
- Consider a lightweight process (e.g., a documentation-review step alongside major PRs to `AIKB` or `Relativity`) to keep the `architecture/` documents from drifting out of sync with the code they describe.
