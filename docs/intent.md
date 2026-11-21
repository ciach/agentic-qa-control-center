# Agentic QA Control Center — Intent

> **Status:** Proposed product intent — v0.1  
> **Date:** 2026-11-21  
> **Deployment decision:** Local Docker Compose only; OpenAI API is the selected external model provider.  
> **Companion documents:** [plan.md](./plan.md) · [spec.md](./spec.md)

## 1. Mission

Build a **human-supervised, locally hosted, autonomous QA exploration and testing platform** that can investigate an application through its browser UI without requiring source-code access, discover states and user journeys, observe runtime and visual behavior, propose and generate useful Playwright end-to-end tests, and produce a trustworthy, readable quality gate report after every run.

The platform must answer, at any moment:

1. **What can the agent do?** Which environment, identity, actions, budgets, tools and permissions are available?
2. **What is it doing?** Which state is it in, which action is executing, why, and with what evidence?
3. **What has it done?** Which pages, states, transitions, user journeys and negative paths have been covered?
4. **What will it do next?** What is planned or queued, what is blocked, and which approval is needed?
5. **What is the result?** What passed, failed, was inconclusive or remains unexplored; what changed since the baseline?
6. **What did it cost?** Tokens, model calls, time, screenshots, browser resource usage and estimated API spend.
7. **Can we trust it?** Can a reviewer reproduce findings, inspect traces, approve decisions, and separate observations from guesses?

## 2. Product vision

A user enters the URL of an **authorized** test application, chooses a test identity and execution policy, and launches a run. The live dashboard draws an evolving **application state graph** while the agent explores. Each navigation, click, network failure, console exception, notification, screenshot and decision becomes a timestamped event with evidence links. Interesting discoveries become hypotheses; approved discoveries can become deterministic Playwright tests. Each run ends with an executive-level **PASS / FAIL / INCONCLUSIVE** gate and progressively deeper report sections.

The user is always in control: pause, resume, cancel, reorder planned work, grant a narrowly scoped approval, reject an unsafe action, inspect artifacts and compare runs.

## 3. Problems being solved

- Traditional E2E automation needs predefined flows and misses unknown branches, unfamiliar states and emerging business logic.
- Unguided AI browser exploration may be unrepeatable, opaque, wasteful or unsafe.
- URLs alone are not a sufficient representation of application coverage: identity, data and UI state matter.
- Visual and behavioral regressions are frequently separated from journey discovery and network/console evidence.
- Test artifacts are often too technical for an executive gate, while summary reports lose reproducibility.
- Test generation can be dangerous if AI silently turns an assumption into an assertion or updates approved snapshots.

## 4. Goals

### G1 — Black-box discovery first

Explore an authorized web application through its UI even when code and repository access are absent. Discover routes, interactive elements, forms, dialogs, notifications, role-dependent states, likely business processes, happy paths, negative paths and boundary conditions. Record an evidence-based state graph.

### G2 — A first-class human control center

Deliver a continuously updating, fully accessible dashboard for live browser view, map, action timeline, planned/blocked queue, run history, findings, evidence, model costs and policy/approval administration.

### G3 — Safe autonomous actions

Support scoped test actions (creating events, orders, bookings or test records) in approved test environments while **never treating a broad testing checkbox as authorization to spend real money, expose data or perform irreversible production operations**. Enforce permissions outside the model.

### G4 — Automated observations and inspections

Capture application behavior, network responses, console errors, screenshots, toast/notification appearance, timing, layout and design changes. Visual comparison must use stable, reviewable baselines and a deterministic screenshot pipeline, supplemented by AI explanations.

### G5 — Deterministic test factory

Turn discovered, evidence-backed journeys into test **proposals**; let a separate coding workflow generate reviewable Playwright tests with fixtures and meaningful assertions; execute them; promote them to the trusted suite only after checks and review.

### G6 — Memory and deliberate reflection ('dreaming')

Persist what was observed, what was tried, what was confirmed and what remains unknown. Between runs, use controlled reflection to group failures, detect coverage gaps, propose future routes and recommend tests. Reflection must not silently modify policies, baselines or approved assertions.

### G7 — Explainable, inexpensive operation

Store a traceable event history, attach cost to every model operation, enforce budgets, deduplicate exploration, and make every conclusion traceable to evidence. Report confidence and unknowns explicitly.

### G8 — Human-readable reports and a quality gate

Produce an accessible HTML dashboard report and downloadable machine-readable JSON / Markdown summary for each run. Show an immediate executive summary first, then gate outcomes, coverage, findings, visual diffs, evidence, limitations, costs and full trace references.

### G9 — Accessible design

Target **WCAG 2.2 Level AA** across the entire control center. Every key graph interaction also has an accessible list/table alternative. Include keyboard, focus, screen-reader, contrast, motion and status-announcement requirements.

## 5. Intended users

- **QA/operator:** starts runs, watches exploration, inspects evidence, validates findings and approves test promotion.
- **Developer:** reviews generated tests, reproduces bugs, fixes application issues and analyzes visual diffs.
- **Team lead/reviewer:** sees executive summary, quality gate, trend and release risk without reading raw traces.
- **Administrator (local MVP):** manages environments, identities, permissions, budgets, retention and secret references.

The MVP may use a single local administrator account, but its data model must preserve actor and authorization information for future roles.

## 6. Confirmed constraints and architectural principles

- **Hosting:** local server with Docker Compose; no cloud-hosted app components in MVP.
- **AI:** use the **OpenAI API** from the locally running orchestrator. This means model-bound inputs leave the local server. Use explicit data minimization, filtering and administrative disclosure.
- **Code availability:** no application source-code access is required for initial exploration; optional source-aware mode comes later.
- **Framework split:** Python/FastAPI/LangGraph for orchestration; TypeScript/Playwright for browser control and deterministic tests; React/TypeScript for the dashboard.
- **Evidence first:** an agent assertion is not a defect until validated against evidence, explicit expectations or reviewed requirements.
- **Separation of duties:** exploration, policy enforcement, deterministic execution and test promotion are separate trust boundaries.
- **Human override:** actions, priorities and approvals can be steered while a run is active.
- **Reproducibility:** every test execution uses recorded environment, identity, viewport, browser version, locale/timezone, data setup and baseline version.
- **No secret exposure:** the model does not see passwords, API tokens, session cookies, raw authorization headers or payment credentials.
- **No surprise external effects:** production and financial actions are denied by default, regardless of whether the application labels itself 'test'.

## 7. MVP definition

The MVP is a **local single-application, single-operator system**, initially with one test identity and one browser context at a time, that can:

1. Configure an allowlisted target and role with credential references.
2. Start/pause/resume/cancel an autonomous, bounded exploration run.
3. Show live state graph, current browser action, completed steps, next action candidates and blocked approvals.
4. Capture screenshots, network/console events and Playwright traces; view them from the UI.
5. Record notifications/toasts and compare controlled screenshots with approved baselines.
6. Discover a login journey, draft a Playwright test, execute it and display its result without auto-promoting it.
7. Produce a signed-off-ready run report with PASS/FAIL/INCONCLUSIVE gate, findings and clickable evidence.
8. Show per-run cost and spending/step budgets.
9. Enforce WCAG 2.2 AA accessibility checks for the control center; retain a manual audit checklist.
10. Run locally with Docker Compose; authenticate to the OpenAI API using a locally managed secret.

## 8. Explicit non-goals for MVP

- Unlimited production browsing, checkout or payments.
- Fully unattended deployment to arbitrary websites or unapproved domains.
- Multi-agent swarms or unsupervised self-modifying test suites.
- A vector database as a prerequisite; start with PostgreSQL and structured memory.
- Full code-aware test generation from an application's repository.
- Native mobile application testing.
- Distributed browser farms or multi-tenant SaaS.
- Proof that *all* application paths were covered; coverage is always relative to discovered or explicitly provided scope.
- Automated declaration of legal WCAG compliance; tool-based audits require human verification.

## 9. Definitions and truth model

- **Page/route:** URL or recognizable navigational location; not necessarily a distinct UI state.
- **State:** normalized browser-visible situation including route, role, meaningful UI controls, modal, workflow state and relevant data.
- **Transition:** action causing a change between states, including failed or blocked actions.
- **Journey:** ordered state/transition sequence representing a suspected or validated user process.
- **Observation:** directly captured evidence, such as a screenshot, network response or console error.
- **Hypothesis:** unverified claim, e.g. 'saving an event should show a toast'.
- **Confirmed finding:** reproducible behavior that violates an explicit expectation, test oracle or reviewed requirement.
- **Candidate test:** generated test that has not passed review/promotion.
- **Approved test:** versioned, reviewed, deterministic test included in a regression suite.
- **Coverage:** coverage of **known states and transitions**, not unknown total application behavior.
- **Gate:** policy-driven result (PASS, FAIL or INCONCLUSIVE) derived from evidence, completion and severity.

## 10. Success measures

The pilot is successful if a single approved application can be explored end-to-end and the dashboard can reliably reconstruct every decision. Initial measurable targets (to validate and adjust during pilot):

- 100% of browser actions have a linked event and outcome.
- 100% of high-severity findings have reproducible steps and attached evidence.
- 100% of sensitive attempted operations have a logged allow/deny/approval decision.
- At least one discovered happy path yields a generated, separately executed Playwright test.
- A completed or interrupted run produces an honest report, including the reason for any INCONCLUSIVE gate.
- Every agent/model call can be reconciled to per-run cost or labeled 'cost unavailable'.
- Keyboard-only users can reach and operate all essential dashboard functions; accessibility audits target WCAG 2.2 AA.

## 11. Remaining product decisions — recommended defaults

These are **not blockers to writing the specification or beginning an initial scaffold**.

| Decision | Recommended initial default | What changes it |
|---|---|---|
| Target application | One dedicated test/staging URL | First application to pilot |
| Repository | Public GitHub repo ciach/agentic-qa-control-center (approved for publication; never commit secrets) | Existing repo preference |
| Runtime | Docker Compose on local server | None for MVP |
| Model provider | OpenAI API with environment-configurable model | Budget/latency/privacy pilot |
| Browser | Chromium only | Browser coverage requirements |
| Authentication | One stored test identity via secret reference | App's auth type |
| Mutations | Explicitly permitted **test data only** | Environment owner approval |
| Purchases | Block actual payments and third-party side effects | Separately scoped, transaction-specific authorization |
| Quality gate | Fail on confirmed critical/high regressions; inconclusive for missing required evidence | Team release policy |
| Approval of baselines | Human review required | No automatic baseline adoption |
| Accessibility | WCAG 2.2 AA + keyboard/screen reader manual checks | None |

## 12. Related references

- WCAG 2.2 Recommendation: https://www.w3.org/TR/wcag/
- Playwright screenshot comparison: https://playwright.dev/docs/test-snapshots
- Playwright MCP screenshots + accessibility snapshots: https://playwright.dev/mcp/snapshots
- OpenAI API data controls: https://developers.openai.com/api/docs/guides/your-data

**Decision:** proceed with local-first architecture and the user-approved public repository ciach/agentic-qa-control-center; do not require application source code for the first milestone.
