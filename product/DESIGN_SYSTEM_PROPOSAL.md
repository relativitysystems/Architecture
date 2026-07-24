# Relativity Design System — Proposal

**Status: proposed, not implemented.** Nothing in this document has been applied to `relativitysystems/Relativity`. It is a design direction and a set of rules for someone (human or agent) to implement in a later, separate change. Source repository for the audit below: `relativitysystems/Relativity`, specifically `public/shared/theme.css`, `public/portal/`, `public/marketing/`, `public/admin/`.

## Implementation scope: marketing landing page only

The audit in §1 covers all three surfaces (client portal, admin console, marketing site) because the inconsistencies are easiest to see in contrast with each other. **Implementation is restricted to `public/marketing/` (`index.html`, `mainPg.css`) only.** The client portal (`public/portal/`) and the internal admin console (`public/admin/`) are not to be touched — no visual, structural, or token changes to either surface.

This has one binding architectural consequence: **`public/shared/theme.css` must not be edited**, because it is loaded by all three surfaces (`portal.html`, `admin.html`, and `index.html` all `<link>` it before their own stylesheet). Any change there — even one intended only for marketing — would bleed into the portal and admin console. Concretely, this means:
- No unifying `--radius`/`--transition` with the portal's values. `mainPg.css`'s own local `:root` (currently `--radius: 8px`, `--transition: 0.3s ease`) stays as the landing page's independent token source — the fact that it now differs from the portal's is a deliberate boundary, not a bug to fix.
- No `--warn`/`--accent`/`--attention` split in the shared token file. If the marketing page's own `--badge-orange` usage (`.label`, `.eyebrow`, `.workflow-step__num`, `.growing-label`) needs a clearer name, that renaming happens via a **marketing-local** variable declared inside `mainPg.css`'s own `:root`, aliasing the shared token — never by editing `theme.css` itself.
- The 8-hue `--badge-*` palette, the admin console's CRM badge system, and every portal component in §1.3/§1.1 are entirely out of scope and untouched by this plan.

Everything below that references the portal or admin console (most of §1.2–§1.4, §7's sidebar guidance, §9's provenance-thread-for-chat and signal-rail-for-document-rows framing) is retained as **future-scope reference only** — useful context for if/when this work extends to the portal, but not part of the current implementation. §11 has been rewritten to reflect the marketing-only phase plan actually being executed.

## 0. Why this exists

Relativity is knowledge infrastructure and an AI operating memory for a company — not a chatbot, not a document dashboard. The interface's job is to make a company trust that what it's told is traceable to a real source, and to feel like it's looking at a system of record, not a demo. The current UI (audited below) already has two genuinely good, non-generic instincts — a burnt-orange signal color instead of AI-purple, and true-black dark mode instead of navy — that most "AI product" styling would sand off. This proposal is built to sharpen those instincts, not replace them.

---

## 1. Current state audit

### 1.1 Design inconsistencies found

| Issue | Where | Detail |
|---|---|---|
| Duplicate token definitions with different values | `portal.css:1-4` vs `marketing/mainPg.css:8-11` | `--radius` is `12px` on the portal/login/admin surfaces but `8px` on marketing. `--transition` is `0.2s ease` on portal but `0.3s ease` on marketing. Same variable names, silently different values depending on which page loaded them — there is no single source of truth for either token. |
| Duplicate component definitions | `admin/admin.css` `.badge` vs `portal/portal.css` `.badge` | Two independent `.badge` rulesets with different variants (`--active`/`--pending` vs `--soon`/`--indexed`/`--indexing`/`--failed`) and different border-radius (`20px` vs `99px`). They'll drift further every time one is edited without the other. |
| Unused / inconsistent color surface | `shared/theme.css:50-58` | Eight `--badge-*` hues defined (purple, blue, teal, orange, emerald, slate, indigo, gray) but the actual UI only draws on orange (`--warn`) and the semantic success/error colors. The other six exist as an untethered rainbow with no clear ownership — the kind of "badge palette" generic dashboards accumulate when every new feature picks its own color. |
| Icon system has two competing conventions | `portal.js`/`portal.html` (inline stroke SVGs, `stroke-width="1.75"`) vs. `portal.css` unicode glyphs (`content: "▾"`, `content: "•"`, `content: "→"`, and a literal `✎` pencil character for session rename) | Most of the UI uses a clean, consistent Feather-style outline SVG icon set. But several controls fall back to raw Unicode glyphs for carets, bullets, and edit affordances. These render inconsistently across platforms/fonts and break the otherwise-deliberate icon language. |
| Hardcoded color bypassing the token system | `portal.css:2165,2167` (`.btn-team-invite-submit`) | Uses literal `#fa4902` and `#fff` instead of `var(--warn)` / `var(--btn-primary-text)`, even though the adjacent `.btn-team-invite` (`portal.css:1999`) correctly uses the token. Same visual color, two different maintenance paths. |
| Card-in-card nesting | `.integration-card` → `.slack-collections-section` → `.slack-collection-option`; `.panel` → `.email-policy-rule-row` inside `.email-policy-builder-section` inside a `.panel-body` | Bordered, radiused, padded containers nested three deep for what is structurally a list of settings rows. Each nesting level adds its own border/radius/background, producing visual noise disproportionate to the information inside. |

### 1.2 Generic UI patterns to move away from

- **Card-grid-as-default-layout.** `overview-grid` (stat cards), `doc-grid` (doc cards), `request-grid` (request cards), `.panel` (three side-by-side info panels) — nearly every piece of information on the Overview tab is boxed independently, each with its own border + `border-radius: var(--radius)` + background. This is the generic SaaS-dashboard tell the brief asks to avoid, and it's the single biggest gap between what exists today and "infrastructure system," which should read more like a control panel / ledger than a grid of tiles.
- **Rainbow badge system.** Eight hue tokens for what functionally needs three states (healthy/processing/attention) reads as generic multi-tenant SaaS labeling rather than a considered signal system.
- **Centered dark-overlay modal.** `.team-invite-modal` is the default Bootstrap-era modal pattern (fixed inset, `rgba(0,0,0,.65)` scrim, centered white card). Functional, but it's the least distinctive surface in the product and the one most likely to look like every other app.
- **Decorative fade-in-on-scroll as the only motion vocabulary** on the marketing site (`fade-in`/`visible` toggled by an IntersectionObserver, implied by the pattern) — motion that exists to look nice on scroll, rather than motion that reports a real state change. The portal side already does this correctly (skeleton shimmer, mic pulse, loading dots all report real async state) — the marketing site should adopt the same discipline instead of generic reveal animation.

### 1.3 Existing components worth preserving as-is

- **The token architecture itself** — `theme.css`'s `:root` / `html[data-theme="dark"]` split, consumed everywhere via `var(--*)`, is the right foundation. Extend it; don't replace it.
- **Space Grotesk + Inter pairing.** Space Grotesk for display/numerals (`.portal-title`, `.stat-value`, `.sidebar-title`, `.request-title`), Inter for body and UI chrome. This is already a considered, technical-feeling pairing rather than a generic system-font stack — geometric, slightly unconventional, not a font choice you'd see on a template SaaS site.
- **True-black dark mode** (`--bg: #000000`, `--surface: #0d0d0d`) rather than the near-universal dark-navy-gray (`#0f172a`/`#1e293b`) every "AI dashboard" style defaults to (confirmed by the design-system tool's own default suggestion below, which this proposal deliberately overrides). True black reads more like an OS/terminal surface than a SaaS dark theme — closer to "infrastructure" than "app."
- **The burnt-orange signal color** (`--warn: #fa4902`), used today for section labels, progress fills, and the invite CTA. It is already the product's only strong hue against a near-monochrome neutral field — the opposite of AI-purple-gradient, and worth deliberately elevating rather than diluting into a badge-color rainbow.
- **Ledger-style rows where they already exist** — `.team-table`, `.kb-doc-row`, `.kb-job-row` use borders-between-rows rather than boxes-around-rows. This is the pattern to generalize, not the card grid.
- **State-driven motion on the portal side** — skeleton shimmer, `mic-pulse` recording indicator, loading-dots thinking bubble, upload-progress fill — all communicate a real system state. Keep this discipline and extend it; this is more "infrastructure feedback" than "UI polish."
- **Inline stroke-SVG icon set**, once the Unicode-glyph stragglers (§1.1) are folded into it.

### 1.4 Opportunities for a recognizable Relativity identity

- The product's actual trust mechanic — every AI answer carries a **citation back to a source document** — has no visual identity of its own today; it's rendered as a plain collapsed list (`.kb-sources`) below the answer, indistinguishable from any other "sources" footnote pattern. This is the single highest-leverage opportunity: make provenance visually structural, not a footnote (see §9).
- There is no data visualization anywhere in the product yet (stat cards show raw numbers only). This is a blank canvas — an opportunity to define a monochrome-plus-signal-accent chart style from scratch rather than retrofit a generic chart library's default palette later.
- The sidebar-plus-card-grid shell is structurally identical to Notion/Linear/every dashboard template; the brand currently differentiates only by color token values, not by layout logic. A ledger-first layout (§5, §9) is the differentiator the brief is asking for.

---

## 2. Visual direction

**"Instrument, not dashboard."** The interface should feel like a well-made piece of lab or server-room equipment: matte surfaces, quiet neutrals, one disciplined signal color, precise alignment, and typography that reads like data rather than marketing copy. Not sterile — the warmth comes from the off-white paper-like light background (`#f5f4f0`, already in use — keep it, it's warmer than the `#fafafa`/`#ffffff` every SaaS app uses) and from generous whitespace, not from decoration.

Concretely, that means:
- Default to **flat matte surfaces** — no blur, no translucency, no gradients except the one functional gradient already in use (progress-fill). This is a hard rule, not a preference: glassmorphism and gradient hero art are exactly the "generic AI product" signals the brief asks to avoid, and they also fight the "calm, trustworthy" brief — translucent surfaces read as decorative, not authoritative.
- Prefer **rules and rows over boxes** for anything that is fundamentally a list (documents, jobs, team members, integrations, chat sessions). Reserve true bordered cards for a small, deliberate set of "vessel" containers: the chat surface, forms, and the top-line stat summary — not for every discrete fact on the page.
- Let the **burnt-orange accent mean one thing**: system activity / signal / attention-needed. Never use it decoratively. If a hue is needed for a state that isn't "signal," use ink-on-neutral (opacity steps of the existing `--text-*` scale) before reaching for a new color.

---

## 3. Typography system

Keep the existing pairing; formalize its usage rather than changing the fonts.

| Role | Font | Weight | Notes |
|---|---|---|---|
| Display / section titles / numerals | Space Grotesk | 600–700 | Already used for `.portal-title`, `.stat-value`, `.sidebar-title`. Extend to **all numeric readouts** (counts, timestamps, IDs, percentages) — not just headline stats — so that "this is a number the system is reporting" has one consistent visual signature across the product. Use `font-variant-numeric: tabular-nums` wherever numbers appear in a list or table so columns of digits align (currently only applied ad hoc, e.g. `.kb-mic-timer`). |
| Body / UI chrome / controls | Inter | 400–500 | Keep as-is. |
| Micro-labels (section labels, table headers, badges) | Inter | 600, `0.06–0.1em` letter-spacing, uppercase | Already the convention (`.section-label`, `.stat-label`, `.form-label`) — keep it, it's doing real work signaling hierarchy without adding size. |
| Long-form / prose (docs, chat answers) | Inter | 400, line-height 1.65–1.85 | Already correct (`.prose p`, `.kb-answer-text`). Do not tighten this for density's sake — trust-carrying text needs room to breathe. |

**Rule to add:** a single type scale, expressed as tokens instead of the current ad hoc rem values scattered across files (`0.62rem` through `2.5rem`, ~15 distinct sizes currently in use). Collapse to an 8-step scale (`--text-xs` 0.68rem → `--text-4xl` clamp(2.4rem,4.6vw,4rem)) and map every existing size to its nearest step during implementation. This is a consolidation, not a redesign — most current sizes already cluster near these steps.

---

## 4. Color system

Keep the token architecture (`:root` / `[data-theme="dark"]`) exactly as-is; change what the tokens are allowed to mean.

**Core neutrals** (unchanged): warm off-white `#f5f4f0` light background, true black `#000000` dark background, ink-based text opacity steps. These are correct and distinctive — do not swap for cooler grays or navy.

**Signal color** — one accent, doing one job:
- `--signal: #fa4902` (rename from `--warn`, which conflates "the brand accent" with "the warning state" — a real inconsistency to fix: today the same token means both "this is the brand color" on section labels and "something needs attention" on the gap-detection card). Split these:
  - `--accent: #fa4902` — brand signal: active processing, freshly-indexed content, primary emphasis (section labels, progress fill, primary invite CTA).
  - `--attention: #f59e0b` (the amber already used ad hoc for the knowledge-gap card, `rgba(245,158,11,...)` — promote it to a token instead of a repeated raw rgba) — reserved specifically for "needs a human decision" states (knowledge gaps, pending approvals).
- Keep `--success` / `--error` as the only other chromatic tokens.

**Retire the eight-hue `--badge-*` rainbow.** Replace with three semantic states only: `--state-healthy` (success green), `--state-processing` (accent orange), `--state-attention` (attention amber). Every badge, status dot, and left-rail indicator in the product should map to one of these three — not to an open-ended palette that grows every time a new feature ships.

**Dark mode:** keep true black; the badge/state token retirement applies identically (dark variants of the same three states already exist as pastels in `theme.css:97-105` — trim to three, drop the other five).

---

## 5. Spacing and layout rules

Formalize the scale that's already *implicitly* in use (current values — 6, 8, 9, 10, 12, 14, 16, 18, 20, 22, 24, 28, 32, 36, 40, 48, 64, 96px — are almost all multiples of 2, just untokenized) into explicit steps:

```
--space-1: 4px   --space-4: 16px   --space-7: 40px
--space-2: 8px   --space-5: 20px   --space-8: 48px
--space-3: 12px  --space-6: 24px   --space-9: 64px
                                    --space-10: 96px
```

Rules:
- **Fix the `--radius`/`--transition` split** identified in §1.1 — one value for each, defined once in `theme.css`, deleted from `mainPg.css`/`portal.css`. Proposed: `--radius: 10px` (splitting the difference, applied uniformly), `--transition: 200ms ease` everywhere except deliberately slower marketing reveals (400–600ms, motion-only, see §8).
- **Layout density is column-count, not card-count.** Where the current UI would add another `.stat-card`, prefer adding a row to an existing ledger or a column to an existing table before introducing a new bordered box.
- **Section rhythm stays generous** — `.portal-section { margin-bottom: 64px }` and `.portal-content { padding: 32px 40px 96px }` are already correct instincts for "calm" (this is not a dense admin-panel product); keep the current spacious content padding even as card usage is reduced.

---

## 6. Border, radius, shadow, and surface rules

- **One border color, two states.** Keep `--border` / `--border-hover` exactly as defined (`rgba(0,0,0,.10)` / `.20` light, `rgba(255,255,255,.08)` / `.16` dark) — hairline borders on a matte surface, no heavier treatment needed.
- **One radius value** (§5): `--radius: 10px`, used for every bordered container. Do not introduce a second "large radius" for hero elements — the calm/trustworthy brief is better served by consistency than by a marketing-vs-product radius split.
- **One shadow, used sparingly.** `--shadow-card` already exists and is appropriately subtle (`0 1px 4px rgba(0,0,0,.08)` light / `rgba(0,0,0,.4)` dark). Keep it as the *only* shadow in the system — reserved for the small set of true "vessel" surfaces (chat card, modals, dropdown menus) that need to visually lift off the page. Rows, table cells, and inline list items should never carry a shadow — flat surfaces read as more trustworthy/less "look at me" than elevated ones, which matters more here than in a typical consumer product.
- **No blur, no translucency, no gradients** except the one functional gradient (progress fill) — see §2. This is the line that keeps the product out of "generic AI SaaS" territory; it should be treated as a hard constraint during implementation, not a style preference to be revisited per-feature.
- **Left-rail state indicators, not full-tint backgrounds, for status rows.** `.upload-progress-panel`'s `border-left: 3px solid var(--warn)` is the right instinct — generalize it. A row reporting "processing" or "attention" gets a 2–3px accent-colored left border on an otherwise neutral row; it does not get a tinted background fill the way `.banner--success`/`.banner--error` currently do. Reserve full-tint banners for page-level, dismissible messages only (auth errors, save confirmations) — not for steady-state row indicators.

---

## 7. Navigation patterns

- **Keep the persistent left sidebar** (`.portal-sidebar`) as the primary navigation for the authenticated product — it's the correct pattern for a console/operating-system product (as opposed to top-nav-only, which suits marketing/content sites). Do not replace it with a top-nav-only or command-palette-only pattern.
- **Add a persistent, ambient system-status element** to the sidebar footer or nav bar — last-sync time, connector health, or "N documents indexed" as a quiet, always-visible line (not a stat card, not a badge — small type, `--text-muted`, no border). This directly reinforces "infrastructure that's always running" versus "app you occasionally open," and costs almost nothing to add given the identity data already exists (`nav-identity`, integration status).
- **Mobile:** keep the existing slide-in sidebar + scrim overlay pattern (`.portal-sidebar--open`, `.sidebar-overlay`) — it's a standard, correctly-implemented pattern and doesn't need reinvention.
- **Replace the centered dark-overlay modal** (§1.2) for one clear case — team invite — with an inline expanding row or a right-anchored slide-over panel instead of a centered dialog. Centered modals interrupt the "console" feeling by borrowing the most generic possible interaction pattern; a slide-over keeps the user's spatial context (they stay "in" the team ledger rather than being knocked into a dialog floating over a dimmed screen).

---

## 8. Motion and interaction principles

- **Motion reports state; it does not decorate.** Every animation in the system should answer "what changed?" — a document moved from uploading to indexed, a connector came online, a chat response arrived. This is already the discipline on the portal side (skeleton shimmer, mic pulse, loading dots) — extend it to the marketing site's current generic scroll-reveal, which should be retimed/re-justified around actual content reveals (e.g., the workflow-steps section animating in sequence to demonstrate the pipeline, not just "fade up because it's on scroll").
- **Timing:** interaction feedback (hover, focus, click) stays fast — 150–200ms ease, per the existing `--transition` token. Reserve slower timing (400–600ms) for content-level reveals only, never for controls.
- **Respect `prefers-reduced-motion`** throughout — not currently handled anywhere in the sampled CSS; add a single global override (`@media (prefers-reduced-motion: reduce) { * { animation-duration: 0.01ms !important; transition-duration: 0.01ms !important; } }`) rather than handling it per-component.
- **No motion as a substitute for hierarchy.** Avoid staggered entrance animations on dashboard load (a common "AI product feels alive" trick) — a system reporting real company knowledge should feel immediately present and stable, not like it's performing an intro animation every time someone opens it.

---

## 9. Signature UI elements unique to Relativity

These are the elements that should make a screenshot of Relativity recognizable without a logo:

1. **The provenance thread.** Today, sources are a collapsed list below a chat answer (`.kb-sources`). Replace it with a thin vertical rule running from the answer bubble down to its source citations, rendered as small inline chips (document name + page), not a boxed list — visually asserting "this claim is tethered to a document" as a permanent structural feature of every answer, not an optional footnote. This is the single highest-identity-value change available, because it's the one thing genuinely unique to what this product does (versus a generic chat UI).
2. **The signal rail**, generalized from the existing upload-progress left border (§6) — any row anywhere in the product that is mid-process or needs attention gets the same 2–3px accent-colored left edge. Once applied consistently (documents indexing, connectors syncing, gaps awaiting review), it becomes a recognizable "something in the system is in motion" language unique to Relativity, rather than borrowed badge/pill conventions.
3. **Tabular-numeral instrument readouts** (§3) for every count/timestamp/ID in the product, set in Space Grotesk with `tabular-nums` — a small, consistent detail that makes the numeric parts of the UI read like telemetry rather than copy.
4. **The ambient status line** (§7) — a permanent, unobtrusive "system is running" indicator that no competitor dashboard bothers to surface persistently, because most of them are session-based tools rather than always-on infrastructure.
5. **Ledger-first list rendering** (§5, §1.3) as the default for every collection of like items, with bordered cards demoted to a deliberately rare "vessel" treatment — the layout-level differentiator from every sidebar-plus-card-grid competitor.

---

## 10. Data visualization style

No charts exist in the product today — this is a clean-slate decision, not a migration.

- **Monochrome-plus-signal-accent only.** Primary series in ink (`--text-primary`/`--text-secondary` at varying opacity), the single `--accent` orange reserved for the one series or highlight that matters most in a given view (e.g., "documents indexed this week"). Never a categorical rainbow palette for multi-series charts — if more than two series are truly needed, differentiate by pattern/label/position before reaching for more hues.
- **Flat, gridline-light.** No 3D, no drop shadows on chart elements, minimal or no gridlines (rely on axis labels and direct labeling instead) — consistent with the flat-surface rule in §6.
- **Every chart gets a citation-equivalent**: a visible "as of [time]" / data-source line, same trust logic as the provenance thread — a data system should always disclose how current its numbers are.
- Favor **sparklines and small multiples** embedded in ledger rows (e.g., a 7-day ingestion-volume sparkline next to a connector's status) over large standalone dashboard charts — consistent with "instrument," not "analytics suite."

---

## 11. Phased implementation plan — marketing landing page only

All phases below touch only `public/marketing/index.html` and `public/marketing/mainPg.css`. `theme.css`, `public/portal/`, and `public/admin/` are not part of any phase.

**Phase 0 — Marketing-local token cleanup (no visual change).**
Formalize a spacing scale and type-scale as local variables inside `mainPg.css`'s own `:root` (§3, §5) — additive only, since `--radius`/`--transition` already live there independently of the portal and should stay that way (§ Implementation scope). If `.label`/`.eyebrow`/`.workflow-step__num`/`.growing-label`'s shared `var(--badge-orange)` reference needs a clearer name, add a marketing-local alias (e.g. `--marketing-accent: var(--badge-orange)`) inside `mainPg.css`, not in `theme.css`.

**Phase 1 — Remove the glassmorphism violation + motion discipline.**
`nav#nav.scrolled` currently applies `backdrop-filter: blur(16px)` on top of the semi-opaque `--nav-bg-scrolled` tint — this is the one place the marketing site actually contradicts the "no blur/glassmorphism" rule (§2, §6). Drop the blur; the existing tint (`rgba(245,244,240,.92)` light / `rgba(0,0,0,.85)` dark) is already dark/light enough to read as a solid bar once the blur is gone. Separately, retime the current scroll-triggered `.fade-in`/`.visible` reveal so it's justified by content sequence (e.g., the workflow-steps section revealing in pipeline order to demonstrate the flow) rather than firing identically on every section regardless of what it contains (§8).

**Phase 2 — Provenance-thread treatment for the hero demo mockup.**
The hero's `.proof-panel`/`.proof-exchange`/`.proof-q`/`.proof-a`/`.proof-cite`/`.proof-gap` markup is a static mockup of the product's Q&A-with-citations experience — this is the marketing-scoped home for the "provenance thread" signature idea in §9.1, applied purely as demo art (a thin connecting rule from the mock answer down to its `.proof-cite` source line) with zero changes to the real chat UI in the portal.

**Phase 3 — Verify.**
Load `public/marketing/index.html` in a browser in both light and dark mode, confirm the nav no longer blurs on scroll, confirm the hero mockup's new citation treatment reads correctly, and confirm nothing outside `public/marketing/` changed (`git status` should show only `index.html`/`mainPg.css`, matching the scope restriction above).

Data visualization (old §10/Phase 5) does not apply here — the marketing site has no charts and none are planned for it. Every portal- and admin-facing item from the original phase plan (signal rail, ledger conversion, real provenance thread on `.kb-sources`, sidebar status line, team-invite slide-over, badge/icon consolidation) is retained in §1–§9 as future-scope reference only, not scheduled work.

---

## Appendix: deliberate deviations from generic "AI product" defaults

Running this proposal's own product-positioning query through the UI/UX Pro Max design-system tool returned "Liquid Glass" styling with a purple/pink AI palette (`#7C3AED` primary, `#EC4899` accent, translucent blur effects) — the exact combination the brief explicitly asked to avoid, and the generic default most AI-branded products converge on. This proposal deliberately overrides that recommendation: the true-black/off-white neutral base and single burnt-orange signal color already in the Relativity codebase are a stronger, more differentiated foundation than the tool's generic AI-product suggestion, and the direction above builds on them rather than replacing them.
