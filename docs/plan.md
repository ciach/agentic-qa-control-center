# Agentic QA Control Center — Delivery Plan

> **Version:** 0.1 · **Date:** 2026-11-21  
> **Status:** Proposed phased plan; no application implementation yet  
> **Related:** [Intent](./intent.md) · [Specification](./spec.md) · [Architecture review](./architecture-review.md)

## 1. Goal and delivery philosophy

Deliver a **local Docker Compose application** that uses the **OpenAI API** to explore an authorized web application, shows an accessible live map of discovered states and actions, generates reviewable Playwright tests, detects visual and behavioral changes and produces evidence-backed reports with a release gate.

### Guiding rules

- **Vertical slices first.** Each milestone must run end-to-end with evidence and acceptance checks.
- **Dashboard is first-class.** A worker without live visibility and control is not a completed milestone.
- **Separate probabilistic exploration from deterministic testing.**
- **Policy outside the model.** Authorizations and egress restrictions must remain effective even under prompt injection.
- **Unknown is not passed.** If the evidence or required coverage is missing, the gate is INCONCLUSIVE.
- **Keep source-code access optional.** The pilot is black-box by design.
- **Protect secrets from models, logs, traces and the public Git repository.**
- **Treat the accessibility requirement as deliverable work**, not a final polish phase.

## 2. Decision register

| Domain | Decision | Rationale |
|---|---|---|
| Hosting | Local Docker Compose only | Confirmed constraint; simple, reproducible operations |
| External inference | OpenAI API | Confirmed; data sent to models leaves the local machine |
| Orchestration | Python, FastAPI, LangGraph | Durable graph execution, human-in-loop and checkpointing |
| Browser/test tools | TypeScript, Playwright, Playwright MCP | Reliable browser automation; deterministic test runner |
| Frontend | React, TypeScript, Vite | Maintainable local UI |
| Exploration map | React Flow + an equivalent accessible table | Dynamic state graph without excluding keyboard/screen-reader users |
| Events | PostgreSQL event journal + SSE | Reliable event replay and low-complexity dashboard streaming |
| Persistent state | PostgreSQL | Events, graph, runs, approvals, audits and learned knowledge |
| Queue | Redis + Dramatiq | Task handling, leases and bounded retries |
| Artifacts | Local MinIO/S3-compatible storage | Screenshots, traces, diff images, reports |
| Observability | Langfuse and Playwright traces | Model costs/decisions + browser reproducibility |
| Secrets | Docker-mounted secrets initially; broker abstraction | No raw credentials in AI context or repository |
| Visual testing | Controlled Playwright screenshots and versioned baselines | Deterministic signal separate from AI interpretation |
| Accessibility | WCAG 2.2 Level AA | Includes interaction, screen-reader support and testing |
| GitHub | Public ciach/agentic-qa-control-center, approved by owner | Docs and code only, no operational secrets |

## 3. Delivery roadmap

**Milestone order matters.** Do not implement a complex autonomous agent before the event stream, security gateway and dashboard exist.

### M0 — Repository and architecture baseline

Deliverables:

- README, intent, plan, spec and independent-review TODO.
- Monorepo structure and code ownership boundaries.
- Example env file with no live credentials; .gitignore; Dependabot/security scanning as available.
- Docker Compose skeleton and local onboarding instructions.
- CI jobs for TypeScript/Python lint, unit tests and accessibility tests.
- ADRs for trust boundary, graph schema, report gates, baselines and model data egress.
- Safe local fixture application with login, form, toast, modal and deliberate failure states.

Exit criteria: docs versioned, repo contains no secrets, a clean local setup runs, and a test fixture can be opened. **No claim of QA functionality yet.**

### M1 — Single browser journey and durable event stream

Deliverables:

- Authenticated local dashboard (one operator).
- FastAPI endpoints to create/view runs, durable lifecycle state and a replayable SSE stream.
- Trusted browser worker with one isolated Chromium context, approved target origin and safe fixture test user.
- Browser action events: planned, started, completed/failed/blocked.
- Current action, action history, stale/disconnected indication and pause/cancel semantics.
- Screenshots and browser traces linked to each run.
- OpenAI API client with a model configuration, token accounting, per-run ceilings and redacted prompts.
- Automatic recovery or explicit INCONCLUSIVE result after worker crash.

Exit criteria: watch a supervised run follow a fixture journey; refresh UI and recover its full event history; cancel safely; inspect browser evidence.

### M2 — Real state graph, accessible exploration map and live controls

Deliverables:

- Normalized route/UI/role/workflow fingerprint schema.
- Nodes for observed states and edges for actions, with provenance and timestamps.
- Graph view with current node and planned frontier highlighted.
- Equivalent sortable/filterable table view with identical selection/actions.
- Planned, running, blocked, completed and skipped action queue.
- Operators can pause, resume, cancel, inspect a node, inspect edges and adjust future priority.
- Resume durability and worker lease/fencing tests.
- Navigation and interface keyboard tests.

Exit criteria: a run discovers multiple states (including modal state on the same URL), and the accessible map/table plus event timeline stays in sync after reconnect.

### M3 — Guided discovery + bounded monkey exploration

Deliverables:

- Restricted tool catalog through a policy gateway (navigate/click/type/inspect/screenshot).
- Exploration strategies: breadth-first candidate discovery, goal-driven paths, negative/boundary checks and seeded random actions.
- Discovery budget per run (steps, depth, duration, tokens, estimated cost).
- Support for test-data create/update only with environment/identity/action scopes; deny production/real purchases.
- Prompt-injection tests and network/DNS/redirect restrictions.
- Proposed business journeys and hypotheses with confidence and supporting evidence.
- De-duplication of states/transitions and coverage denominator shown explicitly.

Exit criteria: discovers new branches without looping forever, correctly refuses forbidden actions and distinguishes observed behavior from unverified business expectations.

### M4 — Visual quality, notifications and accessible reporting

Deliverables:

- Captures screenshot checkpoints with pinned browser, viewport, locale, timezone, fonts, stable test data and ready condition.
- Versioned visual baselines, diff images, tolerances, metadata and explicit human baseline review.
- Toast/notification observation with visible text, timing, disappearance and location metadata.
- Console error and network-failure grouping with URL/headers redacted.
- Per-run HTML dashboard report and downloadable Markdown/JSON summaries.
- Executive outcome PASS/FAIL/INCONCLUSIVE computed from declared required checks, coverage and severity policy.
- Evidence links, limitations, cost ledger, run comparisons and review actions.

Exit criteria: deliberate UI alteration is flagged with diff; a transient toast is recorded; missing oracle leads to INCONCLUSIVE; a readable report links to evidence.

### M5 — Test factory (separate coding workflow)

Deliverables:

- Candidate test proposals based on proven state transitions and journeys.
- Separate coding worker with restricted repository scope and no browser credential access.
- Playwright TypeScript test template with fixtures, roles, stable selectors and assertions.
- Static checks, syntax/type checks, test retries with flakiness classification and report links.
- Draft branch or patch with human review and explicit test promotion.
- Never auto-update snapshots, mutate approved assertions or push to protected main.

Exit criteria: discovery of a login journey produces a draft Playwright test; it runs against the fixture; a human can review/promote it and reproduce it.

### M6 — Reflection, historical trends, scheduling and hardening

Deliverables:

- Structured semantic, episodic and procedural memories with evidence citations.
- Controlled 'dreaming' run that groups findings and proposes next exploration/test priorities.
- Repeated run comparisons, regression trends and budget trend.
- Manual and scheduled execution triggers with queueing and deduplication.
- Retention controls, audit, sensitive-data redaction tests, backup and restore.
- Operator runbooks; fault-injection, security, A11Y and reproducibility testing.
- Broader browser/identity/role support only if validated by the pilot.

Exit criteria: reflection recommendations are auditable and never silently change policies, baselines or approved tests; history and reports remain accessible after restarts.

## 4. Cross-cutting workstreams

### 4.1 Backend and data

- Versioned SQL migrations, typed API models and OpenAPI definitions.
- Immutable event journal with per-run sequence numbers and correlation IDs.
- Idempotent message handling and worker leases.
- State graph stable fingerprint revisions and reversible merges.
- Secrets isolated from agents and telemetry.
- Structured errors, audit records and artifact integrity hashes.

### 4.2 UI and accessibility

- WCAG 2.2 AA design system with Radix-style accessible primitives.
- Responsive dashboard: map, timeline, action queue, approvals, costs, evidence.
- Equivalent table alternative to graph; focus indicators, skip links, descriptive labels.
- Accessible status announcements and keyboard-only functionality.
- Automated axe-core and Playwright checks plus a manual screen-reader matrix.
- No color-only indicators; accessible visual diff descriptions and images.

### 4.3 Security and privacy

- Least-privilege tool broker, domain/IP restrictions, SSRF and redirects policy.
- Test-only identities; credential broker and separate session storage.
- Origin, action and payload allowlists; per-operation authorization logged.
- Approval before risky actions and no automatic retries for uncertain writes.
- OpenAI-bound context filtering and log/trace redaction; canary secret test.
- Rootless/restricted browser containers, capped resources, no host socket mounts.
- Sensitive artifacts encrypted, access-controlled and deleted by retention policies.

### 4.4 Agent quality and evaluation

- Fixture scenarios for normal login, error toast, broken network, modal, validation and misleading instructions.
- Golden event traces and expected state graph snapshots.
- Reproducibility rate, false positives, hallucination/unsupported-claim rate.
- Model-cost accounting, exploration breadth, repeated paths and budget breaches.
- Deterministic test quality; flakiness and assertion validity measured separately.

## 5. Functional backlog (initial epics)

| Epic | First useful capability | Priority |
|---|---|---|
| E01 — Local infrastructure | Compose up, health endpoints, fixture, local auth | P0 |
| E02 — Run lifecycle | Create, start, pause, resume, cancel, replay | P0 |
| E03 — Live control center | Status, current action, timeline, queue | P0 |
| E04 — Policy gateway | Allowed origins/actions, audit, denied operations | P0 |
| E05 — Evidence capture | Screenshots, traces, network and console records | P0 |
| E06 — Semantic state graph | Unique state nodes + action edges + unknowns | P0 |
| E07 — Accessible UI | WCAG 2.2 AA, map table alternative, keyboard | P0 |
| E08 — Autonomous explorer | Bounded candidate exploration + budgets | P1 |
| E09 — Safe writes | Test-data operations + approvals + idempotency | P1 |
| E10 — Visual regression | Reproducible captures, diff review and baselines | P1 |
| E11 — Notification observation | Toast detection + timing + comparison | P1 |
| E12 — Findings | Evidence-backed hypothesis, confirmed defect | P1 |
| E13 — Run report | Executive gate, evidence and cost drill-down | P1 |
| E14 — Test factory | Draft Playwright tests from observed journeys | P1 |
| E15 — Learning | Reflection with citation-backed suggestions | P2 |
| E16 — Scheduling | Queued run triggers and historical comparisons | P2 |

## 6. Definition of Done for every feature

- Requirements and error/empty/loading states documented.
- Unit/integration tests and a fixture-driven happy/negative path.
- Policy decision + audit event for every privileged action.
- Accessibility validation (keyboard, focus, semantics; axe as applicable).
- Telemetry and relevant event types for dashboard visibility.
- No credentials in model payloads, screenshots or logs where avoidable.
- Regression scenario demonstrates failure then verified repair.
- User-visible result is accurate (no premature success).
- Docs and migration updated; feature linked to an acceptance test.

## 7. MVP end-to-end demo / acceptance script

1. Start Compose; open the local dashboard and authenticate.
2. Define a fixture application with allowlisted hostname and test user.
3. Start a run with explicit step/time/token/cost budgets.
4. See the agent inspect login, plan and execute allowed interactions.
5. Watch state map, live status, timeline and plan update during navigation.
6. Refresh dashboard; event stream resumes without loss or duplicate actions.
7. Receive a toast; capture timing and screenshot; simulate console/network failure.
8. Attempt forbidden checkout; policy gateway blocks and logs it.
9. Open visual comparison; inspect approved baseline versus a changed screenshot.
10. Inspect a proposed login test; have the coding worker draft and execute it.
11. Generate report with PASS/FAIL/INCONCLUSIVE policy and exact evidence links.
12. Pause, resume, and cancel another run; inspect full lifecycle and cost.

### Release gate for MVP itself

MVP is **not shippable** until:
- No critical known security failures in trust-boundary tests.
- All core features meet functional acceptance criteria.
- Dashboard A11Y passes automated checks and manual keyboard + screen-reader audit.
- Gate logic cannot return PASS when required tests are missing/blocked.
- Worker crash recovery and event replay are verified.
- Documentation clearly discloses external OpenAI API data flow.

## 8. Suggested repository layout

```text
agentic-qa-control-center/
  README.md
  .gitignore
  .env.example
  compose.yaml
  docs/
    intent.md
    plan.md
    spec.md
    architecture-review.md
    adr/
  apps/
    dashboard/               # React + TypeScript + Vite
    api/                     # FastAPI, domain API, auth, events
  agents/
    orchestrator/            # LangGraph, policies, memory, reflection
    browser-worker/          # TypeScript Playwright tool runner
    test-author/             # Isolated coding agent, generated test drafts
  packages/
    contracts/               # Event/graph/API schemas
    ui/                      # Accessible design-system components
    playwright-tests/        # Reviewed deterministic regression tests
  fixtures/
    sample-web-app/          # Safe local playground
  infrastructure/
    docker/
    migrations/
    scripts/
  tests/
    security/
    accessibility/
    integration/
  reports/
    examples/
```

Note: create these directories only as needed; avoid premature boilerplate and service sprawl.

## 9. Environments and setup

- **Local operator host:** Docker Engine + Compose, outbound HTTPS to OpenAI, limited LAN dashboard access.
- **Internal services:** FastAPI, isolated browser worker, Postgres, Redis, MinIO, Langfuse.
- **Target under test:** an explicitly authorized local/staging site; no default internet-wide exploration.
- **App auth:** local administrator login; short-lived sessions and CSRF protections.
- **Secret configuration:** .env.example holds placeholders only, mounted secrets hold actual values.
- **Rebuilds:** pin application/Playwright browser versions and record build identifiers with reports.
- **Artifacts:** object-store paths are private; UI serves authorized, time-bounded links.

## 10. Key risks, mitigations and review checkpoints

| Risk | Mitigation | Review |
|---|---|---|
| LLM mistakes treated as facts | Hypothesis/confirmed distinction and oracle metadata | Every finding |
| Prompt injection and data leakage | Isolated policy and secret broker | Security test in M1/M3 |
| Unsafe writes or duplicate bookings | Scoped actions and uncertain-write reconciliation | Before enabling writes |
| Flaky screenshot diffs | Stable environment + reviewed baseline | Before release gate |
| Accessible graph impossible to navigate | Full parity table view | M2 A11Y audit |
| Browser worker crashes | Durable journal, leases, safe resume | M1 |
| API costs spiral | Hard budgets, retry ceilings, dedup | Every run |
| Sensitive data in screenshots/traces | Redaction and retention | M1 onward |
| Report falsely passes | Explicit required checks and INCONCLUSIVE | M4 |
| Overengineering | Incremental milestones and one worker initially | Milestone review |

## 11. Questions that can be answered after scaffold (not blockers)

- First authorized target application's URL and authentication flow.
- Which specific OpenAI model and budget limits to use.
- Whether the first useful journey is login, event creation, checkout sandbox or something else.
- Initial visual baselines and target viewports.
- LAN bind address, domain access rules and preferred DNS.
- Whether external observability installations are reused or all services run in one Compose file.

## 12. Immediate next actions

1. Verify GitHub repository access and publish intent/plan/spec/review without secrets.
2. Confirm the first safe fixture and draw a trust-boundary diagram.
3. Scaffold Compose, frontend, API, browser worker and sample site.
4. Wire typed event contracts and a durable browser-action journal.
5. Build the first live dashboard panel, then integrate restricted browser actions.
6. Run fault, permission and accessibility acceptance tests before autonomous exploration.

## 13. Primary references

- Playwright: https://playwright.dev/docs/intro
- LangGraph persistence: https://docs.langchain.com/oss/python/langgraph/persistence
- FastAPI: https://fastapi.tiangolo.com/
- W3C WCAG 2.2: https://www.w3.org/TR/WCAG22/
- React Flow: https://reactflow.dev/
- OpenAI API data controls: https://developers.openai.com/api/docs/guides/your-data
