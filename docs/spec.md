# Agentic QA Control Center — Technical and Product Specification

> **Status:** Implementation-oriented specification v0.1  
> **Date:** 2026-11-21  
> **Companions:** [Intent](./intent.md) · [Delivery plan](./plan.md) · [Architecture review](./architecture-review.md)  
> **Deployment:** Local Docker Compose; OpenAI API is an intentionally external inference/data-processing dependency.

## 1. Product contract and non-negotiable invariants

One operator uses an accessible web console to supervise an agent exploring an **authorized** web application without needing source code. The system maps observable **states and transitions**, not just URLs; records all browser actions and evidence; detects functional and visual changes; suggests business journeys; proposes deterministic Playwright E2E tests; and creates one report with an executive quality gate per run.

1. **The model never controls authorization.** Every browser action passes through a trusted policy/tool gateway.
2. **No raw credentials in prompts.** A trusted broker owns passwords, tokens and browser session material.
3. **Evidence is distinct from inference.** Observed behavior, guessed business intent and verified requirements have different statuses.
4. **Explorer and coding agent are different trust boundaries.** The explorer cannot change approved tests, baselines, release gates or permissions.
5. **Missing required evidence cannot PASS.** Interrupted, blocked and unresolved required checks lead to INCONCLUSIVE unless a confirmed blocking defect independently requires FAIL.
6. **Runs are reproducible.** Persist environment, actor, identity, policy, prompt/model, browser, viewport, locale, fixture and baseline versions.
7. **Events are durable.** Every material action, result and approval is ordered, attributable and replayable.
8. **The application graph is accessible.** A keyboard/screen-reader table offers the same data and actions as the visual graph.
9. **Transaction risk is scoped.** Sandbox test-data mutations can be authorized; real payments, invitations to external recipients, and production mutations are denied by default.
10. **Local hosting is not zero egress.** Carefully minimized observations go to the OpenAI API with explicit operator disclosure and filtering.

## 2. Architecture and technology

| Layer | Choice | Responsibility |
|---|---|---|
| Dashboard | React, TypeScript, Vite, React Flow | Live browser control, application map, reports |
| Accessible UI | Accessible primitives, semantic HTML, axe-core | WCAG 2.2 AA, keyboard and assistive technology |
| API | Python FastAPI/Pydantic | Auth, run lifecycle, commands, SSE, reports |
| Agent orchestrator | Python LangGraph | Planner, checkpointing, interrupts, memory, reflection |
| AI | OpenAI API; configurable model per workflow | Planning, semantic UI interpretation, reasoning |
| Browser worker | Isolated Node/TypeScript Playwright + restricted MCP adapter | Observe and act through policy gateway |
| Regression | Playwright Test (TypeScript) | Deterministic functional/visual test suite |
| Background work | Redis + Dramatiq | Bounded tasks, retries, leases, scheduling |
| Durable storage | PostgreSQL with migrations | Run journal, state graph, policies, tests, approvals |
| Evidence storage | Local MinIO / S3-compatible service | Images, traces, diffs, report artifacts |
| Model observability | Locally hosted Langfuse | Traces, model usage, cost and evaluations |
| Deployment | Local Docker Compose | No cloud-hosted app services |

Start with a **single monorepo and one browser worker**, not a microservice swarm. API and orchestrator may share modules; browser execution and test-authoring must remain isolated by process/container and credentials.

### 2.1 Data and control flow

1. Operator configures a project, authorized environment, identity reference, run goal and approved capability policy.
2. FastAPI validates the request, snapshots run configuration, persists a QUEUED event and enqueues work.
3. LangGraph planner retrieves prior confirmed observations and chooses a bounded next action.
4. A policy gateway evaluates exact capability, identity, origin, parameters, risk, approvals and remaining budget.
5. An isolated Playwright worker executes only allowed browser operations against the allowlisted application.
6. The worker records sanitized DOM/ARIA state, screenshot, console/network observations, action result and trace references.
7. The event journal and state graph are updated. SSE streams the same ordered events to the dashboard.
8. New knowledge may produce a finding hypothesis or a candidate test, never an automatic assertion of correctness.
9. A separate coding workflow drafts, checks and executes Playwright tests, subject to human review.
10. Reporting derives the quality gate from declared required checks and stored evidence; reflection creates future suggestions only.

### 2.2 Docker and network topology

- web: local reverse proxy hosting React assets; authentication required; loopback binding by default.
- api: FastAPI commands, event cursor/replay, status, authorization and artifact access checks.
- orchestrator: LangGraph, model calls, decision traces, checkpoint persistence.
- browser-worker: isolated Chromium and Playwright; one browser context per run/identity; no Docker socket.
- jobs: reports, screenshot diffing, reflection, cleanup; initially can share API image.
- postgres, redis, minio and langfuse: local private networking and persistent volumes.

Internal DB, Redis, MinIO administration, and Langfuse must not be publicly exposed. Make LAN access explicit, authenticated and TLS-protected if leaving loopback. Enforce container resource limits, non-root users, filesystem restrictions and network allowlists. Docker alone is not a hardened hostile-browser sandbox; for real untrusted sites, evaluate VM isolation and stronger egress controls.

## 3. Domain model and run lifecycle

| Entity | Key fields |
|---|---|
| Project | id, name, approved environment IDs |
| Environment | allowed origins/IP policy, tier, fixture/reset policy, auth adapter |
| IdentityRef | role, secret reference, allowed environments, session isolation |
| ExecutionPolicy | immutable version, capabilities, approvals, budget, deny rules |
| Run | id, goal, status, config snapshot, actor, policy, prompt/model, timestamps |
| Step | run, sequence, intended action, actual tool call, state, outcome, times |
| RunEvent | monotonically increasing sequence, type, actor, data, evidence references |
| AppState | semantic fingerprint, role, route, markers, confidence, provenance |
| Transition | source state, action, outcome, target state, evidence |
| Journey | ordered transitions, confidence, observed/reviewed/verified status |
| Finding | observation, severity, status, oracle, reproduction and evidence |
| Approval | canonical action hash, scope, actor, expiry, consumed flag |
| Artifact | kind, run/step, hash, sensitivity, storage reference, retention |
| Baseline | state, viewport/browser, fixture, locale, version and reviewer |
| TestProposal | linked journey, preconditions, oracle, generated patch, review status |
| Report | gate, required checks, coverage definition, findings, evidence, costs |

Lifecycle: DRAFT → QUEUED → STARTING → EXPLORING → FINALIZING → COMPLETED, with PAUSING/PAUSED, WAITING_APPROVAL, FAILED, EXPIRED and CANCELLED branches. **CANCEL_REQUESTED** is not equal to **CANCELLED**: only confirm after the worker stops and durable state is updated. Lease/fencing tokens prevent a previously stopped worker from resuming unexpectedly.

On crash, restore committed checkpoints and reconcile running work. A write whose result is uncertain must **not** be replayed blindly; reconcile against an observable receipt, idempotency key or a human decision.

### 3.1 Event journal

Events are appended in the same transaction as state changes, or through a transactional outbox. Suggested fields:

- event_id, run_id, per-run sequence, occurred_at, event_type
- actor kind/id, correlation_id, step_id, state_id, policy_version
- requested capability and sanitized parameters, decision, outcome and duration
- screenshot/trace/observation references and model usage where relevant

Event families: run.created/started/paused/resumed/cancel_requested/cancelled/completed; action.planned/started/finished/blocked; state.discovered; transition.recorded; approval.requested/decided; notification.detected/timed_out; network.failed; console.error; visual.compared; model.usage; test.proposed/executed; report.generated.

SSE consumers reconnect using Last-Event-ID or a run-scoped sequence cursor and obtain a full backfill. UI reconstruction from events must **never cause browser actions to repeat**.

## 4. Discovery model and business-process mapping

At each meaningful boundary, the explorer records:

- Normalized route/URL and title; accessible landmarks, headings and interactive control roles/names.
- Role/identity, dialogs/modals, selected navigation, meaningful form and validation state.
- Selected tab/filter and relevant non-secret business-data markers.
- Redacted screenshot, action result, network anomalies, console issues and timing.

Define semantic state fingerprint as a **versioned canonical combination** of route, role, relevant UI markers, workflow state, dialog, navigation and app-specific discriminator. Raw DOM hashes alone are unsuitable because timestamps and randomized IDs generate excessive false states. When ambiguous, record UNCONFIRMED and allow reversible operator merge/split.

Every attempted action becomes a graph edge, including failed and policy-blocked actions. Edges store source/target states, action, outcome, preconditions, side effects, risk, confidence, locator provenance and evidence. Journeys are paths labelled observed, hypothesized, human-reviewed or verified by deterministic testing. UI labels alone cannot prove business rules.

Planner modes:

- **Goal-directed exploration:** e.g., discover login and authenticated dashboard.
- **Bounded monkey exploration:** seeded randomized allowed clicks, typing, toggles and navigation, with strict step/time/depth budgets.
- **Risk-driven exploration:** prioritize unvisited and error-prone transitions.
- **Negative/boundary exploration:** invalid inputs, forms, dialogs, failures and denied operations.

Candidate next actions are stored and displayed with rationale, confidence, estimated cost and risk. Operator may pause, prioritize or skip candidates. Detect cycles, near-duplicate states and low-information edges.

**Coverage is relative to scope:** show visited / known reachable states, explored / known transitions, and verified / explicitly declared requirements. Label unknown reachable behavior as unknown; do not present discovered graph coverage as total application coverage.

## 5. Agent tools, constraints and OpenAI interaction

Allowed initial tool capabilities: observe_state, navigate, click, type, select, scroll, wait_for, inspect_network, inspect_console, capture_screenshot, authenticate_as(identity_ref), request_approval, record_observation, propose_test and propose_finding.

The planner receives **structured, filtered observations**, not arbitrary raw credentials. Tool calls have typed schemas, stable IDs, timeouts, per-step budgets and policy evaluation. The explorer has **no shell, unrestricted HTTP, filesystem writes, Git push, payment credentials, database access or policy-editing tool**.

### 5.1 Prompt injection defense

HTML, text, screenshots, metadata, comments, links, third-party widgets and downloaded files from the target application are untrusted *data*, even if they contain commands. Never promote page text to system instructions. Validate navigation targets, redirects, resolved IP ranges and forbidden hosts at the egress boundary; reduce DNS rebinding/SSRF risk. Record blocked instructions as evidence. Network inspection must not become another route for exfiltration.

### 5.2 Model and budget policy

Model IDs are configurable per run/workflow (planning, visual interpretation, coding and reflection). Record model ID, prompt version, request count, input/output tokens, token estimates, actual or estimated cost and latency. Enforce hard caps on steps, calls, elapsed time, screenshots and total cost. Malformed responses retry only within budget and then become INCONCLUSIVE; never assume a failed model request yielded a successful action.

## 6. Credentials, authorizations and approvals

Secrets: OpenAI API key, target login credentials, OAuth/client secrets, signing keys, DB passwords, browser cookies and authentication storage. Place them **outside Git** in protected local mounted secret files or Docker-managed secrets; a more capable local vault can be added later. Files are accessed only by required trusted processes. Sensitive persisted sessions must be encrypted and short-lived; do not include credentials in CLI arguments, logs, screenshots, telemetry or prompts.

The trusted credential broker resolves an identity reference and performs login without exposing the secret to the agent. For OTP/MFA, pause for assisted authentication; do not invent automated bypass. A compromised browser worker remains a security risk; minimize injected secret lifetime, restrict origins and use dedicated least-privilege test accounts.

Capabilities: browse.read, form.fill, testdata.create/update/delete, sandbox.booking.submit, sandbox.order.submit, external.message.send, financial.transaction, production.mutate. Last two **DENY in MVP**. Each call evaluates actor, environment, origin/IP, scope, identity, exact parameters, data classification, side effects, step/cost budget and required approval.

The dashboard may show capability toggles, but a checkbox merely requests a policy; it does not override independent gateway enforcement.

Approval records must show exact operation, destination, identity, payload summary, amount/currency when applicable, risks, expiration and approver. Bind approval to a canonical hash of run/step/action/parameters; consume once; reject parameter changes or expired grants. Denials remain recorded. Writes that can send emails, issue tickets, place orders, create external events or charge cards need explicit side-effect classification, not a generic 'test' exemption.

### 6.1 External model data boundary

OpenAI API calls leave the local Docker host. Redact identifiers, authorization headers, secrets, personal/financial fields and potentially sensitive screenshots **before** model egress. Send only relevant sanitized DOM summaries or visual crops. Maintain configurable policy and audit of outbound payload classification. Show operators that external processing is enabled. Review API retention/legal requirements before using non-public customer data.

## 7. Browser instrumentation, screenshots and notifications

### 7.1 Evidence streams

Capture Playwright actions and trace ZIP, console errors/warnings, uncaught exceptions, selected request/response metadata, failed requests, navigation redirects, event timeouts, DOM/ARIA summary and screenshots. Do not capture response bodies or authorization headers by default. Attach exact step/state IDs and artifact hashes. Retain sensitivity tags and access checks for every file. A missing trace is reported as unavailable rather than silently omitted.

### 7.2 Visual-regression pipeline

Reproducible screenshot comparison requires pinning:

- Chromium/browser build, viewport size, DPR and device emulation.
- Locale, timezone, color scheme, fonts, animations and reduced-motion preferences.
- Auth role, state fingerprint, fixture data and reset/seed identifiers.
- Stable ready condition (network/UI readiness, web fonts, loading indicators) instead of fixed arbitrary sleeps.
- Screenshot scope (viewport/full-page/component), mask definitions and baseline revision.

Pipeline: capture controlled checkpoint → select approved versioned baseline → compute deterministic pixel/structural difference → produce before/after/diff images and metrics → attach machine facts → optionally ask visual model for explanation. **The LLM cannot set the pass threshold or approve a new baseline.** Human review accepts deliberate redesigns. Store baseline metadata, reviewer, change reason and compare run ID.

Screenshot pixels alone cannot reliably establish notification semantics or functional correctness. Separate visual checks from DOM/event assertions and network/console error checks. Some visual changes are intentional, and some errors produce no visible change.

### 7.3 Notification/toast observer

Observe toast, alert, dialog and status regions around actions. Capture redacted text, accessible role/live-region, appearance timestamp, disappearance/timeout, overlapping notifications and obstruction of other controls. To mark an assertion failed, the system must have an **explicit expectation and timeout**. Otherwise record discovery/hypothesis, not a confirmed bug. Take event-driven screenshots to catch transient UI.

### 7.4 Tested-site accessibility versus dashboard accessibility

The dashboard itself targets **WCAG 2.2 AA**. The explorer can test target-site accessibility with axe-core, accessible role/name checks, keyboard/focus behavior and contrast observations, but automated scans alone cannot certify full WCAG compliance. Store rule ID, affected element and reproducible steps for each observation; keep manual checks available.

## 8. Test factory — separate coding workflow

Input: evidence-backed journey with route/state IDs, allowed role, fixture, preconditions, oracle/expected outcome and test style rules. The coding agent creates a **reviewable draft** Playwright Test (TypeScript), not a trusted test by default.

Requirements:

- Arrange / act / assert, test isolation and cleanup.
- Prefer role/name or documented test IDs over brittle nth-child selectors.
- No hardcoded tokens, passwords, session blobs or confidential customer details.
- Separate positive, negative, permissions and boundary scenarios.
- Explicit wait conditions and assertions; no sleep-based guessed success.
- Linters, typecheck, security checks and isolated execution.
- Trace/screenshots/test result, failure logs and repeated-run flakiness data.
- Dedicated branch/patch and human review before promotion.
- No explorer repository write rights and no automatic snapshots/baseline updates.

Status: DISCOVERED → PROPOSED → REVIEWING → GENERATED → EXECUTED → APPROVED / REJECTED / NEEDS_ORACLE. Retain traceable links back to journey, observation, model ID, code revision and reviewer.

## 9. Memory and 'dreaming'

Begin with **structured PostgreSQL memory**, not a mandatory vector DB.

- Semantic: verified app states, routes, roles, restrictions and reviewed journeys.
- Episodic: runs, action outcomes, console/network/visual evidence and historical failures.
- Procedural: confirmed exploration strategies, tool templates, prior test patterns and their versions.

A scheduled/on-demand **reflection job** ingests bounded run summaries and evidence refs, groups recurring failures, identifies missing paths and recommends the next actions/tests. Suggestions have citations and confidence and require human review for anything promoted. Reflection cannot modify policy, expected assertions, baselines, permissions or completed reports.

## 10. Human dashboard: pages and behavior

Primary navigation:

1. **Overview:** health, current runs, blocked approvals, recent quality gates and spend.
2. **Live Explorer:** current state, screenshot/browser view, real-time timeline, now/next queue, controls and cost.
3. **Application Map:** state/transition graph, journey filters, table alternative and detailed evidence.
4. **Findings & Visuals:** confirmed/suspected issues, before/after/diff overlays, toasts, console and network diagnostics.
5. **Test Factory:** evidence-backed candidates, requirements/oracles, generated code, executions and review.
6. **Runs & Reports:** history, trends, executive gate, detailed report and exports.
7. **Policies & Settings:** allowed environments, identity refs, permissions, approvals, models, retention, budgets.

### 10.1 Live Explorer layout

- Header: project, target, environment, actor/role, run ID, status, duration, remaining budget, Pause/Resume/Cancel.
- Primary map: current state distinctly marked, attempted/success/failed/blocked edges, search/filters, linked table.
- Timeline: per-step time, rationale, action/tool, result, duration, evidence and trace links.
- Now/Next: exact current tool call, prioritized candidate frontier, risk/cost, blocked/approval queue.
- Inspector: screenshot, DOM/ARIA markers, console errors, request summary, Playwright trace and Langfuse trace.
- Health and finance: model tokens, estimated API spend, hard budget, worker/queue status and last observed event time.

When streaming disconnects, show **STALE** with last event time. On reconnect, replay stored events, not a fabricated snapshot of activity. Support keyboard actions and clear confirmation/cancellation of sensitive commands.

### 10.2 Application Map semantics

Node: state label, URL/route, role, observed/verified status, current marker, coverage, finding count. Edge: intended action, outcome, risk and test coverage. Node selection opens accessible details with evidence links. Provide search, filtered journeys, fit-to-view, legend and history.

An **accessible table with functional parity** is mandatory: sortable states with status/role/route/findings; transition details and same open/inspect/steer functions. No state facts should require hovering or dragging a graph.

### 10.3 WCAG 2.2 Level AA acceptance

- Semantic landmarks and headings, accessible names, clear labels/instructions and errors.
- All controls keyboard operable; visible focus, logical order, escape/close behavior, no traps.
- Status announcements via appropriate live regions, **throttled** to avoid overwhelming readers.
- Reduced motion, pause moving updates, no color-only encoding or flashing.
- Text contrast and non-text contrast; 200% zoom, responsive reflow at 320 CSS px.
- Graph/diffs/charts have accessible table/text alternatives; controls do not depend solely on dragging.
- Meet applicable WCAG 2.2 target size, consistent help and redundant-entry guidance.
- Automated axe-core + Playwright tests on all critical workflows and manual keyboard/screen-reader audit. Passing automation alone is insufficient.

Design states explicitly: loading, empty, no run, paused, disconnected, stale, unauthorized, approval pending, missing evidence, incomplete report, exhausted budget and cancelled run.

## 11. API and live-event contract

Versioned authenticated REST endpoints (illustrative):

    POST /api/v1/projects/{project_id}/runs
    GET  /api/v1/runs/{run_id}
    POST /api/v1/runs/{run_id}/pause
    POST /api/v1/runs/{run_id}/resume
    POST /api/v1/runs/{run_id}/cancel
    POST /api/v1/runs/{run_id}/steer
    GET  /api/v1/runs/{run_id}/events?after_sequence=...
    GET  /api/v1/runs/{run_id}/events/stream?after_sequence=...
    GET  /api/v1/runs/{run_id}/graph
    GET  /api/v1/runs/{run_id}/steps
    GET  /api/v1/runs/{run_id}/findings
    GET  /api/v1/runs/{run_id}/report
    GET  /api/v1/artifacts/{artifact_id}
    POST /api/v1/approvals/{approval_id}/decision
    POST /api/v1/test-proposals
    POST /api/v1/test-proposals/{id}/generate
    POST /api/v1/test-proposals/{id}/review
    GET  /api/v1/metrics/cost

Use UUID/ULID IDs, Pydantic validation, server-side authorization, CSRF protection for cookie-auth writes, idempotency keys, optimistic concurrency, bounded pagination and correlation IDs. SSE supports cursor replay and scoped authorization. Artifact links are local, short-lived and subject to access checks. API must not expose credential values or raw browser auth storage.

## 12. Persistence, reliability and observability

Suggested tables: projects, environments, identities, policy_versions, runs, run_steps, run_events, run_checkpoints, app_states, app_transitions, journeys, observations, findings, approvals, artifacts, visual_baselines, visual_comparisons, notifications, test_proposals, test_executions, reflection_jobs, reports, model_usage, outbox, audit_log. Include timestamps, indexes, actor provenance, artifact hashes and schema versions.

Queue tasks: start_run, browser_step, generate_test, execute_test, build_report, reflect_on_runs, prune_artifacts. Workers use leases/fencing and strict per-run concurrency. Retry safe reads with backoff; never blindly retry uncertain writes. Worker failure yields INTERRUPTED/INCONCLUSIVE, not invented success.

Three correlated observability layers:

1. Structured app/worker logs and health metrics.
2. Langfuse spans for model decisions and token/cost information.
3. Browser Playwright traces, screenshot/network/console evidence.

Shared correlation IDs join a model decision to tool call, policy decision, UI action and report finding. Apply redaction **before** OpenAI egress, logging and Langfuse trace storage; verify using synthetic secret canaries.

Configurable initial retention proposal: run metadata ~90 days; screenshot/trace evidence ~30 days; ephemeral session state for the minimal required lifetime. Encrypt backups, test restoration, purge stale files and keep foreign-key references consistent.

## 13. Report per run and deterministic quality gate

Report opens with an **executive summary**: scope, start/finish, environment/role/browser, overall gate, number of required checks and their outcomes, confirmed critical/high issues, suspected issues, visual changes, coverage denominator, test proposals, policy blocks and model cost.

Progressive drill-down: required-check table → journeys and state graph → findings by severity → notification timing → before/after/diff images → console/network diagnostics → candidate/approved tests → policies/approvals → limitations → trace links and export.

Deliver accessible local HTML and Markdown/JSON export. Include immutable versions of policy, schema, prompt/model, baseline, required test suite and artifact hashes. The report is reproducible from stored evidence **without another model call**.

### 13.1 Gate precedence

**FAIL:** any confirmed blocking functional, visual or accessibility regression, unauthorized side effect, sensitive data leak or policy violation according to release policy.

**INCONCLUSIVE:** otherwise, if any *required* check was unexecuted, interrupted, blocked, cancelled, lacks an oracle or lacks evidence.

**PASS:** only when all declared required checks passed with required evidence and no blocking issue.

Default severity policy: Critical/High block; Medium/Low inform. Unsupported LLM hypotheses do not automatically FAIL. Discovery-only runs without a declared required test oracle normally show **INCONCLUSIVE for release**, while reporting useful discoveries. A visual difference awaits baseline review unless it violates a required checkpoint under explicit criteria.

Example (illustrative, not actual results):

    Quality gate: INCONCLUSIVE
    Required checks: 6 passed, 2 not run
    Confirmed blocking issues: 0
    Suspected issues needing review: 2
    Visual changes requiring review: 1
    Explored: 8 / 12 currently known states (total app coverage unknown)
    Policy-blocked actions: 1
    Estimated OpenAI usage: USD 0.42
    Reason: required journey/check not completed

The gate must distinguish FAIL, NOT RUN, NOT APPLICABLE, SUSPECTED and INCONCLUSIVE at individual check level.

## 14. Acceptance scenarios

| ID | Fixture scenario | Required behavior |
|---|---|---|
| A-01 | Start authorized fixture run | QUEUED → EXPLORING with durable events |
| A-02 | Dashboard disconnect/reload | Complete event replay without duplicate actions |
| A-03 | Same route with distinct modal/role state | Two semantic nodes; correct edge |
| A-04 | Malicious instructions in page text | Untrusted; policy prevents prohibited action |
| A-05 | Forbidden write attempt | Denied or explicitly approved; logged, no unauthorized effect |
| A-06 | Authorized test-data creation | Once only, with evidence and cleanup |
| A-07 | Worker crash after submit | No automatic duplicate submit; reconcile |
| A-08 | Secret canary in headers/DOM/console | Absent from prompts, logs, traces and exports |
| A-09 | Controlled UI layout changed | Deterministic visual diff; human baseline review |
| A-10 | Expected toast never appears | Timeout failure with evidence and oracle |
| A-11 | Login journey discovered | Reviewable Playwright draft and separate execution |
| A-12 | Required test interrupted | INCONCLUSIVE, not PASS |
| A-13 | Confirmed high-severity regression | FAIL under default policy |
| A-14 | Keyboard and screen-reader-only operator | All critical map, table, approvals and controls usable |
| A-15 | Redirect to unapproved/internal host | Block and record |
| A-16 | Run budget exhausted | Bounded termination with report explanation |
| A-17 | Reflection over completed runs | Evidence-linked recommendations only, no self-promotion |

## 15. Delivery slices

1. Foundation: docs, Compose, fixture app, auth, secrets template, threat model and accessibility scaffolding.
2. Event loop: FastAPI/SSE/Postgres, one restricted browser action and basic dashboard.
3. Map: normalized state graph, planned queue, pause/resume/cancel, table parity, budgets.
4. Visual/reporting: controlled captures, notifications, baselines, diffs, report worker and gate.
5. Test factory: proposal → reviewed Playwright draft → isolated execution → approval.
6. Reflection and hardening: retained memory, dreaming, scheduling, threat tests, backup and restores.

Each slice must have a runnable end-to-end demo and verifiable acceptance criteria.

## 16. Open decisions (non-blocking)

- First authorized target and authentication: use a **local fixture** until selected.
- Browser matrix: start one pinned Chromium build and desktop viewport.
- Model and budgets: configurable OpenAI model ID and explicit spend/call/step limits.
- Business expected outcomes: only confirmed or reviewed oracles can block release.
- Snapshot thresholds/masks and baseline checkpoints: selected through review.
- Host LAN access and remote site allowlist: configure explicitly.
- Generated test publishing: initial local patch; GitHub branch integration after sandboxed authoring security tests.

## 17. References

- WCAG 2.2: https://www.w3.org/TR/WCAG22/
- WAI-ARIA Authoring Practices: https://www.w3.org/WAI/ARIA/apg/
- Playwright traces: https://playwright.dev/docs/trace-viewer
- Playwright screenshots: https://playwright.dev/docs/test-snapshots
- Playwright accessibility: https://playwright.dev/docs/accessibility-testing
- LangGraph interrupts: https://docs.langchain.com/oss/python/langgraph/interrupts
- LangGraph persistence: https://docs.langchain.com/oss/python/langgraph/persistence
- React Flow: https://reactflow.dev/
- OpenAI API data controls: https://developers.openai.com/api/docs/guides/your-data

## 18. TestSprite-informed product intent, failure triage and UX contracts (2026-11-21)

Companion: [TestSprite 4.0 benchmark](./testsprite-benchmark.md). These are **proposed implementation requirements**, not features already built. The external reference was released 2026-11-20 and describes brief-driven verification, editable feature maps, exploration/replay, triage and controlled test maintenance. Our runtime remains local Docker plus outbound OpenAI API.

### 18.1 Two-map architecture and oracle ownership

Maintain **separate canonical models**, cross-linked through evidence:

1. **Intent Map** (normative): project briefs, requirements, roles, acceptance criteria, reviewed/approved oracles, feature dependencies, assumptions, version history, reviewers and upstream source locations. A requirement's expected outcome cannot be inferred from current observed behavior and then treated as approved.
2. **App State Map** (observational): discovered states, role-specific conditions, observed transitions, network/API outcomes, screenshot/DOM/console artifacts and action histories. An observation may inform a question or proposal; it is not proof of correct business behavior.
3. **Coverage links**: Requirement ↔ Journey ↔ AppState/Transition ↔ TestCase/Execution ↔ Finding/Report. Each link has provenance, run/correlation ID, version and a status such as PROPOSED / VERIFIED / CONTRADICTED / NOT_REACHED.

A feature in a product brief can be **UNREVIEWED, PROPOSED, APPROVED, REJECTED or SUPERSEDED**. Edits to an approved requirement create a new revision and require reason, reviewer and explicit effective version. Store approved test expectations independent of generated code and of any test-healing component.

**Truth table (minimum):**

| Approved expected result? | Observed execution? | Allowed interpretation |
|---|---|---|
| No | Yes | DISCOVERED / NEEDS_ORACLE; never declare release PASS |
| Yes | No | NOT_EXECUTED; INCONCLUSIVE if required |
| Yes | Yes, matches | VERIFIED/PASS for that check with reproducible evidence |
| Yes | Yes, contradicts | FAILURE candidate, corroborate and classify before release gate |
| No | No | OUT_OF_SCOPE / UNKNOWN, never silently counted as covered |

### 18.2 Project brief, assumptions and approved test plan

- Input may include a plain-language goal, target URL, selected test environment/identity ref and optional **operator-supplied** PRD/ticket/design document; document parsing and external integrations are later phases. Never upload raw credentials or session state.
- Model output is a structured **proposed** feature map with named roles, preconditions, expected outcomes, risk, source snippets/refs and explicit unanswered questions; no auto-approval.
- Assumption record: id, text, discovered_by, source, affected requirements, impact, proposed answer, decision status, actor, timestamp and review version. The dashboard provides Approve/Revise/Reject actions.
- Proposed test plans: test IDs, linked requirement versions, deterministic oracle, preconditions, fixture/setup/cleanup, roles, risk, required/optional flag, expected evidence and review state. Operator approves a plan before it becomes the release-required check set.
- **No oracle laundering:** source code, current DOM, API response, prior passed run or AI confidence cannot silently replace approved expectations. For undocumented behavior, the agent may perform exploratory probes and ask questions, not produce a release PASS.

### 18.3 Failure taxonomy and reproducible evidence bundle

Maintain separate axes: **execution outcome** (PASS/FAIL/NOT_RUN/BLOCKED/INTERRUPTED), **triage class** (PRODUCT_DEFECT, TEST_FRAGILITY, ENVIRONMENT_FAILURE, UNKNOWN) and **review confidence/status** (SUSPECTED, CONFIRMED, DISMISSED). Do not conflate them.

Per failure, produce a versioned bundle:
- Project/run/step/test/requirement IDs and approved oracle version; expected versus actual;
- Failing step and adjacent steps, before/after state IDs, normalized DOM/ARIA snapshot and screenshot/diff;
- Playwright trace and optional recorded video, accessible timestamped text replay, relevant redacted console and network entries;
- Browser/viewport/locale/timezone, fixture, auth role, app revision (if known), worker and model versions;
- Repeatability/retries, suspected cause with confidence, suggested fix, classification decision and reviewer provenance;
- Local evidence links with access checks, artifact hashes and retention policy.

A model-suggested "root cause" is an **explicit hypothesis** unless deterministically corroborated. An environment outage is not automatically an application defect; a failed required check may still make the release gate INCONCLUSIVE. Confirmed critical/high product regressions fail per existing section 13.

### 18.4 Test self-healing boundary (post-MVP)

Permitted **proposal-only** repairs: selector strategy, stable waits, navigation transitions, fixture cleanup when consistent with reviewed setup and approved policy. Preserve the **same approved test oracle, requirement version, assertion semantics and negative-test intent**.

Forbidden automatic repair: changing expected text/status/data, dropping assertions, expanding domain/action permissions, creating new test accounts without policy, accepting new visual baselines, masking a real defect, auto-merging to protected branches.

Review process: proposed source diff → static/policy check → rerun against unchanged oracle → compare before/after evidence and flakiness → human review → optional promotion. If the original expectation fails after the mechanical repair, report the product failure; do not self-certify success.

### 18.5 Visual and semantic UX inspection

In addition to deterministic pixel/screenshot comparison (section 7), observe:
- Toast/notification presence, disappearance timing, accessibility announcement and overlap/occlusion.
- Consistency of action labels, dates/currency formats, focus and interactive semantics across states.
- Empty-state accuracy, user feedback after success/failure, destructive confirmation clarity, disabled/loading states, responsive overflow and focus restoration.
- Reference design mismatch (if explicitly supplied) marked **review-required** unless approved deterministic criteria exist.

AI visual comments create reviewable observations, **not automatically confirmed bugs or legal accessibility verdicts**. Show source screenshot, current screenshot, diff mask, test device and all uncertainty. Redact screenshots before external model calls; if content cannot be safely redacted, do not use external vision inference.

### 18.6 Dashboard acceptance / WCAG 2.2 AA amendments

- Distinct **Intent Map** and **Observed App Map** with relationship overlay; each has a fully functional sortable/filterable table/list alternative and equal review/inspect controls.
- A brief wizard with accessible labels, validation and clear boundary warnings before any approved test data modifications.
- Requirement details show version, source, review state, proposed test plan, observed evidence and unresolved assumptions.
- Run replay has an equivalent **ordered text step transcript** with timestamps and access-controlled artifact links; video alone does not establish accessibility.
- Findings show expected/actual and triage status with text labels rather than color only; status updates are announced without flooding screen readers.
- All approvals and changes to approved oracles use explicit confirmation, visible reviewer identity and persistent audit details.

### 18.7 Data, events and API deltas

Additional tables: project_briefs, requirements, requirement_sources, requirement_revisions, requirement_approvals, feature_dependencies, assumptions, test_plans, test_plan_items, requirement_journey_links, requirement_execution_links, failure_triage, healing_proposals.

Versioned events: brief.created/updated; requirement.proposed/approved/revised; assumption.opened/resolved; test_plan.proposed/approved; requirement.journey_linked; failure.triaged; healing.proposed/executed/approved/rejected. Keep original step correlation and evidence references.

Versioned endpoint additions (illustrative and subject to threat review):

    POST /api/v1/projects/{project_id}/briefs
    GET  /api/v1/projects/{project_id}/requirements
    POST /api/v1/projects/{project_id}/requirements/proposals
    POST /api/v1/requirements/{id}/reviews
    GET  /api/v1/projects/{project_id}/assumptions
    POST /api/v1/assumptions/{id}/decision
    GET  /api/v1/projects/{project_id}/traceability
    POST /api/v1/projects/{project_id}/test-plans
    POST /api/v1/test-plans/{id}/review
    GET  /api/v1/runs/{run_id}/failure-bundles
    POST /api/v1/failures/{id}/triage

Approvals require server-side actor authorization, optimistic version checks and recorded reasons; never rely only on client UI. Existing SSE replay, idempotency and audit requirements apply.

### 18.8 Additional acceptance tests

Reference benchmark scenarios **B-01 to B-10**. Highest-value release checks:
- Inferred but unapproved requirement **cannot** make a required test PASS;
- Same route but different modal/business state remains visible in App State Map;
- Approved business expectation fails: do not turn that incorrect output into a new assertion;
- Missing target/API or missing browser produces ENVIRONMENT_FAILURE/INCONCLUSIVE, not an invented product bug;
- A proposed selector repair leaves assertion/oracle unchanged and shows original and repaired trace;
- Both maps and replay support keyboard + screen reader; no silent external secret egress.
