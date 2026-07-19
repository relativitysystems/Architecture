# Demo Video Strategy

Source repositories referenced for product accuracy: `relativitysystems/Relativity` and `relativitysystems/AIKB`, as documented in [../product/CLIENT_PORTAL.md](../product/CLIENT_PORTAL.md), [../architecture/CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md), [../product/KNOWLEDGE_GAP_DETECTION.md](../product/KNOWLEDGE_GAP_DETECTION.md), [../architecture/AIKB.md](../architecture/AIKB.md), and [../product/KNOWLEDGE_ANALYTICS.md](../product/KNOWLEDGE_ANALYTICS.md). Every product claim in this document is checked against those sources — see [Trust and Honesty Requirements](#trust-and-honesty-requirements) for the specific guardrails and why they exist.

## Purpose and Scope

This is the detailed strategy, script framework, and production plan for the demo video milestone tracked at roadmap level in [../roadmap/MASTER_ROADMAP.md](../roadmap/MASTER_ROADMAP.md)'s [Demo Video and Sales-Ready Demo Environment](../roadmap/MASTER_ROADMAP.md#demo-video-and-sales-ready-demo-environment) section and [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md)'s `GTM1` item. Those documents track the milestone's status, acceptance criteria, and dependencies; this document is the narrative, storyline, and script-level detail that sits underneath it. Update this document as the strategy evolves; update the roadmap documents only for status/dependency changes, not narrative detail, so the two don't drift into duplicated or contradictory content.

This document is a **content strategy and production plan**. It does not change, and has no bearing on, application code, deployment configuration, or product architecture.

## Demo Objective and Core Narrative

The demo is not a software walkthrough. Its job is to help a business owner understand what changes for their business — not to explain how the product works internally.

The demo should help a business owner see how Relativity can:

- Save employee time
- Reduce repeated questions and interruptions
- Lower the cost of onboarding and training
- Reduce mistakes caused by missing or outdated information
- Preserve institutional knowledge
- Reduce dependence on individual employees
- Improve operational consistency
- Identify missing documentation
- Give owners and managers greater peace of mind

**Core narrative, to be carried through every version of the demo:**

> Relativity turns scattered company knowledge into a trusted system that saves time, protects institutional knowledge, and gives the business confidence that important information will remain available when it is needed.

**What the demo is not about:** AI models, embeddings, vector search, retrieval-augmented generation, databases, APIs, or infrastructure. These are real parts of how Relativity works (see [../architecture/AIKB.md](../architecture/AIKB.md)), but they are not what a business owner is buying. Technical concepts may be mentioned briefly, only where they build trust (e.g., "every answer links back to the source document" is a trust statement, not a technical explanation of retrieval) — they should never dominate the presentation. Refer to the product as **Knowledge Infrastructure** for the business, not as an AI tool, chatbot, or search engine.

## Product Accuracy Guardrails

These constraints apply to every section of this document, the script, and all shorter variants. They exist because the current product intentionally does less than a naive reading of "AI knowledge assistant" would suggest, and overstating it would undermine the trust the demo is trying to build.

| Topic | What is actually true today | What must not be implied |
|---|---|---|
| Google Drive | A one-time, user-initiated Picker import of selected files ([../product/CLIENT_PORTAL.md](../product/CLIENT_PORTAL.md), [../architecture/CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md)) | Continuous sync, automatic refresh, change detection, or live synchronization |
| Slack | A question-and-answer surface: a user asks, Relativity answers with citations, scoped to whatever collections that Slack workspace is allowed to search ([../architecture/CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md)) | That Slack conversations are ingested, that Relativity learns from Slack messages, that historical Slack data is imported, or that Slack is itself a knowledge source |
| Knowledge Collections | Full CRUD, owner/admin-only, and their only currently-wired effect is scoping what Slack is allowed to search ([../product/CLIENT_PORTAL.md](../product/CLIENT_PORTAL.md)) | Per-user permissions, or any authorization model beyond owner/admin management and Slack-scoping |
| Knowledge gap detection | Automatic, heuristic detection on every query; **persistence requires an explicit "Save gap" click** in the portal today ([../product/KNOWLEDGE_GAP_DETECTION.md](../product/KNOWLEDGE_GAP_DETECTION.md)) | That gaps are automatically written, deduplicated, assigned, resolved, or reviewed by an admin workflow — none of that exists yet |
| Onboarding | Existing documents can be uploaded and made available quickly; the portal's Overview tab shows an onboarding-progress checklist tied to real indexed-document and question counts ([../product/KNOWLEDGE_ANALYTICS.md](../product/KNOWLEDGE_ANALYTICS.md)) | Any specific onboarding duration ("start seeing value in 10 minutes") unless it has actually been tested and verified |
| Citations | Every answer carries a structured `sources[]` array (filename, page numbers where available), generated programmatically, not by the model ([../architecture/AIKB.md](../architecture/AIKB.md)) | That citations are optional, approximate, or that the source shown might not be the actual retrieved document |
| Professional judgment | Relativity surfaces documented company knowledge | That Relativity replaces professional, legal, compliance, medical, or managerial judgment |
| Financial savings | Time and cost savings are plausible and estimable | That any specific dollar figure or percentage is guaranteed — see [Financial-Value Guidance](#financial-value-guidance) |

Before recording, and before publishing any cut of the video, re-check the script line-by-line against this table.

## Core Demo Framework

Every feature in the demo — in the full video, the shorter variants, and any live sales demo — should be presented using this six-step framework, not as a feature tour:

1. **Business problem** — What frustrating, expensive, risky, or repetitive situation already exists?
2. **Cost of the problem** — How does it waste time, increase labor cost, delay work, create mistakes, or add stress?
3. **Relativity solution** — Explain the outcome *before* naming the feature.
4. **Feature demonstration** — Show the exact product workflow.
5. **Business value** — Explain what the company gains.
6. **Peace-of-mind value** — Explain what the owner or manager no longer has to worry about.

Every feature section must answer, explicitly or implicitly: **why should a business owner care?** If a feature's narration can't answer that question in one sentence, rewrite the narration — don't just show the screen.

## Demo Storyline

### The fictional company: Meridian Property Group

A single, consistent, believable company anchors the whole demo — the video should feel like one connected story, not a series of unrelated examples.

**Meridian Property Group** is a residential and small-commercial property management company with 22 employees: leasing agents, on-call maintenance technicians, property managers, an accounting/billing team, and front-desk administrative staff, spread across several managed properties. It has exactly the conditions that make Relativity valuable:

- Multiple employees who need the same information at different times
- Written SOPs and policies (maintenance procedures, tenant escalation rules, refund policy) that already exist but are scattered across Google Drive, a shared drive folder, and a few people's memory
- Regular new-hire onboarding (leasing agents and maintenance techs turn over more than the core office staff)
- A senior operations coordinator, **Dana**, who has been there eight years and is the de facto answer to almost every "how do we handle this" question
- Real, daily use of Slack (agents coordinating in the field) and Google Drive (where SOPs and policies already live)

### The recurring story used throughout the demo

The narration follows one throughline, revisited feature by feature rather than jumping between unrelated examples:

1. **A new leasing agent, Jordan, joins Meridian.** Jordan needs answers fast — pet policy, lease-concession limits, escalation rules — but the SOPs are scattered across Drive folders Jordan hasn't seen yet, and asking a coworker means waiting or interrupting someone busy.
2. **Dana, the experienced operations coordinator, is interrupted constantly.** Every "where's the document for X" and "what's our policy on Y" question routes to Dana by habit, even when it's written down somewhere.
3. **A tenant asks about a security-deposit refund**, and the front-desk staff isn't sure of the current timeline without pinging Dana.
4. **A maintenance emergency comes in after hours** — and it turns out there's no documented after-hours escalation procedure at all. Meridian didn't know that gap existed until someone needed the answer and it wasn't there.
5. **The owner, watching all of this, wonders:** what happens when Dana takes a vacation — or eventually leaves? Is Meridian's operational knowledge actually the company's, or is it one person's?

This story is revisited in each feature section below — Jordan's onboarding, Dana's interruptions, the refund question, the after-hours gap, and the owner's peace-of-mind concern all reappear rather than being introduced once and abandoned.

## Feature-by-Feature Value Narrative

### 1. Fast onboarding and document upload

**Business problem:** Meridian already has SOPs, policies, and checklists — but adopting new software usually means a big, dreaded "implementation project" before anyone sees value. That fear alone stops small businesses from ever starting.

**Cost of the problem:** weeks of delayed value, employee time spent reorganizing content instead of using it, and reluctance to adopt yet another system.

**Relativity solution:** Relativity starts with what the business already has. Meridian doesn't need to rebuild its documentation before getting value from it.

**Product action shown:**
- Signing into the Meridian client portal
- The Overview tab's onboarding-progress experience
- Uploading an existing Meridian document (e.g., the Employee Handbook)
- Showing the document move through processing to "Ready in knowledge base"
- Asking an initial question grounded in that uploaded document

**Suggested narration:**
> "Relativity is designed to start with what your business already has. You can bring in existing documents, invite your team, and begin receiving value without completing a lengthy systems-integration project first."

**Time-saving value:** no weeks-long rollout project; the team can start asking questions the same day documents are uploaded.

**Financial value:** avoided implementation labor cost — no dedicated project to "get ready" for the tool.

**Risk-reduction value:** existing documentation isn't lost or orphaned during a platform switch.

**Peace-of-mind value:** the owner doesn't have to clear a big block of time or hire help just to "turn the system on."

**Transition:** once Jordan's onboarding documents are in the system, the natural next question is: what about everything already sitting in Meridian's Google Drive?

*(Do not promise a specific onboarding duration in the script unless it has been tested and verified against a real account.)*

### 2. Google Drive import

**Business problem:** Meridian's SOPs and policies already live in Google Drive. Nobody wants to manually recreate that work in a new tool.

**Product action shown:**
- Opening the Google Drive import flow from the Documents tab
- Selecting and importing a folder of existing Meridian policies (e.g., the Tenant Escalation Procedure, Maintenance SOP)
- Showing the imported files appear inside Relativity
- Asking a question grounded in one of the imported documents

**Suggested narration:**
> "Your company has already invested time creating this documentation. Relativity lets you bring that existing knowledge with you instead of starting over."

**Time-saving value:** no manual re-entry of existing SOPs; import happens in the time it takes to select the files.

**Financial value:** the labor already spent writing those SOPs isn't wasted or duplicated.

**Risk-reduction value:** avoids the common failure mode of software rollouts — documentation that "we'll migrate later" and never does.

**Peace-of-mind value:** adoption doesn't require a parallel effort to rebuild a document library from scratch.

**Accuracy requirement:** describe this strictly as a one-time import. Do not say or imply continuous sync, automatic refresh, change detection, or live synchronization — that is not how the current Google Drive integration works (see [Product Accuracy Guardrails](#product-accuracy-guardrails)).

**Transition:** with Meridian's existing SOPs now imported alongside the handbook, the next question is how a growing pile of documents stays organized as more come in.

### 3. Knowledge Collections

**Business problem:** as Meridian's document count grows — leasing policies, maintenance SOPs, HR documents, accounting procedures — a single undifferentiated pile of files becomes hard to navigate and hard to trust.

**Product action shown:**
- Opening the Collections tab
- Showing documents grouped into collections (e.g., "Leasing & Tenant Policies," "Maintenance Operations," "HR & Onboarding")
- Assigning a document to a collection from the per-document dropdown
- Explaining that a collection can scope which knowledge a connected surface like Slack is allowed to search

**Suggested narration:**
> "Company knowledge should be organized around how the business actually operates. Collections keep information structured as the company grows and help make answers more focused and relevant."

**Time-saving value:** employees and managers find relevant material faster as the document count grows.

**Financial value:** less time spent per search compounds across every employee, every week.

**Risk-reduction value:** reduces the chance that an answer draws on the wrong department's material (e.g., a leasing policy showing up in a maintenance question).

**Peace-of-mind value:** the owner has a structured way to keep the system organized as Meridian adds properties, staff, or documentation, instead of watching it become an unmanaged pile.

**Accuracy requirement:** do not imply per-user permissions or any authorization model beyond owner/admin management of collections and their current, sole wired effect — scoping what Slack can search. Do not describe collections as a general access-control or security feature.

**Transition:** organized knowledge sets up the next, most important question: can employees actually trust the answers they get back?

### 4. Trusted answers with source citations

**Business problem:** employees and owners hesitate to rely on an AI-generated answer if they can't verify where it came from.

**Product action shown:**
- Asking a realistic operational question (e.g., "What is the process for approving a security deposit refund?")
- Receiving an answer in the portal
- Opening or pointing to the cited source
- Showing the answer is grounded in an approved Meridian document, not a general answer

**Suggested narration:**
> "The goal is not to give employees another AI response they have to blindly trust. Every answer points back to the company source it came from, so the information can be verified."

**Business outcomes:** reduces the risk of unsupported answers; builds employee trust; makes the system appropriate for actual operational use, not just casual reference.

**Time-saving value:** managers no longer need to manually re-check every answer against the source document line by line — the source is already attached.

**Peace-of-mind message:**
> "The owner knows employees can verify important answers rather than acting on an unsupported guess."

**Transition:** portal answers are useful, but employees like Jordan and Dana spend most of their day in Slack, not the portal — which raises the next problem.

### 5. Slack integration

**Business problem:** employees constantly interrupt managers or experienced coworkers — like Dana — to ask where something is documented or how a process works. That interruption has a cost for *two* people: the employee waiting, and the experienced person who has to stop what they're doing.

**Product action shown:**
- Jordan mentioning Relativity in Meridian's Slack workspace
- Asking a natural-language operational question (e.g., "What's our pet policy for new leases?")
- Receiving a threaded response with a source citation
- Narration noting Jordan never had to leave Slack or open another tool — and never had to interrupt Dana

**Suggested narration:**
> "Instead of tracking down the person who knows the answer, employees can ask inside the place where work is already happening and receive a trusted response immediately."

**Business outcomes:** fewer repeated questions; fewer interruptions to Dana and other experienced staff; faster decisions in the field; stronger adoption because employees never have to change tools.

**Potential financial framing (example only, not a guarantee):**
> "If five employees each spend fifteen minutes per day searching for information or waiting for answers, that adds up to more than six employee hours each week."

Label any such calculation explicitly as an example — see [Financial-Value Guidance](#financial-value-guidance) for the reusable formula.

**Accuracy requirement:** Slack is a question-and-answer surface today. Do not imply that Slack conversations are ingested, that Relativity continuously learns from Slack messages, that historical Slack data is imported, or that Slack itself is a knowledge source (see [Product Accuracy Guardrails](#product-accuracy-guardrails)).

**Transition:** Slack handles the questions that already have good answers. The next feature covers what happens when a question doesn't.

### 6. Knowledge gap detection

**Business problem:** most businesses don't know which procedures or policies are missing until an employee urgently needs one. A typical AI tool may guess confidently anyway, hiding the fact that the business has a real documentation gap.

**Product action shown:**
- Asking the intentionally unsupported question: "What is the process for handling a maintenance emergency after business hours?"
- Showing the gap response — Relativity says it doesn't have this documented, rather than guessing
- Clicking "Save gap" in the portal
- Explaining how a saved gap becomes an item Meridian can act on — write the missing after-hours procedure, rather than continuing to improvise it

**Suggested narration:**
> "Relativity does not hide missing knowledge behind a confident-sounding answer. When the information is not documented well enough, it surfaces the gap so the business knows what needs to be improved."

Position this as a major differentiator, not a minor feature.

**Business outcomes:** identifies undocumented processes before they become an emergency; prevents the same unanswered question from recurring silently; gives management a concrete, evidence-based documentation to-do list; reduces reliance on tribal knowledge.

**Peace-of-mind message:**
> "The owner no longer has to assume everything important has been documented. Relativity helps show where the organization is still exposed."

**Accuracy requirement:** reflect the current workflow exactly — the system detects insufficient knowledge, the portal surfaces the gap, and a user can save it. Do not imply that the system automatically writes, assigns, resolves, or approves missing documentation; none of that exists today (see [../product/KNOWLEDGE_GAP_DETECTION.md](../product/KNOWLEDGE_GAP_DETECTION.md) and [Product Accuracy Guardrails](#product-accuracy-guardrails)).

**Transition:** this gap — an undocumented after-hours emergency procedure — is exactly the kind of exposure that becomes serious the moment a key person is unavailable, which leads directly into the demo's emotional close.

## Institutional Knowledge and Peace of Mind

This is one of the emotional centers of the video, not a footnote. Business owners carry a quiet, ongoing worry that rarely gets said out loud:

- What happens if a key employee — like Dana — leaves?
- What if the operations manager is out sick or on vacation during an emergency?
- Where is the latest version of this process, really?
- Can a brand-new employee, like Jordan, actually find the right answer without waiting on someone else?
- Is important knowledge trapped in one person's head?
- Will the business keep running consistently through employee turnover?
- Do we even know which processes are undocumented?

Relativity's value is not just faster search. It helps turn knowledge that has been held by individuals into an asset the organization actually owns.

**Suggested narrative direction:**
> "Every business depends on knowledge that has accumulated over years — policies, procedures, customer lessons, pricing decisions, and experience held by key employees. When that knowledge is scattered or trapped in people's heads, the company is exposed. Relativity helps preserve that knowledge as part of the business itself."

Peace-of-mind value is real value, even where it can't be reduced to a single number. It includes:

- Confidence during employee turnover
- Less dependence on any one experienced person, like Dana
- Confidence that employees can find approved, current information
- Reduced fear of knowledge walking out the door when someone leaves
- Greater operational continuity through absences, vacations, and emergencies
- A clearer, evidence-based view of what the company knows — and doesn't yet know

## Financial-Value Guidance

Discuss money as an estimation framework, never as a guarantee. Plausible savings categories:

- Reduced time searching for documents
- Fewer interruptions to managers and senior staff
- Faster onboarding
- Less repeated training
- Fewer preventable mistakes
- Reduced duplication of work
- Less time answering recurring internal questions
- Better reuse of documentation the company has already created

**Optional ROI framework (present as an example calculation, not a fact):**

```
Employees affected × minutes saved per employee per week × hourly labor cost × 52 weeks
```

**Manager-interruption cost framework:**

```
Managers interrupted × average interruption time × interruptions per week × manager hourly cost × 52 weeks
```

**Rules for using these in the script or any conversation with a prospect:**

- These are estimation frameworks for the viewer or prospect to apply with their own numbers — not Relativity's numbers.
- State clearly, on-screen or in narration, that actual savings vary by company size, wages, usage, and existing inefficiency.
- Never state an invented ROI percentage or dollar figure as fact.
- As real customer usage data accumulates, replace these illustrative frameworks with actual case studies — see [Future Case-Study and ROI Evidence Plan](#future-case-study-and-roi-evidence-plan) below.

## Recommended Video Structure

An approximately 6–10 minute primary demo video. Treat these timings as planning targets — the final length may shift during scripting and editing.

| Section | Approx. time | Content |
|---|---|---|
| 1. Opening business problem | 30–60s | Company knowledge scattered across documents, Google Drive, Slack, individual employees, old processes, informal conversations. Do **not** open with AI technology. |
| 2. The cost of scattered knowledge | 30–60s | Repeated questions, employee search time, interruptions, slow onboarding, mistakes, dependence on key employees (Dana), owner stress. |
| 3. Relativity overview | 30–45s | Introduce Relativity as company Knowledge Infrastructure: *"Relativity turns the knowledge your company already has into one trusted system that employees can use through the portal and Slack."* |
| 4. Product walkthrough | 4–6 min | The six features in [Feature-by-Feature Value Narrative](#feature-by-feature-value-narrative), in this connected order: (1) onboarding and document upload, (2) Google Drive import, (3) Collections, (4) cited portal answer, (5) Slack question and answer, (6) knowledge gap detection. Follow Meridian's story continuously — do not jump between unrelated screens. |
| 5. Business impact recap | 45–90s | Less time searching, fewer repeated questions, faster onboarding, more consistent answers, better documentation, preserved institutional knowledge, reduced dependence on individuals. |
| 6. Peace-of-mind close | 30–60s | *"Relativity is not just about finding an answer faster. It is about knowing that your company's knowledge is preserved, accessible, and improving over time — even when employees are busy, unavailable, or no longer with the company."* |
| 7. Call to action | — | Invite the viewer to book a demo, discuss their current knowledge workflow, and identify where their business loses time or depends too heavily on individual employees. Avoid a generic "contact us to learn more." |

## Demo Script Requirements

The final script — a separate working document produced from this strategy, not part of this document itself — must specify:

- Exact narration, word for word
- Exact screen actions, in order
- Expected screen state at each step
- Demo account data required for each step
- Documents that must be uploaded in advance, and into which collections
- The exact questions that will be asked, in order
- The expected cited answer for each question
- The intentional knowledge-gap question and its expected gap response
- Slack workspace/channel setup requirements
- Backup actions to take if a live response fails during recording (e.g., a pre-recorded clip of the same interaction) — see [Demo Account Preparation](#demo-account-preparation-checklist)
- Sensitive-data precautions (no real customer, tenant, or employee data anywhere in the account)
- A recording checklist (see below)
- Editing notes (cuts, callouts, captions, pacing)
- Exact call-to-action language

The script must be **rehearsable and repeatable** — a different presenter (or the same one, weeks later) should be able to follow it and get the same result.

## Demo Account Preparation Checklist

- [ ] A fictional company with a believable name and business type — **Meridian Property Group**, property management (see [Demo Storyline](#demo-storyline))
- [ ] A clean, dedicated demo user account — no personal, customer, tenant, or production data of any kind
- [ ] A curated set of realistic company documents (see [Suggested Demo Documents](#suggested-demo-documents))
- [ ] Multiple Collections configured (e.g., Leasing & Tenant Policies, Maintenance Operations, HR & Onboarding)
- [ ] A working Google Drive import example, pre-tested
- [ ] A connected Slack workspace or dedicated demo channel, pre-tested
- [ ] Pre-tested questions with confirmed, reliable answers (see [Suggested Demo Questions](#suggested-demo-questions))
- [ ] At least one intentional unanswered question, confirmed to trigger gap detection reliably
- [ ] Citations confirmed to resolve to the correct source document
- [ ] Onboarding-progress steps completed where the walkthrough calls for a "before" state and a "just completed" state
- [ ] Clean chat history — no leftover test questions or broken exchanges
- [ ] No developer logs, admin console views, or internal clutter visible on screen
- [ ] Browser bookmarks and tabs prepared and limited to what's needed
- [ ] Notifications disabled on the recording machine (OS, browser, Slack app)
- [ ] Personal profile information (real name, email, avatar) replaced with demo-appropriate values
- [ ] Backup screenshots or short recorded clips prepared for each live integration step, in case of a failure during recording

## Suggested Demo Documents

A coherent document set for Meridian Property Group, sized to support every question in [Suggested Demo Questions](#suggested-demo-questions):

- Employee Handbook
- PTO and Attendance Policy
- Security Deposit Refund Policy (the "customer refund" analog for a property manager)
- New Leasing Agent Onboarding Checklist
- Tenant Escalation Procedure
- Lease Pricing and Concession Guidelines (the "discount approval" analog)
- Data Security Policy (tenant and payment information)
- Prospective Tenant Qualification Guide (the "sales qualification" analog)
- Maintenance Request SOP
- Frequently Asked Questions

**Intentionally left undocumented:** the after-hours emergency-maintenance escalation procedure. This is the realistic gap used to demonstrate knowledge gap detection honestly — it is not documented anywhere in the demo account on purpose, not because of an oversight.

## Suggested Demo Questions

Answerable questions, each grounded in one of the documents above:

- What is the process for approving a security-deposit refund?
- How many days of PTO do new employees receive?
- When should a tenant issue be escalated to a property manager?
- Can a leasing agent offer a lease concession without approval?
- What steps should a new leasing agent complete during their first week?

**Intentionally unsupported question** (triggers knowledge gap detection):
- What is the process for handling a maintenance emergency after business hours?

Only use questions that match Meridian's actual prepared documents — do not introduce a question the demo account can't actually answer correctly, except the one intentional gap question above.

## Trust and Honesty Requirements

The demo must clearly distinguish current capabilities, planned capabilities, and long-term direction. Before finalizing the script or any cut of the video, confirm it does **not** imply:

- That Google Drive continuously syncs, when it is currently a one-time import flow
- That Relativity ingests Slack conversations, when Slack is currently a Q&A surface only
- That knowledge gaps are automatically resolved, assigned, or reviewed
- That Collections provide permissions beyond their current owner/admin-managed, Slack-scoping behavior
- That Relativity replaces professional, legal, compliance, medical, or managerial judgment
- That any financial savings figure is guaranteed rather than an illustrative estimate

See [Product Accuracy Guardrails](#product-accuracy-guardrails) for the full reference table this checklist is derived from.

## Recording Style

A calm, confident, founder-led presentation. Tone: clear, business-focused, trustworthy, specific, not overly technical, not overly dramatic, free of startup jargon.

**Avoid:**
- Reading feature names one after another
- Long technical explanations
- Overusing the word "AI"
- Calling everything revolutionary
- Making unverified numerical claims
- Moving through screens too quickly
- Showing unfinished interfaces
- Apologizing during the recording
- Presenting future features as though they exist today

**The viewer should come away feeling:**
- This solves a real problem
- I understand how it works
- My team could use this
- This could save us meaningful time
- This would make our operations less fragile
- I would feel more confident knowing our knowledge is preserved

## Deliverables and Variants

### Deliverables

1. Demo strategy — this document
2. Fictional company scenario — Meridian Property Group (above)
3. Demo document set — see [Suggested Demo Documents](#suggested-demo-documents)
4. Demo account setup checklist — see [Demo Account Preparation](#demo-account-preparation-checklist)
5. Full narration script (separate working document, produced from this strategy)
6. Screen-action script (separate working document, paired with the narration script)
7. Recording checklist (part of the script requirements above)
8. Editing checklist (part of the script requirements above)
9. Short-form demo variant (below)
10. Sales-call live-demo variant (below)
11. Follow-up process for collecting viewer feedback (below)
12. Future case-study and ROI evidence plan (below)

### Recommended variants

All shorter variants are edited down from the same full demo recording — the 6–10 minute full demo is the source material, not a separate production:

| Variant | Length | Use |
|---|---|---|
| Homepage overview | 60–90s | Website hero section, cold outreach |
| Product overview | 2–3 min | Warmer prospects, email follow-up |
| Full demo | 6–10 min | Primary sales asset, direct outreach, first substantive prospect conversation |
| Live founder-led sales demo | Variable | Real-time, using the same script and Meridian account, adapted live to prospect questions |

### Follow-up process for collecting viewer feedback

After each founder-led sales conversation or outreach send using the video:
- Note any question the viewer asked that the video didn't answer
- Note any point where the viewer looked confused, disengaged, or skeptical
- Note any objection raised, and whether it was about the product, the pricing, or the pitch itself
- Feed this back into script revisions on a regular cadence (e.g., after every 5–10 conversations), rather than waiting for a full re-shoot

### Future case-study and ROI evidence plan

Once real customers have used Relativity long enough to produce evidence:
- Replace the illustrative ROI formulas in [Financial-Value Guidance](#financial-value-guidance) with actual customer numbers, clearly attributed
- Add a customer-quote or case-study section to the full demo and the product-overview variant
- Keep the estimation frameworks available for prospects who don't yet have a matching case study, but prefer real evidence wherever it exists

## Success Criteria

A successful viewer, after watching the full demo, should understand:

- What problem Relativity solves
- Why scattered knowledge costs businesses time and money
- How the current product works
- Why Slack access improves adoption
- Why citations make answers trustworthy
- Why Collections matter as knowledge grows
- Why gap detection is strategically valuable
- Why onboarding does not require rebuilding everything
- How Relativity protects institutional knowledge
- Why this can give an owner peace of mind
- What action to take next

**Potential metrics to track once the video is live** (not yet instrumented — a future measurement plan, not a current capability):
- Video completion rate
- Demo-booking conversion rate
- Call-to-action click rate
- Common viewer questions
- Sections where viewers stop watching
- Number of qualified conversations generated
- Number of viewers who can correctly explain the product afterward
- Objections raised after viewing

## Related Documents

- [../roadmap/MASTER_ROADMAP.md](../roadmap/MASTER_ROADMAP.md) — roadmap-level tracking of this milestone, acceptance criteria, and dependencies
- [../roadmap/FEATURE_BACKLOG.md](../roadmap/FEATURE_BACKLOG.md) — the `GTM1` backlog item
- [../product/CLIENT_PORTAL.md](../product/CLIENT_PORTAL.md) — portal features referenced throughout this script
- [../architecture/CONNECTOR_FRAMEWORK.md](../architecture/CONNECTOR_FRAMEWORK.md) — Slack and Google Drive behavior referenced in the accuracy guardrails
- [../product/KNOWLEDGE_GAP_DETECTION.md](../product/KNOWLEDGE_GAP_DETECTION.md) — the current gap-detection workflow shown in feature 6
- [../architecture/AIKB.md](../architecture/AIKB.md) — citation generation, referenced in feature 4
- [../product/KNOWLEDGE_ANALYTICS.md](../product/KNOWLEDGE_ANALYTICS.md) — the onboarding-progress checklist referenced in feature 1
