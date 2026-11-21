# Agentic QA Control Center — Rubber-Duck Architecture Review

> **Review type:** Self-contained critical architecture review, **not an independently spawned agent**. No separate agent-spawning tool was available in this chat.
> **Date:** 2026-11-21
> **Reviewed artifacts:** [intent.md](./intent.md), [plan.md](./plan.md), [spec.md](./spec.md).
> **Disposition:** Ready to scaffold a guarded prototype; NOT approved for arbitrary production or financial transactions.

## Review questions and findings

| Priority | Gap / risk | Decision or required design control | Validation |
|---|---|---|---|
| Critical | LLM instruction is not a security control | Enforce every browser action in an independent trusted policy gateway | Prompt-injected fixture tries prohibited external action and fails |
| Critical | Credential/token exposure in browser telemetry or OpenAI calls | Credential broker, minimal DOM extraction, request/response redaction, allowlist egress | Canary secret absent from all stored/exported artifacts and model payloads |
| Critical | Real-world purchases/messages/bookings may have side effects | Sandbox allowlist + capability scopes; real-money/production actions denied in MVP; transaction-specific approvals later | Unauthorized checkout fails before submit |
| Critical | Report might falsely PASS when agent never ran required checks | Gate computed from explicit required checks; incomplete = INCONCLUSIVE | Worker kill, blocked branch, no-oracle test all not PASS |
| High | A resumed browser write can duplicate an order/event | Idempotency keys, receipt reconciliation and write-uncertain status; no blind retry | Crash immediately after submit; no double-write |
| High | Graph based on URL misses business state, while raw hashes explode cardinality | Versioned semantic state fingerprints, dedup/merge with reversible audit | One route with two modal/role states remains distinguishable |
| High | AI 'observed' business rules are guesses | Every finding records oracle: explicit requirement, test rule or reviewed assumption | Deliberately ambiguous site is marked hypothesis |
| High | Visual comparisons may be flaky | Pin viewport, browser, fonts, data, locale/timezone, ready state; review masks/baselines | Stable page repeats with low diff; controlled change detected |
| High | Baseline approval could conceal regression | No automatic baseline promotion; version/reviewer audit | Automated attempt to overwrite baseline denied |
| High | Dashboard map not screen-reader compatible | Table representation with equivalent actions; live region throttled | Full keyboard/screen-reader journey works |
| High | UI disconnect hides stale agent / loses timeline events | Durable event log, reconnect cursor, stale indicator | Reload and reconnect reproduces complete history |
| High | Local Docker does not imply local data processing | OpenAI external API disclosure, pre-egress redaction, sensitive-data policy | Outbound payload audit with safe fixture |
| High | Browser worker redirects or fetches internal resources | Restrict origins, DNS/IP ranges and redirects at egress boundary | SSRF/private-IP and rebinding fixture denied |
| Medium | False notion of complete app coverage | Explicit 'known explored / known reachable' denominator; unknown remains unknown | Scope and denominator shown in every view/report |
| Medium | Cost overruns during cyclic exploration | Hard token/step/time/cost budgets and duplicate-frontier detection | Cyclic page terminates within budget |
| Medium | 'Dreaming' can hallucinate history | Reflection cites run events; suggestions only, no policy/test edits | No-evidence reflection cannot promote changes |
| Medium | Coding agent may introduce insecure or flaky tests | Separate worker, protected secrets, lint/type/static tests, human promotion | Generated test cannot access secret broker or push to main |
| Medium | Sensitive evidence persists beyond useful life | Retention windows, encryption, access checks, deletion and backup strategy | Cleanup eliminates expired artifacts and dangling refs |
| Medium | Concurrency may run multiple browser workers on one account | Single active session per identity in MVP, worker leases/fencing | Concurrent start is queued/denied as configured |

## Adversarial / failure-mode questions

1. **What if the webpage instructs the agent to send credentials to a different host?** Page content stays untrusted; tool gateway rejects arbitrary network egress, and raw credentials are unavailable to the model.
2. **What if a login requires an OTP/MFA?** Identity broker cannot assume automation. Put the run into operator-assisted authentication state and resume with a fresh approved session.
3. **What if a click succeeds but the browser crashes before recording its outcome?** Mark write outcome uncertain; reconcile with observable application data or operator review before retrying.
4. **What if a visual difference is an intentional design update?** Report change with evidence; human reviews baseline promotion; do not silently update.
5. **What if an AI asserts a bug without known expected behavior?** Keep it SUSPECTED / NEEDS_ORACLE, never confirmed blocking failure.
6. **What if a required test cannot be performed due to permission or budget?** Report INCONCLUSIVE, with exact reason and policy/step references.
7. **What if a user opens the dashboard from a keyboard only?** Reach all controls, status, queue, approvals and state transitions via the accessible table, not solely graph drag-and-drop.
8. **What if API or SSE stream fails?** Show stale status, store events durably and replay from last committed cursor on reconnect.
9. **What if model use becomes costly?** Hard ceilings halt plan creation; keep deterministic regression runner separate and usable without model calls.
10. **What if OpenAI sees confidential application content?** Egress classification and redaction happen before the request, with an operator-visible policy and synthetic canary validation.
11. **What if a test writes to a production endpoint behind a staging UI?** Network policy and app-side test credentials must independently restrict side effects; environment naming is insufficient proof.
12. **What if the generated test passes only after retries?** Mark flaky, keep test unpromoted until isolated and stable; show retry evidence.

## Decisions tightened by this review

- The release gate evaluates **declared checks**, not exploratory enthusiasm, counts of pages visited or AI confidence.
- All relevant artifacts and decisions carry immutable run/policy/model/baseline version information.
- Source-of-truth event log and state graph are shared between explorer, dashboard and report generator.
- Explicit CANCEL_REQUESTED versus confirmed CANCELLED prevents false claims about running workers.
- Dedicated browser execution and test-authoring workers prevent privilege creep.
- Initial scope uses a safe local fixture rather than trusting any unidentified public target.
- Test-data actions may be allowed at a scoped sandbox, whereas real payments remain denied even when broad testing is enabled.

## Unresolved decisions that do not block the repository bootstrap

- First test site's URL and authorization, plus desired auth/MFA handling.
- First OpenAI model choice and per-run spending limits.
- Which known business flows or required assertions define a PASS rather than INCONCLUSIVE.
- Initial baseline checkpoints and viewport matrix.
- Local server host/LAN bind and external target egress ranges.

## Reviewer recommendation

**Proceed with M0/M1 using an authorized fixture site, one identity, bounded read-only exploration and a visible event stream.** Expand scoped test-data writes only after gateway/approval/redaction tests pass. Do not promise production-safe or release-gating functionality merely because the dashboard renders.

This review should be repeated by an independently configured second model or human reviewer before the first deployment against a real application. The current review was performed within the same assistant session as authoring, so it is **not an independent assurance assessment**.
