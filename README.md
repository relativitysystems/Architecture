# Relativity Systems — Architecture Repository

## Overview

This repository is the canonical source of technical documentation for Relativity Systems. It documents the **actual, verified implementation** of the platform's two production codebases — `relativitysystems/AIKB` (the knowledge engine) and `relativitysystems/Relativity` (the client-facing gateway, portal, and integration layer) — the architecture decisions behind that implementation, the roadmap on top of it, and the historical review that produced all of it.

This repository contains **no code**. It is a documentation-only project; nothing here is executed, imported, or deployed. Its purpose is to give any engineer, without reading both codebases end to end, an accurate picture of what exists today, how it works, where its limitations are, and where deliberate extension points already exist for future work.

## Start Here

New to the platform? Read in this order:

1. **[architecture/SYSTEM_OVERVIEW.md](architecture/SYSTEM_OVERVIEW.md)** — the primary technical entry point: repository maps, responsibility matrix, data ownership, and the platform's request flows.
2. **[decisions/](decisions/)** — the architecture decisions already made and why.
3. **[roadmap/MASTER_ROADMAP.md](roadmap/MASTER_ROADMAP.md)** — where the platform is headed next.
4. **[history/ARCHITECTURE_REVIEW_PHASES.md](history/ARCHITECTURE_REVIEW_PHASES.md)** — how the platform got here, for context on *why*, not just *what*.

## Documentation Structure

```
Architecture/
├── README.md                          — this file
├── architecture/
│   ├── SYSTEM_OVERVIEW.md             — primary entry point: repo maps, ownership, request flows
│   ├── AIKB.md                        — AIKB engine: schema, retrieval, security, citations
│   ├── INGESTION_PIPELINE.md          — document upload → chunk → embed → index, end to end
│   ├── CONNECTOR_FRAMEWORK.md         — Slack, Google Drive, Dropbox; pattern for future connectors
│   ├── SERVICE_CONTRACTS.md           — the Relativity ↔ AIKB request/response contracts
│   └── SECURITY.md                    — auth, tenant isolation, secrets, current risks
├── product/
│   ├── CLIENT_PORTAL.md               — portal UX and feature surface
│   ├── CLIENT_ONBOARDING.md           — account provisioning and initial data migration
│   ├── KNOWLEDGE_ANALYTICS.md         — what's measured today vs. planned
│   ├── KNOWLEDGE_GAP_DETECTION.md     — current heuristic detection vs. a full gap-review system
│   └── AI_AGENTS.md                   — current RAG chat vs. a future agentic roadmap
├── decisions/                         — architecture decision records (ADRs)
│   ├── ADR-001-RELATIVITY-OWNS-INTEGRATIONS.md
│   ├── ADR-002-AIKB-OWNS-KNOWLEDGE-PROCESSING.md
│   ├── ADR-003-SLACK-EVENTS-LIVE-IN-RELATIVITY.md
│   ├── ADR-004-SIGNED-SERVICE-REQUESTS.md
│   ├── ADR-005-COLLECTION-FILTERING-FAILS-CLOSED.md
│   ├── ADR-006-OAUTH-CREDENTIAL-ENCRYPTION.md
│   └── ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md
├── roadmap/
│   ├── MASTER_ROADMAP.md              — high-level architecture/product sequence
│   ├── FEATURE_BACKLOG.md             — evidence-based technical backlog, prioritized
│   └── CONNECTOR_ROADMAP.md           — connector-by-connector implementation status
├── history/
│   └── ARCHITECTURE_REVIEW_PHASES.md  — the original review's phases, findings, and milestones
├── go-to-market/
│   └── DEMO_VIDEO_STRATEGY.md         — demo video narrative strategy, storyline, and script framework
├── vision/
│   ├── COMPANY_MISSION.md             — (not yet populated — no source material in scope)
│   ├── FOUNDING_PRINCIPLES.md         — (not yet populated — no source material in scope)
│   └── PRODUCT_VISION.md              — (not yet populated — no source material in scope)
└── ideas/
    └── IDEAS_VAULT.md                 — (not yet populated — no source material in scope)
```

## How Developers Should Use This Repository

- **Before making a backend change**, read the relevant `architecture/` document first. `SYSTEM_OVERVIEW.md` orients you; `AIKB.md` and `INGESTION_PIPELINE.md` cover the knowledge engine; `CONNECTOR_FRAMEWORK.md` covers anything touching Slack, Google Drive, Dropbox, or a new integration; `SERVICE_CONTRACTS.md` covers the cross-repository API contract; `SECURITY.md` covers auth and tenant isolation and should be read before touching any auth-adjacent code.
- **Before building a new integration**, read `CONNECTOR_FRAMEWORK.md` in full and check `roadmap/CONNECTOR_ROADMAP.md` for that connector's current status — the pattern Slack already established (encrypted OAuth, a signed service-request envelope for non-human callers, fail-closed collection scoping) is what new connectors should be consistent with.
- **Before making a significant architectural change**, check `decisions/` for an existing ADR covering the area — and add a new one if you're making a decision of similar weight.
- **Before starting new product work**, check `roadmap/FEATURE_BACKLOG.md` and `roadmap/MASTER_ROADMAP.md` — they are derived directly from dead code, documented TODOs, and known gaps found in the current codebases, and should be checked to avoid duplicating already-identified work.
- **Every document distinguishes current implementation from future work.** Content under a "Future Roadmap," "Future Extension Points," or "Proposed, Not Implemented" heading is **not implemented** — treat it as design intent, not as a description of running code. If a document doesn't explicitly say a capability exists, verify against the codebase before relying on it; this documentation set reflects the state of both repositories at the time it was written and will drift as the code changes.
- **Verify before you build.** Every claim in the `architecture/` and `product/` documents is sourced from a specific file and, where practical, a line reference in one of the two source repositories. If you find a discrepancy between this documentation and the code, the code is authoritative — please update the relevant document in the same change.

## Folder Descriptions

| Folder | Contents |
|---|---|
| `architecture/` | System-level technical architecture of the AIKB engine and the Relativity gateway: data model, pipelines, integration patterns, service contracts, and security. Written for engineers making backend or infrastructure changes. |
| `product/` | Product- and feature-level documentation of user-facing capability: the client portal, client onboarding and data migration, analytics, knowledge-gap handling, and AI capabilities. Written for anyone scoping product work, not just backend changes. |
| `decisions/` | Architecture decision records — what was decided, why, what alternatives were considered, and what evidence shows it was actually implemented (or not). |
| `roadmap/` | Prioritized, evidence-based backlog (`FEATURE_BACKLOG.md`), connector sequencing (`CONNECTOR_ROADMAP.md`), and the high-level sequence tying them together (`MASTER_ROADMAP.md`). |
| `history/` | The original architecture review's phases, findings, and milestone-by-milestone implementation record — preserved so historical reasoning isn't lost, clearly labeled so it isn't mistaken for current architecture. |
| `go-to-market/` | Go-to-market content: the demo video's narrative strategy, fictional-company storyline, feature-by-feature script framework, and production checklists. A business/content deliverable, not architecture — see `roadmap/MASTER_ROADMAP.md` for its milestone-level status and acceptance criteria. |
| `vision/` | Company mission, founding principles, and product vision. These are stubs today — no company-level source material was in scope to populate them from. |
| `ideas/` | An open idea vault for unvetted future concepts. A stub today, intentionally not populated since its content is by definition not yet evidenced in either codebase. |

## Current Limitations

- `vision/` and `ideas/` remain empty stubs — no company-level (non-codebase) source material has been in scope to populate them from.
- Documentation here reflects a point-in-time review of both source repositories, most recently updated through the [ADR-007](decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md) bounded-Slack-delivery-retry implementation (superseding the earlier Milestone 4 production-deployment snapshot). Neither repository is version-pinned against this documentation set, so drift is possible, especially in the connector and security areas, which change most frequently.
- There is no automated check tying this repository's claims back to the source repositories — accuracy currently depends on manual review at documentation-update time.

## Future Extension Points

- Populate `vision/` and `ideas/` once company-level (not codebase-level) source material is available to verify against.
- Add new ADRs to `decisions/` as further significant architecture decisions are made (e.g., migrating Google Drive/Dropbox to encrypted credentials). The Slack delivery-retry sweep question is decided **and implemented** — see [ADR-007](decisions/ADR-007-SLACK-BOUNDED-DELIVERY-RETRY.md).
- Consider a lightweight process (e.g., a documentation-review step alongside major PRs to `AIKB` or `Relativity`) to keep the `architecture/` documents from drifting out of sync with the code they describe.
