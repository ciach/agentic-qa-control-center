# TestSprite benchmark and adoption decisions

> **Research date:** 2026-11-21
> **Reference product:** TestSprite 4.0 (announced 2026-11-20)
> **Status:** Product benchmark and design input, **not** an implementation record, integration commitment, endorsement or test of TestSprite source code.
> **Companions:** [Intent](./intent.md) · [Plan](./plan.md) · [Specification](./spec.md) · [Architecture review](./architecture-review.md)

## 1. Why this benchmark exists

The product owner identified **TestSprite** as a close illustration of the experience desired for Agentic QA Control Center. Our objective is to implement the *product loop*, not clone TestSprite's branding, graphics, proprietary code or cloud service.

**Reference loop:** describe expected behavior → normalize and approve requirements → explore the running app → identify observable journeys → propose and approve test plan → author and run tests → classify failures → safely repair test mechanics when necessary → publish evidence-backed report → retain regression coverage and history.

Our project keeps its distinct constraints: **Docker Compose locally**, **OpenAI API for external inference** (controlled outbound traffic), **local artifacts and secrets**, explicit action authorization, full exploration state/transition graph, cost accounting and **WCAG 2.2 AA** dashboard.

### Source verification and scope

- [TestSprite 4.0 launch (2026-11-20)](https://www.testsprite.com/blog/introducing-testsprite-4-0): plain-language/PRD/ticket intent, approval of expectations, live-browser exploration, replay, triage, test-healing without changing expected behavior, GitHub/Slack/Linear/Jira, CLI and MCP.
- [TestSprite official product page](https://www.testsprite.com/): browser, CLI and CI entry points, evidence bundles, tests retained between runs, schedules and QA gates.
- [TestSprite 3.2 changelog (2026-10-08)](https://www.testsprite.com/changelog): live Agent Action View, page movement map, list view, GitHub triggers, Slack results, issues, interactive SSO.
- [TestSprite 3.0 changelog (2026-04-24)](https://www.testsprite.com/changelog): editable feature map from PRD, exploration fleet, fixtures, API data flow and auto-cleanup.
- [How TestSprite works (2026-08-19)](https://www.testsprite.com/blog/what-is-testsprite-and-how-does-it-test-a-project-automatically): discover/plan/generate/execute/analyze/heal/report loop.

These are primarily **vendor descriptions**, not independent claims of quality or a verified API contract. Distinguish a visible capability from a proposed analogue; do not assert our project has implemented one.

## 2. Feature benchmark and design decisions

| Observed TestSprite concept | Our equivalent / difference | Priority and phase |
|---|---|---|
| Conversational brief and uploaded PRD/ticket | Local Project Brief: target URL, written goals, optional requirements file, roles, guardrails, test oracles and explicit scope | P1/M3; simple free-text run goal in M1 |
| **Editable Feature Map** with dependencies and edge cases | **Intent Map**: requirements, user outcomes, roles, expected behavior, test coverage and unknowns, stored separately from observed App State Map | P1/M3 (minimal requirement CRUD and approval); deeper dependencies later |
| Agent explores app before generating tests | Already core: evidence-backed semantic states and transitions with live map, timeline and frontier | P0/M2–M3 |
| Live action map, running previews and replay | Live Explorer + App Map + event timeline + Playwright trace/video replay, all evidence linked | P0/M1–M2; optional video optimization M4 |
| Explicit approval of test plan and uncertain assumptions | **Proposed → approved Test Plan**; assumption register with question, owner, source, status and expiry; unknown requirement is not a PASS | P1/M3–M5 |
| Tests derived from product intent, not code's current output | Dual-evidence test oracle: approved expected outcome from requirement; observed runtime result is evidence, never self-validating expectation | P0 design invariant; P1/M5 |
| Separate real product bugs from broken tests and environment faults | Failure triage: PRODUCT_DEFECT / TEST_FRAGILITY / ENVIRONMENT_FAILURE / UNKNOWN; evidence, confidence and reviewer override | P1/M4–M5 |
| Auto-heal steps when UI shifts | **Mechanics-only self-healing proposal** (locator/wait/navigation) followed by rerun; no automatic change to assertions, requirements, authorization or visual baselines | P2/M6; no auto-promotion in MVP |
| Real browser + backend API flows with setup/cleanup | UI journeys in MVP; passive API observation; optional API-chain test generation, fixture variables and verified cleanup after pilot | P2/M6+ |
| Consistent failure bundles with likely cause | Versioned finding package: expected/actual, failing step, adjacent steps, repro, source/trace/screenshot/network/console, likely cause and confidence | P1/M4–M5 |
| Human-readable dashboards and CI gates | Report with executive PASS/FAIL/INCONCLUSIVE and details; local runs first, optional GitHub status checks and CLI later | P1/M4; P2/M6+ for CI |
| Regression suite retention across changes | Reviewed test repository, requirement-to-test matrix, rerun history, versioned expectations and state coverage | P1/M5; advanced trends M6 |
| Design/Figma fidelity and visual issues | Controlled screenshot diff plus separate **semantic UX review**: toast visibility, consistency of labels, empty-state honesty, destructive confirmations, accessibility concerns | P1/M4; external design-file imports P2+ |
| Parallel exploration/browser fleet | **Not MVP**: start with one isolated browser/context, one identity, bounded costs and deterministic auditable execution | Deferred, after security and evidence quality |
| SaaS cloud sandboxes | **Not adopted**: all execution locally in Docker; OpenAI inference is the only chosen remote AI service | Non-negotiable |

### Competitive differentiation / must preserve

1. **Operator control of every action:** live view of planned work, allow/deny and human approval for sensitive operations. A model's plan is never authority.
2. **True application state graph**, with dialogs, roles, data markers and network failures as first-class state/transition evidence, not merely a page sitemap.
3. **Locally stored deep traces and costs**: link model reasoning summary, step IDs, browser actions, network/console and visual artifacts; never export secrets to OpenAI.
4. **Accessible control center:** WCAG 2.2 AA, equivalent graph table, accessible execution/replay transcript, status announcements.
5. **Deterministic release gate** built from reviewed required checks; discovery without an approved oracle must not become PASS.

## 3. The two-map model (critical architecture update)

**Intent Map = what should work.** Stable project requirements and approved expected outcomes, with source provenance: operator brief, PRD, specification, ticket, optional code/design analysis. Feature nodes have ID, title, business objective, role, preconditions, expected result, severity/priority, dependencies, review status, approval metadata and version. Inferred expectations are **hypotheses** until approved.

**Application State Map = what actually happened.** Evidence-linked states/transitions discovered in an authorized browser; action outcome, role, route, UI state, timing, network, console and screenshots. This is observational, never the independent source of expected truth.

**Coverage/traceability links** connect Requirement ↔ Journey ↔ AppState/Transition ↔ TestCase/Execution ↔ Finding ↔ Report. They are many-to-many, with explicit status: proposed, supported, verified, contradicted, unmapped. The UI must show (a) specified-but-unvisited features, (b) observed-but-unspecified behaviors, (c) specified and verified, (d) contradictory outcomes and (e) blocked/unknown scope.

Changing approved expectations requires an actor, review, recorded reason and a version bump. Neither test auto-healing nor reflection may change expectations.

## 4. Target experience and user journeys

### 4.1 Brief the project

An authenticated operator picks an allowlisted local/staging target, approved identity reference and scoped action policy; writes a short natural-language brief; optionally attaches requirements/design documents; chooses browser, viewport and budget. Secret values remain in the broker, never uploaded.

The agent proposes a structured feature/requirement map with evidence/source citations and questions. Operator edits/approves expected outcomes. For incomplete briefs, exploring unknown behavior is permitted but the release gate remains INCONCLUSIVE until required oracles exist.

### 4.2 Observe and discover

The operator starts a constrained exploration. Live Explorer displays currently viewed state, prioritized next actions, action reason, approval blocks, per-step evidence and running cost. The **App State Map** grows separately from the Intent Map, and both have accessible list/table views. Selecting a requirement filters observed journeys and candidate tests; selecting a state shows matching requirements and gaps.

### 4.3 Approve the plan and execute

The agent proposes tests grouped by requirement and risk: happy/negative/boundary, role/permission, toast/notification, visual checkpoint and optional API observation. Each proposal records the expected behavior **from approved intent**, observed preconditions and coverage links. Human reviews plan before promotion into a trusted suite; the coding agent generates Playwright tests separately.

### 4.4 Triage failure and replay

A failing check creates a **failure bundle**: requirement ID, expected-versus-actual, failing action and preceding/following events, screenshot(s), optional video/trace, network/console, exact run environment, artifact IDs and root-cause *hypothesis*. A triage state distinguishes suspected app bugs, test mechanics, environment errors and unknown oracles. AI may recommend, but unconfirmed hypotheses do not count as proven regressions.

### 4.5 Repair, rerun and report

If a selector drifts but intended behavior has not changed, the agent proposes a locator/wait repair as a code diff. It reruns the unchanged assertion, retains before/after traces and needs reviewer approval for promotion. If expected behavior no longer matches the approved requirement, it reports a product regression instead of silently making the test pass. Report begins with executive gate and drills down to requirements, journeys, diffs, errors, approvals and cost.

## 5. Information architecture / UI reference

- **Project Brief & Intent:** goal, product scope, editable feature map, assumptions, approval history and requirements coverage.
- **Live Explorer:** application state map, current browser/step, planned queue, live screenshots (not necessarily video), action controls, trace/cost.
- **Test Plan & Factory:** generated plan, intent-linked cases, reviewed oracles, Playwright drafts, executions and test-mechanics repair proposals.
- **Runs & Reports:** run list, gate, replay timeline, failure bundles, before/after visual diffs, coverage, API/network links and exports.
- **Policies & Settings:** target allowlist, identity broker, scoped actions, budgets, retention and OpenAI data egress.

**WCAG detail:** no meaningful information solely through map color, canvas motion or hover; map/list parity; replay has an accessible timestamped step transcript; test plan and assumptions are manageable by keyboard; visual diff includes text description; changing an approved oracle requires explicit dialog confirmation and accessible validation errors.

## 6. Proposed domain/schema delta

Add entities: ProductBrief, Requirement, RequirementSource, RequirementRevision, RequirementApproval, FeatureDependency, Assumption, TestPlan, TestPlanItem, RequirementJourneyLink, RequirementTestLink, FailureTriage and HealingProposal. Retain existing AppState, Transition, Journey, Finding, TestProposal and approved test storage.

Versioned events: brief.created/updated; requirement.proposed/approved/revised; assumption.opened/resolved; test_plan.proposed/approved; requirement.linked_to_journey; failure.triaged; healing.proposed/tested/approved/rejected. All include run/project correlation and reviewer provenance.

A minimum requirement contract:
```json
{
  "requirement_id": "REQ-LOGIN-001",
  "title": "Valid test user can sign in",
  "source": "operator_brief",
  "review_status": "APPROVED",
  "expected_result": "Authenticated dashboard is visible",
  "role": "test-user",
  "priority": "HIGH",
  "version": 1
}
```
This is an **example**, not a requirement learned from a live application.

## 7. Implementation priorities and exit tests

**M0/M1:** deliver existing infrastructure, restricted browser, events, redacted screenshots/traces and authenticated dashboard. Add a free-text run goal but do not delay infrastructure for full PRD import.

**M2/M3:** implement two-map data contract, minimal requirement editor, assumption ledger and approved plan state; visualize mappings with accessible tables. Reject unapproved guessed oracles when gating.

**M4/M5:** implement visual + semantic UX observation, expected/actual failure packages, triage and regression plan from approved intent, Playwright code generation and human promotion. Show linked requirements and evidence in reports.

**M6+ (deferred):** controlled locator/wait self-healing with immutable assertions; optional scoped API-chain test generation and cleanup; optional GitHub CI, schedules, tickets and CLI/MCP adapters; parallel workers only after isolation and policy verification.

### Benchmark acceptance tests

- B-01: A one-paragraph brief produces **proposed**, editable requirements; none are silently approved.
- B-02: Agent discovers a login flow absent from the brief → state appears as observed-but-unspecified, not automatically PASS.
- B-03: Approved login requirement is linked to a witnessed journey and a proposed Playwright test with exact expected behavior.
- B-04: App visibly behaves differently from approved requirement → FAIL when a required deterministic check confirms it; agent never rewrites oracle.
- B-05: Locator moves but expected behavior remains correct → repair proposal changes only mechanics; original assertion still executes.
- B-06: Browser fails to load because target is offline → ENVIRONMENT_FAILURE / INCONCLUSIVE, not an app bug without proof.
- B-07: A toast never arrives by approved deadline → deterministic failure with screenshot, timeline and relevant network/console evidence.
- B-08: Requirement map and state map are both usable with keyboard and screen reader; replay has equivalent text steps.
- B-09: Unconfirmed spec inference creates an open question and cannot upgrade an unknown check to PASS.
- B-10: All actions, changed expectations, approvals, repair proposals and costs are auditable; no secrets appear in OpenAI payloads or artifacts.

## 8. Explicit anti-goals and IP boundary

Do not copy TestSprite assets/UI source or rely on undocumented private APIs. Do not claim feature equivalence without acceptance testing. Do not deploy TestSprite cloud components into our local-only MVP. Do not import its reported marketing metrics as project targets. Our quality depends on **independent expected behavior, observable execution, safe execution policies and reviewer control**, not vendor feature labels.
