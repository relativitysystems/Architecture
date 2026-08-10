# Quality Assurance

**Status: partial — index only.** This file was a stub. It now serves as the index of QA execution records that exist today. The comprehensive, repo-wide QA plan described in the root [README](../README.md) (test environments, regression policy, per-feature use-case libraries) is **still unwritten** — treat the sections below as a directory, not as that plan.

## Execution records

Real-account, fill-in-the-blanks QA documents. Each follows the same discipline: prerequisites → scenarios with Pass/Fail/Partial and captured evidence → bug log → documentation corrections → written readiness decision. None of them is a design document; each records what actually happened when someone ran it.

| Document | Question it answers | Status |
|---|---|---|
| [EM10_5_STAGING_CHECKLIST.md](EM10_5_STAGING_CHECKLIST.md) | Did Gmail ingestion work correctly? (label → policy → sync → chunk → embed → index) | In progress — Scenarios 1–3 pass; 4–9 unrun; Bug 6 open |
| [EM10_5_SLACK_VALIDATION.md](EM10_5_SLACK_VALIDATION.md) | Can Slack correctly retrieve and present the knowledge Gmail ingestion produced? | Not started |
| [GOOGLE_OAUTH_VERIFICATION_CHECKLIST.md](GOOGLE_OAUTH_VERIFICATION_CHECKLIST.md) | Is the Gmail OAuth Client verified for use beyond the test-user allowlist? | External process, parallel and non-blocking |

Planned but not yet written: `EL10_STAGING_CHECKLIST.md` — real-account and prompt-injection validation of the live email lookup **backend**, via direct signed calls. See [LIVE_EMAIL_LOOKUP.md §EL10](LIVE_EMAIL_LOOKUP.md#el10--staging-validation-and-security-testing). The Slack validation document above contains a Part D live-lookup smoke check, which does **not** discharge EL10.

## Standing rules these records share

- **Record the environment.** EM10.5's own prerequisites required staging and its execution ran against production anyway. Every record must state which environment, which Supabase projects, and — for destructive scenarios — an explicit acknowledgement of blast radius.
- **A found bug is the record succeeding, not failing.** These passes exist to let reality disagree with unit tests. Log it, fix it if small, spin it out as its own tracked item if large; do not let a validation pass silently become a second implementation pass.
- **Sign the readiness decision.** "Proceed" / "proceed with named caveats" / "block on X" — written by a reviewer, never inferred from "the tests passed."
- **Assert against the right layer.** A defect found through one surface may belong to another. Attribute it before logging it.

## Automated test suites

Not a substitute for the above — every EM1–EM10 implementation record independently notes that no live OAuth flow or real provider round trip was exercised by unit tests.

- `Relativity` — `npm test` (`test/`, node:test; DI-faked provider and Supabase clients throughout)
- `aikb` — `npm test` (`test/`)

Both must be green before any execution record above is started, and re-run after any fix a record produces.
