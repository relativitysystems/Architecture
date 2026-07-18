# Architecture Review Report

This report has been reorganized into maintainable architecture documentation. Every section of the original report has a destination below — current architecture, accepted decisions, remaining work, and historical findings are now kept as separate, purpose-specific documents instead of one large audit report.

## Current architecture

- [System Overview](architecture/SYSTEM_OVERVIEW.md) — start here
- [AIKB](architecture/AIKB.md)
- [Ingestion Pipeline](architecture/INGESTION_PIPELINE.md)
- [Connector Framework](architecture/CONNECTOR_FRAMEWORK.md)
- [Service Contracts](architecture/SERVICE_CONTRACTS.md)
- [Security](architecture/SECURITY.md)

## Product

- [Client Portal](product/CLIENT_PORTAL.md)
- [Knowledge Analytics](product/KNOWLEDGE_ANALYTICS.md)
- [Knowledge Gap Detection](product/KNOWLEDGE_GAP_DETECTION.md)
- [AI Agents](product/AI_AGENTS.md)

## Architecture decisions

- [Architecture Decision Records](decisions/)

## Roadmap

- [Master Roadmap](roadmap/MASTER_ROADMAP.md)
- [Feature Backlog](roadmap/FEATURE_BACKLOG.md)
- [Connector Roadmap](roadmap/CONNECTOR_ROADMAP.md)

## Historical report

- [Architecture Review History](history/ARCHITECTURE_REVIEW_PHASES.md) — preserves the original review's phases (discovery, platform architecture, domain model, Slack MVP implementation), its findings as originally written, and the milestone-by-milestone implementation record (test counts, migration filenames, files changed), with a current-status table showing how each original finding was resolved. Read this for the reasoning and history behind the current architecture; read the documents above for what is true today.

See [README.md](README.md) for the full documentation map.
