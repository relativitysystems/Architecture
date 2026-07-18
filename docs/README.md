# Platform Documentation — Index

This `docs/` tree is the audit-driven counterpart to the narrative documentation in [`../architecture/`](../architecture/), [`../decisions/`](../decisions/), [`../product/`](../product/), and [`../roadmap/`](../roadmap/). Where the two overlap, `docs/` is the one whose claims have been checked directly against live Supabase state and repository file trees in this pass — see each page's verification labels (**Verified / Code Verified / Database Verified / Migration Verified / Historical / Inference / Unknown**).

## Master audit status

This documentation set is built in phases. Status as of 2026-07-18:

| Phase | Scope | Status |
|---|---|---|
| 1 | Repository discovery | ✅ [repositories/](repositories/) |
| 2 | Global Supabase audit | ✅ [supabase/global/](supabase/global/) |
| 3 | AIKB Supabase audit | ✅ [supabase/aikb/](supabase/aikb/) |
| 4 | Cross-database data model | ⏳ not yet written — see [supabase/global/TABLES.md](supabase/global/TABLES.md) + [supabase/aikb/TABLES.md](supabase/aikb/TABLES.md) for the per-table detail in the meantime |
| 5 | Cross-repository ownership matrix | ✅ [repositories/CODE_OWNERSHIP.md](repositories/CODE_OWNERSHIP.md) |
| 6 | Data flow analysis (auth, OAuth, ingestion, chat, Slack, deletion, etc.) | ⏳ not yet written — existing [../architecture/SYSTEM_OVERVIEW.md](../architecture/SYSTEM_OVERVIEW.md) covers several flows at a narrative level (Historical, not independently re-verified) |
| 7 | Cross-database architecture + diagrams | ⏳ not yet written |
| 8 | Security audit (full, classified) | ⏳ partial — see [supabase/global/SECURITY.md](supabase/global/SECURITY.md) / [supabase/aikb/SECURITY.md](supabase/aikb/SECURITY.md) for the database-level layer; [../architecture/SECURITY.md](../architecture/SECURITY.md) for the app-level layer (Historical) |
| 9 | Migration/schema-drift audit | ✅ [audits/SCHEMA_DRIFT.md](audits/SCHEMA_DRIFT.md) |
| 10 | Legacy audit | ✅ folded into [audits/SCHEMA_DRIFT.md](audits/SCHEMA_DRIFT.md) (every drift finding is also a legacy-cleanup candidate) |
| 11 | Performance audit | ⏳ partial — see INDEXES.md in both [supabase/global/](supabase/global/INDEXES.md) and [supabase/aikb/](supabase/aikb/INDEXES.md) |
| 12 | Mermaid diagrams | ⏳ not yet written |
| 13 | Full docs/ tree structure | ⏳ partial — this index plus repositories/, supabase/, audits/ exist; database/, architecture/, integrations/, security/, flows/, diagrams/, migrations/ subtrees are not yet populated |
| 14 | Executive master audit | ⏳ not yet written |

## Navigation

- [repositories/](repositories/) — what each repo contains, verified against its actual file tree
- [supabase/global/](supabase/global/) — the Relativity-owned Supabase project, table-by-table
- [supabase/aikb/](supabase/aikb/) — the AIKB-owned Supabase project, table-by-table
- [audits/SCHEMA_DRIFT.md](audits/SCHEMA_DRIFT.md) — every undocumented-in-migrations live object, with classification and evidence

## Pre-existing documentation (treat as Historical unless cross-linked as re-verified)

- [../architecture/](../architecture/) — `SYSTEM_OVERVIEW.md`, `AIKB.md`, `SECURITY.md`, `CONNECTOR_FRAMEWORK.md`, `INGESTION_PIPELINE.md`, `SERVICE_CONTRACTS.md`
- [../decisions/](../decisions/) — ADR-001 through ADR-006
- [../product/](../product/), [../roadmap/](../roadmap/), [../vision/](../vision/), [../history/](../history/) — product, roadmap, vision, and how-we-got-here documentation, not re-audited in this pass
