# PromptLab

Oct 21, 2026 · @Rushabh Patel

PromptLab lets teams test, refine and optimize the prompts inside ADK and LangGraph workflows that run as backend services, on real data and with statistically sound results. It starts with standalone prompt and model-upgrade testing, then grows into testing prompts inside the running workflows, without changing any service’s API contract.

## What is in this doc

| File | What it holds | Open it when you want to |
| --- | --- | --- |
| [Architecture & design](01-Architecture-and-design.md) | The full design: research, principles, architecture, interception planes, memoized execution, API contracts, workflow inputs, evaluation, statistics, security, roadmap, decisions | Understand how and why PromptLab works |
| [Diagrams](02-Diagrams.html) | Architecture with supported features, the user journey, and the API flow and dataflow of one real service | See the system at a glance |
| [Requirements](03-Requirements.md) | Testable requirements for every phase, API-contract compatibility, non-functional and UI/UX requirements | Plan, build or accept a phase |
| [Integration cookbook](04-Integration-cookbook.md) | Every supported service shape, a catalog of ADK and LangGraph scenarios, and worked end-to-end examples | Connect a specific service |
| [Workflow developer guide](05-Workflow-developer-guide.md) | How to build ADK and LangGraph services that integrate cleanly, with a readiness checklist | Write or refactor a workflow |

## Where to start

- **Architects and leads:** [Architecture & design](01-Architecture-and-design.md), then [Diagrams](02-Diagrams.html).
- **Planning a phase:** [Requirements](03-Requirements.md).
- **Onboarding a service:** [Integration cookbook](04-Integration-cookbook.md), then [Workflow developer guide](05-Workflow-developer-guide.md).

## Key decisions

- ADK first; LangGraph support arrives in Phase 2.
- Phase 0 is standalone prompt testing and model upgrades, built on your own PostgreSQL.
- Workflows write to production PostgreSQL, so every test run uses a read-only test deployment enforced in the database.
- Leads approve promotions of prompts and models.
- Services keep their own API contracts; PromptLab adapts with a declarative adapter.

## Roadmap at a glance

1. **Phase 0, standalone Prompt Lab:** registry, datasets, prompt and model comparisons, evaluators, paired statistics, lead approval.
2. **Phase 1, connect ADK services:** tracing, captured real inputs, endpoint adapters including SSE, production safety.
3. **Phase 2, in-workflow testing:** memo cache with FORK and REPLAY, write blocking, CI gate, LangGraph.
4. **Phase 3, optimize and scale:** GEPA optimization, production traces as datasets, scenarios, canary rollout.
