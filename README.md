# Agentic QA Control Center

**Status: planning baseline; no application implementation has been deployed.**

A self-hosted Docker-based, human-supervised browser QA exploration and Playwright regression platform. The OpenAI API is the selected external inference provider; browser execution, UI, data, queues, reports and traces remain on the operator-managed host.

## Read in order

1. [Product intent and goals](docs/intent.md)
2. [Implementation plan and milestones](docs/plan.md)
3. [Detailed functional, UI, accessibility and technical specification](docs/spec.md)
4. [Critical architecture / rubber-duck review](docs/architecture-review.md)

## Initial build target

M0/M1: a local Docker Compose application with a reachable authenticated dashboard, event-driven run status and one restricted Playwright browser worker, running against an authorized local fixture site. Later phases add discovery, the app-state graph, visual regression, notifications, report gates, test authoring and reflection.

## Guardrails

Do not check in secrets, test credentials, session storage, logs or screenshots. Never give the explorer repository write access or permission to transact with real money. See `docs/spec.md` for environment policy, OpenAI data egress and WCAG 2.2 AA requirements.
