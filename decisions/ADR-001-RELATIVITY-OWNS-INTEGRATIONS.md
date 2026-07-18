# ADR-001: Relativity owns all external integrations

## Status
Accepted

## Date
Undated (decision made during the original architecture discovery; formalized and implemented starting 2026-07-14)

## Context

The platform has two repositories: Relativity (customer-facing gateway) and AIKB (knowledge engine). Every external provider integration — Slack, Google Drive, Dropbox, and future connectors (Teams, Gmail, Outlook, CRM) — needs OAuth, webhook handling, provider API calls, and tenant mapping. Early in the platform's history, AIKB had grown its own, disconnected Slack Events handler (`aikb/routes/slack.js`) independent of Relativity's Slack OAuth flow — using a single static global bot token and deriving a fake `clientId` by hashing the Slack channel ID, with no real tenant lookup. This produced two incompatible, half-built Slack integrations in two different repositories, neither safe to build on as-is.

A decision was required: should provider integrations (OAuth, webhooks, provider API calls, tenant mapping, credential storage) live in Relativity or AIKB, and should this be true for every current and future connector?

## Decision

**Relativity owns every external integration, end to end:**

- OAuth authorization flows and token exchange
- Webhook/event ingestion and signature verification
- Provider API calls (Slack Web API, Google Drive API, Dropbox API)
- Provider account / tenant mapping (which external account maps to which `client_id`)
- Credential storage (`oauth_connections`/`oauth_credentials`)
- Customer-facing integration UI (the portal's Integrations panel)

**AIKB remains provider-agnostic.** It never contains provider-specific logic, never receives or stores a provider credential, and never branches its request handling on which provider originated a request beyond tagging `origin` metadata (`portal`, `slack`, etc.).

## Alternatives Considered

- **Split by convenience** (leave Slack Events in AIKB since a signature-verified handler already existed there): rejected — this is exactly the arrangement that produced the disconnected, unsafe dual-Slack-integration problem in the first place. It also means every future connector (Teams, Gmail) would need its own bespoke decision about which repo owns which half, rather than one consistent rule.
- **AIKB owns tenant mapping since it owns retrieval**: rejected — tenant/identity mapping is an identity-and-tenancy concern, and Relativity is already the sole owner of `clients`/`client_members`. Giving AIKB even a partial identity table (a channel→client hash) is what caused the original problem.

## Consequences

- Every new connector (Teams, Gmail, Outlook, CRM, meeting transcripts, storage providers) follows one consistent pattern: implement `start`/`callback` OAuth routes, verify the provider's webhook signature, resolve tenant via a trusted server-side lookup, then call AIKB's existing knowledge pipeline. See [../architecture/CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md).
- AIKB stays a smaller, more reusable service — it can be called by any number of future surfaces without ever being rewritten for a new provider.
- Relativity's surface area grows with each new connector (more routes, more provider-specific code), which is the accepted tradeoff for keeping AIKB provider-agnostic.

## Implementation Evidence

- Slack: OAuth (`/api/integrations/slack/start`/`callback`), Events (`/api/integrations/slack/events`), delivery (`/api/integrations/slack/deliver`) all live in Relativity. AIKB's legacy `routes/slack.js` is retired to a `410 Gone` stub with no tenant-mapping or retrieval logic remaining (Milestone 1, deployed before Milestone 4).
- Google Drive and Dropbox OAuth and API calls live entirely in Relativity (`routes/auth.js`, `services/googleDriveService.js`, `services/dropboxService.js`); AIKB contains no working Google Drive/Dropbox code (its own `services/googleDriveService.js` is dead/orphaned).
- The `oauth_connections.provider` CHECK constraint already anticipates `slack`, `microsoft`, `gmail`, `google_drive`, `dropbox` — the schema is provider-generic in Relativity, not AIKB.

## Related Documents

- [../architecture/SYSTEM_OVERVIEW.md](../architecture/SYSTEM_OVERVIEW.md)
- [../architecture/CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md)
- [ADR-003-SLACK-EVENTS-LIVE-IN-RELATIVITY.md](ADR-003-SLACK-EVENTS-LIVE-IN-RELATIVITY.md)
- [../roadmap/CONNECTOR_ROADMAP.md](../roadmap/CONNECTOR_ROADMAP.md)
- [../history/ARCHITECTURE_REVIEW_PHASES.md](../history/ARCHITECTURE_REVIEW_PHASES.md)
