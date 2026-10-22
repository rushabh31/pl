# PromptLab: Backend Architecture & Design Proposal

*A production, multi-tenant platform to test, refine and optimize prompts inside real AI workflows · 20 October 2026 · Sources are listed in section 19.*

## 1. Executive summary

**Recommendation: build PromptLab as a build system and experimentation platform for AI programs, not as a prompt playground.** Every workflow, whatever framework it uses, is a program with text parameters. PromptLab intercepts it at the two boundaries every framework shares, the model call and the data read. It memoizes every call by content, and it turns prompt testing into incremental, statistically paired recomputation on real data.

**The problem.** A workflow’s accuracy comes from the *whole* flow: what was retrieved, what a tool returned, what an upstream prompt produced. A playground that sends one prompt and a sample input to a model tests a fiction. Langfuse’s documentation draws exactly this line: its UI experiments run a prompt and model directly, while SDK experiments run your code and therefore exercise your real retrieval and tool logic.

**Why the boundaries matter.** There are dozens of agent frameworks, and new ones every quarter. All of them end in a request to a handful of model SDKs or a model gateway, and all of them read data through a handful of client libraries (HTTP, SQL, vector stores, MCP). A request to the model already carries the system prompt, messages, tool descriptions and parameters, so steering at that boundary works the same for ADK, LangGraph, CrewAI and hand-rolled code. Framework-specific code shrinks to optional extras: names, templates and native checkpoints.

| # | Idea | What it gives you |
| --- | --- | --- |
| 1 | **Steer at the model boundary** | One interceptor per model SDK, or a gateway, edits system prompts, tool descriptions and parameters in any framework |
| 2 | **Capture at the data boundary** | OpenTelemetry auto-instrumentation for HTTP, database, vector-store and MCP clients, extended to record payloads, freezes what the flow read. No decorators needed |
| 3 | **Memoize every call** | A content-addressed cache of model and data calls. Fork-from-step then *emerges* in any framework, even inside agent loops: unchanged calls hit the cache, and the first changed call and everything after it run live |
| 4 | **Common random numbers** | Variants share identical upstream samples, so a paired difference isolates the prompt change and fewer items are needed |
| 5 | **Prompts are runtime configuration** | Resolution follows OpenFeature semantics (evaluation context, variants, targeting) with an OFREP-compatible endpoint, so offline experiments and production canaries use one mechanism |
| 6 | **Production traces are datasets** | Any captured production run can be re-executed with new parameters on exactly the context production saw |
| 7 | **Test cases beyond input and output** | Scenarios with simulated users, metamorphic groups and trajectories, with pass^k reliability |
| 8 | **One narrow waist, negotiated by capability** | One canonical workflow model and run protocol, with features enabled by what each runner declares |
| 9 | **Statistical honesty** | Paired designs, intervals, a held-out confirmation split, calibrated judges, gated promotion |

**Build vs. adopt.** Reuse OpenTelemetry and OpenInference for traces, a durable-execution engine for orchestration, ClickHouse for spans and scores, and the open-source `gepa` library for optimization. Build the differentiators: interception planes, the memo cache, the narrow waist and statistically gated promotion. Section 17 gives a phased roadmap.

**What changed from the previous draft, and why**

| Previous draft | This version | Why |
| --- | --- | --- |
| Per-framework shims to steer prompts | Model-boundary interceptor; shims only add names and templates | A handful of model SDKs instead of dozens of frameworks |
| Decorators to mark retrieval, database and tool calls | Data-boundary capture through auto-instrumentation; decorators optional | Zero code change keeps the flow intact |
| Tapes plus fork-from-step that needed checkpoints and excluded loops | Content-addressed memoization from which fork emerges | Works in loops and in any framework |
| A bespoke overlay resolution API | OpenFeature-style resolution with an OFREP endpoint | A standard, and the same mechanism as production rollout |
| Curated datasets only | Adds production counterfactuals, scenarios and metamorphic tests | Removes the dataset bottleneck and tests agents properly |
| An inventory of microservices | A modular-monolith control plane, separately scaled hot paths, cells | Faster to build and cheaper to run |
| OpenAPI as the source of truth | Protobuf as the source; JSON Schema, OpenAPI 3.2 and gRPC generated | A2A v1.0 defines its model the same way |

### Your decisions and what they change (21 October 2026)

| Topic | Your answer | Design consequence |
| --- | --- | --- |
| Frameworks | Mostly ADK; add LangGraph | ADK first: the Google GenAI interceptor and the ADK plugin come before anything else. LangGraph follows in Phase 2. Other frameworks leave the near-term roadmap |
| Data and traces | Data lives in PostgreSQL; no tracing is in place today | PromptLab provides tracing from scratch through its launcher and the OpenInference ADK instrumentor. Postgres reads are captured through SQL instrumentation. There is no existing trace store to migrate |
| Scale | Not yet known (explained below) | Start on Postgres only, and add ClickHouse when the thresholds in Section 13 are crossed |
| Starting point | Test prompts outside the workflow first, including model upgrades | Phase 0 becomes a standalone Prompt Lab (6.8). In-workflow testing follows in Phase 2, on the same registry, datasets, evaluators and statistics |
| Production writes | Workflows write to production PostgreSQL | No test run may reach production writes. Enforced in Postgres itself (read-only role on a replica or staging copy), with SQL write blocking as a second layer (6.5) |
| Approvals | Leads approve promotions; no compliance regime stated | A `lead` role approves moves to protected aliases (staging, prod), with an audit record. Revisit if a regime applies |
| Service style | Workflows run as backend APIs, each with its own contract and code style | A declarative endpoint adapter maps dataset items to each service’s own request and response (7.7). Services change neither contract nor code |

**About the scale question.** It asks how much load PromptLab itself will carry, which decides when an analytics database is worth adding. Three numbers matter:

- **Users and workflows:** how many teams and services will run tests.
- **Runs at once:** an experiment of 200 items × 3 variants × 3 repetitions is 1,800 runs, and several experiments may overlap.
- **Steps per run:** every model call, database query and tool call is recorded as one span; an ADK agent typically produces somewhere between a handful and a few dozen.

Postgres alone is the right start. As a planning estimate to confirm with a load test, move spans to ClickHouse when trace queries take more than a few seconds or span volume passes roughly 50 million a month.

**Companion tabs.** Requirements lists testable requirements for every phase, including API-contract compatibility and UI/UX. Workflow developer guide tells workflow teams how to build ADK and LangGraph services that integrate cleanly.

## 2. What the research says, and what we do about it

| Finding (as of Oct 2026) | Design consequence |
| --- | --- |
| OpenTelemetry’s GenAI conventions define spans for model calls, agents (create\_agent, invoke\_agent, invoke\_workflow, plan, execute\_tool) and MCP, plus a gen\_ai.evaluation.result event. They moved to a dedicated repository in June 2026. Most sources still label them Development status; one source claims they are stable. | Use OTLP with GenAI attributes as the trace format, pin the convention version per span batch, isolate the mapping in one module, and keep our own attributes under `promptlab.*`. Verify stability before pinning. |
| OTel spans carry timing and metadata by default; prompt and completion content is opt-in. | Experiments need content, so the SDK enables content capture for experiment runs only, with redaction inside the customer’s plane. |
| GEPA (ICLR 2026 oral) evolves the text components of an arbitrary system by reflecting on full execution traces and textual feedback, keeping a Pareto frontier of candidates. Its adapter interface is evaluate(batch, candidate as a dict of component texts, capture\_traces) plus make\_reflective\_dataset. It ships inside DSPy and MLflow. | Our overlay is a GEPA candidate, the Runner Protocol is evaluate(), and traces feed reflection. GEPA is the default optimizer. |
| MLflow’s optimize\_prompts (3.5+) wraps GEPA and supports multi-prompt optimization, but the app must load its prompts through the MLflow registry. | That is the integration gap. Decorators make hard-coded prompts addressable so existing code works with minimal edits. |
| Braintrust remote evals run on your infrastructure while its playground triggers runs, sends parameters and shows results. Langfuse offers webhook-triggered SDK experiments. | Confirms the callback model. We add pull-based runners (no inbound ports), several prompts per overlay, and strict modes. |
| Microsoft’s Agent Lightning separates agent, tracer, store and algorithm. It collects prompts, tool calls and rewards as spans, feeds improved prompt templates back, and uses a LiteLLM proxy to collect LLM traffic. | Validates the gateway tier and the idea of pushing updated “resources” into an unmodified agent. |
| MCP 2026-07-28 made the core stateless and moved Tasks into an extension: a call can return a task handle that clients poll (tasks/get), update or cancel. tasks/list was removed. | Experiments become handles with get, update and cancel. We also expose PromptLab as an MCP server. |
| A2A reached v1.0 in 2026 under the Linux Foundation’s Agentic AI Foundation: Agent Cards, tasks with states such as input-required and rejected, signed cards. | Optional binding so an agent-style workflow can be driven as an A2A agent; our run states align with its lifecycle. |
| OpenAPI 3.2 (Sept 2025) adds first-class streaming media types (SSE, JSONL) through itemSchema and the QUERY method, still on JSON Schema 2020-12. | Contract-first in 3.2, run events streamed as SSE, QUERY for complex searches. |
| LLM judges show position, length and self-preference bias, and the size of each bias varies by judge. Practice: a human gold set, calibration (for example Cohen’s kappa), order swapping, cross-family ensembles. An ICML 2026 paper shows how to correct judge-induced bias in reported scores. | Judges are versioned, calibrated artifacts. Reports use bias-corrected intervals when calibration data exists. |
| For prompt comparisons, practitioners converge on paired designs, bootstrap intervals on per-item differences, McNemar for pass/fail, an untouched confirmation set and multiple-comparison correction. Anytime-valid intervals allow safe peeking. | A stats engine is a core service, not a chart added later. |
| Agent-eval practice: record tool and retrieval outputs once and replay them. Replay should never silently fall through to live calls. | Tapes, and a strict REPLAY mode where a miss fails the trial. |
| Durable-execution engines (Temporal, Restate, DBOS on Postgres) are now standard under long-running agent workloads. | Experiments run as durable workflows. |
| Langfuse v4 runs on Postgres, ClickHouse, Redis and S3-compatible storage. ClickHouse acquired Langfuse in January 2026. | We use the same storage shape. |

*Source quality: specifications and official docs wherever possible. Several 2026 details come from vendor and practitioner blogs, so re-verify before pinning versions.*

**Second pass: what the frameworks already expose.**

- Google ADK lets one Plugin, registered on the Runner, hook every agent, tool and model call, and ships an `adk api_server` with sessions, `/run` and `/run_sse`.
- LangChain v1 agents accept middleware (`dynamic_prompt`, `wrap_model_call`) that receives the model request and can rewrite it.
- LangGraph checkpoints support replay and fork (change state at a past checkpoint, then resume). Its Agent Server exposes threads, runs, per-run `configurable` and `context`, graph introspection, and A2A and MCP endpoints.
- OpenInference ships instrumentors for LangChain and LangGraph, Google ADK, CrewAI, LlamaIndex, OpenAI Agents, DSPy and more, so observation needs no new tracers.
- Consequence: integrate through each framework’s own hooks and servers instead of asking teams to rewrite code around our SDK. Section 5 builds on this.

**Third pass: standards and techniques this revision rests on.**

- OpenInference already instruments the model SDKs themselves (OpenAI, Anthropic, Google GenAI, Bedrock, LiteLLM, Mistral, Groq, VertexAI), which shows that a stable interception point exists at the model-client boundary in each of them.
- OpenFeature’s Remote Evaluation Protocol (OFREP, spec version 0.3.0) standardizes flag evaluation with a per-request evaluation context, bulk evaluation with ETags, and community providers in many languages.
- Record-and-replay eval tools key cached model, API and judge calls by the canonical request, and their strict replay mode turns a miss into a failure instead of a live call.
- τ-bench and its successors evaluate agents against an LLM-simulated user with a hidden goal and check the final database state, and they report pass^k because one success can be luck. A 2026 study found that changing only the user simulator changes measured agent performance.
- Metamorphic testing checks expected relations between outputs of semantically equivalent inputs, which needs no labels. A 2026 study found that model scale did not predict robustness under such transformations.
- A2A v1.0 defines its data model in Protocol Buffers and publishes JSON Schema generated from it.

## 3. Design principles

1. **Intercept at universal boundaries before framework-specific ones.** Model calls and data reads are where every framework meets reality.
2. **The workflow stays where its data is.** Database access, credentials and network position remain on the customer side.
3. **Every call is a function of its content-addressed request.** Whatever is non-deterministic is masked or keyed explicitly, which makes caching, replay and fork correct.
4. **Prompts are configuration, resolved at call time with an evaluation context.** Changing a prompt is never a deploy.
5. **Everything is a trace, and every trace is replayable.**
6. **One contract for any granularity.** A single call, a step, a subset or the whole workflow all use the same run request.
7. **Determinism is visible.** Every result states its mode, cache hits and misses, and replay fidelity.
8. **Production never depends on PromptLab.** Interceptors and resolvers fail open to cached or built-in prompts.
9. **Failures count.** A crashed trial is a scored failure.
10. **Security starts in the data plane.** Sensitive payloads can stay with the customer.
11. **Simple to operate.** One deployable control plane first; split services only when measured load demands it.

## 4. System architecture

The system splits into a control plane that decides what to run and judges results, and a data plane inside the customer’s runtime where the workflow runs against real data. Three interception planes inside the data plane are what make any framework testable.

```
 CONTROL PLANE: one deployable, modular monolith           HOT PATHS: scaled separately
 ┌────────────────────────────────────────────────────┐   ┌───────────────────────────┐
 │ Registry & resolution (OFREP)  ·  Datasets & scenarios│   │ Runner gateway (push/pull)│
 │ Experiments (durable)  ·  Evaluation  ·  Optimization │   │ OTLP ingest + normalizer  │
 │ Statistics & promotion  ·  Tenancy & audit            │   │ Memo & capture store      │
 └──────────┬──────────────────────▲─────────────────────┘   └─────────────▲─────────────┘
            │ parameters, runs      │ results, scores                     │ spans, captures
            ▼                       │                                     │
 DATA PLANE: customer runtime or VPC
 ┌────────────────────────────────────────────────────────────────────────────────────┐
 │  Your workflow: ADK · LangGraph · LangChain · CrewAI · custom code · remote agents   │
 │   Semantic plane  framework instrumentors (+ optional shim or pl.slot) → names      │
 │   Model plane     model-SDK interceptor or gateway → steer · memoize · trace        │
 │   Data plane I/O  HTTP · SQL · vector · MCP auto-instrumentation → capture · guard  │
 │        └─► real databases · RAG stores · tools · model providers                    │
 └────────────────────────────────────────────────────────────────────────────────────┘
 Stores: Postgres · ClickHouse · Redis/Valkey · object storage (captures, payloads)
```

**Why a modular monolith.** The control-plane modules share one transactional model (registry, experiments, promotions) and modest traffic. One deployable with strict module boundaries ships faster and fails in fewer ways than a dozen services. The three hot paths scale with trace and trial volume, so they run separately from day one. Larger tenants move into their own **cell**, a full copy of the stack, which also serves data residency.

| Module | Responsibility | Default technology |
| --- | --- | --- |
| Registry & resolution | Immutable content-addressed prompt and tunable versions, aliases, lineage; resolution with an evaluation context, also exposed through OFREP | Postgres, Redis cache |
| Datasets & scenarios | Versioned items, captured runs, scenarios, metamorphic groups, fixed splits | Postgres, object storage |
| Experiments | Durable fan-out of trials, retries, budgets, cancel and resume | Temporal (or DBOS on Postgres, Restate) |
| Evaluation | Deterministic, judge, human and simulator-based evaluators; judge calibration | Workers via provider gateway |
| Optimization | GEPA and other optimizers behind one adapter | Python workers (`gepa`) |
| Statistics & promotion | Paired intervals, tests, power, sequential monitoring, gates, alias moves | Python |
| Runner gateway (hot path) | Push dispatch and pull leases, invocation profiles | Go service, Redis |
| Ingest (hot path) | OTLP receiver, OpenInference and GenAI normalization, redaction | OpenTelemetry Collector + processor |
| Memo & capture store (hot path) | Content-addressed cache lookups and writes for model and data calls | Go service, object storage, Redis index |
| Provider gateway | Keys (BYOK), rate limits, caching for judge and reflection calls; also the steer point for non-SDK stacks | LiteLLM-class proxy or in-house |

**Lifecycle of one experiment**

1. A user, CI job or agent posts an experiment: workflow, variants (overlays), dataset version, evaluators, mode, budget.
2. The orchestrator expands it into trials (variant × item × repetition), interleaved so provider drift hits every variant alike.
3. Each trial reaches a runner through its invocation profile. The run carries the overlay reference, mode, cache scope, sample index, trace context and an idempotency key.
4. Inside the workflow, the model plane resolves and applies the overlay to each matching model request and looks it up in the memo cache. The I/O plane serves or records data reads. Both emit spans.
5. Results return with cache hits, misses and the hash of the overlay actually applied.
6. Evaluators score outputs and spans. The statistics module compares variants item by item.
7. The result is a reproducible record. A promotion gate may move an alias, and an optimizer may reuse the same machinery as its evaluate().

## 5. Integration: three interception planes and a narrow waist

Any workflow becomes testable by intercepting it at three planes. Two of them, the model plane and the I/O plane, are framework-independent, so a new framework works on day one. The third adds framework-specific names and templates, and it mostly comes from existing open-source instrumentors.

### 5.1 The three planes

| Plane | What it sees | What it enables | Mechanism | Framework-specific code |
| --- | --- | --- | --- | --- |
| **Model plane** | The canonical model request: system prompt, messages, tool definitions, parameters, and the response | Steer prompts, tool descriptions and parameters; memoize; cost and latency | An in-process interceptor on each model SDK (OpenAI, Anthropic, Google GenAI, Bedrock, LiteLLM and others), loaded without code changes the way OpenTelemetry’s zero-code agents are; or the provider gateway by base URL | None |
| **I/O plane** | Requests and responses of HTTP, SQL, vector-store and MCP clients | Capture and replay the data the flow read; block writes during tests | OpenTelemetry auto-instrumentation libraries, extended with payload capture in experiment runs | None |
| **Semantic plane** | Agent, node, tool and step names; templates and variables; graph structure | Call-site identity, per-step scoring, template-level edits, native checkpoints | OpenInference instrumentors; an optional thin shim (ADK plugin, LangChain middleware); `pl.slot` for hand-rolled code | Small and optional |

**Call-site identity without framework code.** When the model interceptor fires, it reads the active span ancestry the framework instrumentor has already created, for example an `invoke_agent billing` span with a chat span inside it. From that it derives a stable `call_site_id` such as `agent:billing/llm`. With no instrumentor present, it falls back to a fingerprint of the request’s static parts.

**Invocation** is the fourth concern and the only one outside the workflow process: an invocation profile drives a surface the workflow already exposes (LangGraph Agent Server, `adk api_server`, A2A, MCP, plain HTTP, a CLI, or an in-process call). See 7.5.

### 5.2 Overrides are structural patches on the canonical request

An overlay entry targets paths inside the canonical request for one call site, written as JSON Patch operations (RFC 6902). The same override then means the same thing in every framework.

```json
{
  "call_site": "agent:billing/llm",
  "patch": [
    {"op": "replace", "path": "/system", "value": {"template_ref": "prompt://billing.system@v14"}},
    {"op": "replace", "path": "/tools/refund_lookup/description", "value": "Look up refund status by order id. Never guess an id."},
    {"op": "replace", "path": "/params/temperature", "value": 0}
  ]
}
```

**Templates with variables.** Static instructions, tool descriptions and parameters can be replaced outright. When a prompt embeds per-run values (a user name, retrieved text, a date), replacing the whole string would erase them. Two mechanisms handle that: the semantic plane supplies the template and its variables where a shim or `pl.slot` exists, and template induction (5.7) infers the holes elsewhere. Steer verification (7.3) still confirms that each override actually landed.

### 5.3 Framework matrix

Verified in documentation: ADK plugins and callbacks, LangChain v1 middleware, LangGraph checkpoints and Agent Server API, and OpenInference instrumentors for the model SDKs and frameworks listed. Model-plane interception on each SDK version, and every steer hook not listed as verified, are assumptions to confirm in a one-week spike. The Integration cookbook walks through ADK, LangChain and LangGraph end to end.

| Framework | Steer by default | What the optional shim adds | Replay and fork | Invoke | Caveat |
| --- | --- | --- | --- | --- | --- |
| **Google ADK** | Model plane on the Google GenAI client | A Runner plugin for instruction templates and session-state variables (`before_model_callback` sees the model request) | Memo cache | `adk api_server` | Plugins are not supported in the ADK web UI |
| **LangGraph and LangChain v1** | Model plane on the provider SDKs | `dynamic_prompt` or `wrap_model_call` middleware for templates | Memo cache; native checkpoints optional for long human-in-the-loop runs | Agent Server | Native replay re-executes nodes, and subgraphs need their own checkpointer for time travel |
| **CrewAI, LlamaIndex, OpenAI Agents SDK, DSPy, Haystack, Strands, Agno, smolagents** | Model plane | To validate per framework | Memo cache | In-process or HTTP | DSPy ships its own GEPA; PromptLab adds datasets, governance and promotion |
| **Services in other languages** | Gateway by base URL | None | Memo at the gateway for model calls; I/O capture where an OpenTelemetry agent exists | HTTP | Templates are best effort |
| **Remote agents over A2A or MCP** | Only what the remote honors, for example overlay metadata | None | None | A2A, MCP | Otherwise black-box benchmarking |

### 5.4 Capability negotiation

Every runner publishes a manifest of capabilities, and the platform enables features accordingly instead of demanding all-or-nothing integration.

| Declared capabilities | What the platform enables |
| --- | --- |
| `observe` + `invoke` | **Benchmark mode**: run datasets, score outputs and traces, compare versions of your code |
| + `steer.model` | Experiments and optimization over static prompts, tool descriptions and parameters |
| + `steer.template` | Edits to templated prompts with variables preserved |
| + `memo` | Cheap re-runs, fork-from-step, common random numbers |
| + `capture.io` | REPLAY and HYBRID runs on frozen data, CI regression, production traces as datasets |
| + `introspect.graph` | A pre-run map of call sites and per-node credit assignment |
| + `effects.control` | Safe LIVE runs against systems with writes |

New capabilities ship as **extensions** with reverse-DNS ids and versions (for example `io.promptlab/memo`), the mechanism MCP formalized in 2026, so the core does not change for each new idea.

### 5.5 Canonical Workflow Model

The narrow waist is a framework-neutral graph of what a workflow does: model calls, tool calls, retrievals, agents, routers and handoffs, subgraphs and human steps. Each node has a `call_site_id`, edges to its parents, and its tunables. It is built from traces and enriched by static introspection where available (the LangGraph graph endpoint, an ADK agent tree, an A2A agent card). Trace views, per-step scoring, optimizer component selection and statistics read only this model, so N frameworks and M platform features cost N + M integrations, not N × M. In multi-agent systems a handoff is an edge, each agent’s instruction and handoff description are tunables, and evaluators score each agent and the trajectory separately.

### 5.6 Tunables, not just prompts

A tunable is a typed parameter on a call site: system instruction, template with variables, tool name and description, few-shot examples, model and sampling parameters, retrieval parameters (k, thresholds, reranker), routing or handoff text, guardrail text. Each type has a contract (required variables, schema, maximum length). Text tunables are GEPA components; numeric and enum tunables go to grid, Bayesian or bandit search. One overlay can mix them, so a multi-agent system is tuned as one system.

### 5.7 Template induction

Comparing several captured renderings of the same call site reveals the static skeleton and the variable holes. An override can then edit the skeleton while the holes are refilled on each call. It is best effort, reports a confidence score, and refuses when it cannot locate the holes. The same technique suggests `pl.slot` placements for teams that want exact templates.

### 5.8 Low-friction onboarding

1. **Zero code:** `promptlab run -- python app.py` (or the equivalent Node launcher) loads the interceptors and auto-instrumentation, the same pattern as OpenTelemetry zero-code agents. For other stacks, point the model client’s base URL at the gateway.
2. **Configuration:** `promptlab init` detects the framework and writes a `promptlab.yaml` with the invocation profile, effect policy and prompt source.
3. **Verification:** `promptlab doctor` probes each capability: a span arrives, a canary phrase swapped into a prompt appears in the model request, a repeated call hits the cache, a captured read replays. A failed probe lowers the declared capabilities instead of failing the integration.

```yaml
workflow: support-triage
framework: langgraph                 # detected; adk, langchain, custom, ...
invoke: {profile: langgraph-agent-server, url: http://localhost:2024, assistant: agent}
prompts: {source: git, paths: [prompts/**/*.md]}   # registry, git or code
effects: {write: deny}
capture: {io: [http, sql, vector, mcp], volatile_masks: ["$.messages[*].content#timestamp"]}
```

The prompt source of truth is a per-workflow choice: a registry alias, **Git files** (the optimizer opens a pull request and CI runs the experiment), or the default in code.

### 5.9 Execution backends

One executor interface serves three backends: a **customer runner** (push or pull), a **managed sandbox** that runs a container image for teams without their own infrastructure, and a **local dev tunnel** (`promptlab dev`) so a laptop can act as a runner while iterating.

### 5.10 Why this beats the alternatives

| Concern | SDK-first | Per-framework shims (previous draft) | Interception planes (this design) |
| --- | --- | --- | --- |
| Work for an ADK or LangGraph team | Rewrite prompts as slots | One line per framework | None to start; a shim only for templates |
| A new framework appears | New adapters per feature | A new shim | Works at once through the model and I/O planes |
| Replay of RAG and database reads | Decorators on every boundary | Decorators or native checkpoints | Automatic I/O capture |
| Fork inside agent loops | Not supported | Not supported | Emerges from memoization |
| Maintenance | N × M | N + M | A handful of SDK interceptors + optional shims |

**Honest risks.** Interceptors must track model-SDK releases, so each ships with a conformance suite and a published support matrix. Streaming responses, provider-side features (built-in tools, cached contexts) and async clients each need explicit handling. Template induction is brittle by nature and always reports its confidence.

## 6. Memoized execution: replay and fork without checkpoints

Every model call and every captured data read is keyed by a hash of its canonical request. Re-running a workflow with an overlay reuses every call whose request is unchanged and runs live from the first call that differs. That single mechanism provides replay, fork-from-step, cheap re-runs and variance reduction in any framework.

### 6.1 How it works

1. **Canonicalize** each request into a provider-neutral form: sorted keys, normalized whitespace, and configured volatile fields (timestamps, request ids) masked.
2. **Key** it: hash of the canonical request, pinned model version, sample index, and for data reads the capture epoch.
3. **Look up** on every call. A hit returns the recorded response and emits a span marked as a hit. A miss executes according to the mode and records the result.
4. **Changes propagate by themselves.** Downstream requests embed upstream outputs, so a changed prompt changes its own request and, transitively, every request that depends on it. Unchanged upstream calls still hit. Fork-from-step is derived, not declared.

```
Baseline run (recorded)            Variant run: the summarize prompt changed
kb_search(q)       vector read     kb_search(q)       HIT   same request
plan               model call      plan               HIT   same request
summarize          model call      summarize          MISS  request changed → live
classify           model call      classify           MISS  input changed → live
update_ticket      HTTP write      update_ticket      BLOCKED  write effect, mocked
```

**Why this beats checkpoints.** It needs no framework support, so it works the same in ADK, LangGraph and plain Python. It works inside agent loops, because calls hit until the trajectory diverges and run live after that. The cache is shared across experiments, so a hundred candidate prompts pay for the upstream calls once. A **tape** is simply a named, frozen set of memo entries, for example the set committed for CI.

### 6.2 Modes are cache policies

| Mode | Model calls | Data reads | Typical use |
| --- | --- | --- | --- |
| **LIVE** | Always executed, recorded | Always executed, recorded | Ground truth; the numbers you ship on |
| **FORK** (default while iterating) | Hit if unchanged, otherwise live | Hit if unchanged, otherwise live read (or fail, by policy) | One prompt or several, inside the real flow |
| **REPLAY** | From cache; a miss fails the trial | From cache; a miss fails the trial | Deterministic CI regression |
| **HYBRID** | Per call-site policy | Per boundary policy | Some data volatile, some stable |

Every result reports its mode, hits, misses and **replay fidelity** (the share of calls served as recorded). A REPLAY number is never mistaken for a LIVE one. Working rule: iterate in FORK, gate in REPLAY, confirm in LIVE.

### 6.3 Common random numbers

The sample index is part of the key, so repetition *r* of variant A and repetition *r* of variant B receive identical upstream samples, including identical retrievals. Their paired difference then reflects the prompt change rather than upstream randomness, the classic variance-reduction technique of simulation studies, and fewer items are needed for the same confidence. Two cautions: results are conditional on the shared upstream samples, so enough repetitions and items are still required; and the cache never fabricates new samples, because a new sample index is always a miss.

### 6.4 Correctness guards

- **Volatile masks** keep timestamps and random ids from causing false misses, and `promptlab doctor` flags unstable fields it detects.
- **Model version is keyed; code version is recorded but not keyed.** An identical request to the same model is a draw from the same distribution, whoever built it.
- **Capture epoch** makes stale data visible. Each result shows how old its captured reads are, and scheduled LIVE runs detect drift.
- **Fidelity checks:** a sampled fraction of hits is re-executed live and compared, which catches caching bugs before they mislead anyone.
- **Isolation:** the cache is per tenant and write-once, and its entries live in the data plane when payloads are sensitive.

### 6.5 Effect safety

The I/O plane classifies calls as read or write by default (HTTP GET versus POST, PUT, PATCH and DELETE; SQL SELECT versus statements that modify data; MCP tools by annotation), and configuration or decorators override the default. Outside LIVE mode, writes are never executed: they are mocked from the recording or blocked. LIVE runs deny writes unless an experiment explicitly allows them against a sandbox target.

**Your case: workflows write to production PostgreSQL.** Protection is layered, strongest first:

1. **Never test against the production primary.** Test runs call a separate test deployment of each service, connected to a staging copy or a read replica.
2. **Enforce it in Postgres.** The test deployment connects with a role that has SELECT-only grants and `default_transaction_read_only = on`, so any write fails inside the database whatever the code does.
3. **Block and score writes in the I/O plane.** In FORK and REPLAY modes, write statements become recorded no-ops, so the workflow keeps running and the attempted write becomes a test signal (for example, did it try to refund the right order).
4. **Disposable databases for write tests.** A LIVE test that must exercise writes runs against a per-run database or schema restored from a snapshot, never a shared one.

Rolling back a transaction around the run is not relied on: services open their own connections and commit, and a rollback cannot undo effects outside the database.

### 6.6 Testing one prompt, some prompts, or all

| You want to test | How |
| --- | --- |
| One prompt in isolation | Re-send its captured requests with the patch applied; no workflow run at all |
| One prompt inside the real flow | FORK with a one-entry overlay |
| Several prompts together | A multi-entry overlay, a factorial grid, or a leave-one-out ablation |
| The whole workflow end to end on real data | LIVE with an overlay |
| Cheap, deterministic regression on every pull request | REPLAY against a committed tape |

### 6.7 Escape hatches

Use explicit markers only where interception cannot see: templates with variables in hand-rolled code, an in-memory index, or a custom transport.

```python
import promptlab as pl

SUMMARIZE = pl.slot('triage.summarize',
    default='Summarize in 3 bullets. Context: {context} Ticket: {ticket}',
    variables=['context', 'ticket'])

@pl.boundary('local_index', effect='read')   # captured and replayed like any I/O call
def search_local(q: str) -> list[Doc]: ...
```

At call time a slot resolves in this order: an explicit per-call override, the active overlay, the registry alias for the environment (cached with ETag and stale-while-revalidate), then the default in code. Run context travels in `contextvars` or `AsyncLocalStorage` in-process, and in W3C `traceparent` and `baggage` headers across services, carrying references rather than payloads.

### 6.8 Standalone prompt testing and model upgrades (Phase 0)

The first release tests a prompt as a function: template, variables, model and parameters in; output out, scored and compared statistically. No workflow runs and no service or production database is touched, so it delivers value on day one. Everything it builds (registry, datasets, evaluators, statistics, experiments) is reused unchanged when in-workflow testing arrives.

**Its weakness is input realism, and the roadmap fixes it.** A prompt tested on invented variables can pass and still fail on what the workflow really sends it. Inputs therefore come from three sources, in rising realism:

1. Hand-written items, or CSV and JSON uploads.
2. Values copied from application logs.
3. Inputs captured from real ADK calls once the interceptor is in place (Phase 1): the exact rendered variables, including what was read from Postgres. A standalone test on captured inputs is a faithful single-call test of the real workflow.

**The prompt unit.** A prompt version bundles the messages, typed variables, an optional output schema, and the model with its parameters, so a model change is a new prompt version that can be compared and promoted like any other. Templates use ADK’s `{variable}` syntax so existing ADK instructions import unchanged.

```yaml
# prompts/billing.system.yaml
id: billing.system
version: 14
messages:
  - role: system
    template: "You are a billing assistant for {customer_tier} customers. Always confirm the order id before discussing refunds."
  - role: user
    template: "{message}"
variables:
  customer_tier: {type: string, enum: [standard, premium]}
  message: {type: string}
output_schema: {type: object, required: [answer], properties: {answer: {type: string}, needs_human: {type: boolean}}}
model: {provider: google, name: gemini-2.5-flash, params: {temperature: 0.2}}
```

**API contract.** One endpoint runs a prompt once; experiments run it over a dataset. The playground UI calls the same API.

```json
POST /v1/prompt-runs
{
  "prompt": {"ref": "prompt://billing.system@14"},
  "variables": {"customer_tier": "premium", "message": "Where is my refund for order 8812?"},
  "model": {"name": "gemini-2.5-flash", "params": {"temperature": 0.2}},
  "seed": 7
}
→ 200 {"output": "…", "parsed": {"answer": "…", "needs_human": false},
       "schema_valid": true, "usage": {"input_tokens": 412, "output_tokens": 96, "cost_usd": 0.0004},
       "latency_ms": 820, "trace_id": "…"}
```

`model` overrides the version’s default, which is how an upgrade candidate is tried without editing the prompt.

**Model upgrade flow**

1. **Baseline:** the current prompt version on the current model, run once over the dataset and cached.
2. **Candidate:** the same prompt on the new model.
3. **Adapted candidate (optional):** a prompt version tuned for the new model, by hand or with GEPA, because new models often need prompt changes to show their best.
4. **Compare** quality, schema validity, refusal rate, cost and latency with paired statistics, using a judge from a different model family than the arms wherever possible.
5. **Promote** the winning prompt version, model included, after a lead approves.

```yaml
# experiments/billing-model-upgrade.yaml
experiment: billing-model-upgrade
target: {prompt: billing.system}           # a prompt, not a workflow
dataset: billing-inputs@v2
split: validation
repetitions: 3
variants:
  baseline:        {prompt_version: 14}
  new_model:       {prompt_version: 14, model: {name: "<candidate-model>"}}
  new_model_tuned: {prompt_version: 15, model: {name: "<candidate-model>"}}
evaluators: [answer_quality_judge@v1, schema_valid, refusal_rate]
stats:
  primary: answer_quality_judge
  guards: {cost_usd: "+20%", latency_p95_ms: "+20%"}
```

## 7. API contracts

Contracts are the product: one small, versioned core that any workflow can implement, plus profiles that reuse the invocation surfaces frameworks already ship.

### 7.1 The contract family

| Surface | Audience | Specification | Notes |
| --- | --- | --- | --- |
| **Control API** | UI, CLI, CI, scripts | OpenAPI 3.2, JSON Schema 2020-12 | Workflows, tunables, prompts, datasets, experiments, trials, evaluators, optimizations, promotions. Complex searches use the QUERY method |
| **Run Protocol** (core + profiles) | Runners and shims | OpenAPI 3.2 plus profiles for existing surfaces | 7.2 to 7.5 |
| **Evaluator protocol** | Custom scorers in any language | OpenAPI 3.2 | Section 10 |
| **Event streams** | Live experiment views, CI | SSE with `itemSchema` (OpenAPI 3.2) | Resumable with Last-Event-ID |
| **Webhooks** | Notifications, runner triggers | HMAC-signed | Replay-protected with timestamps |
| **MCP facade** | Coding agents and IDEs | MCP 2026-07-28 with the Tasks extension | 7.6 |
| **A2A binding** | Agent-style workflows | A2A v1.0 | 7.6 |

**Resolution endpoint.** Tunable resolution is also exposed through an OFREP-compatible endpoint, with tunables served as object flags. Any OpenFeature SDK can then resolve prompts with an evaluation context carrying run, experiment, environment and call site, and production canaries use the same targeting as offline experiments. OFREP is at spec version 0.3.0, so it is a compatibility layer beside the native API, not a core dependency.

**Rules for every contract**

- Contract-first with one source: messages are defined once in Protocol Buffers, and JSON Schema 2020-12, OpenAPI 3.2 documents, gRPC stubs, SDKs and mock servers are generated from them, the approach A2A v1.0 takes. Contract tests run in CI.
- Date-based API versions in a header (proposal: `PromptLab-Version: 2026-10-01`), additive changes within a version, explicit deprecation windows.
- Extensions carry reverse-DNS ids and versions. A peer ignores unknown extensions unless they are marked required.
- Every POST that creates work takes an `Idempotency-Key`. Lists use cursors. Prompt and overlay fetches use ETags.
- Errors use RFC 9457 problem details. Content hashes are SHA-256 over RFC 8785 canonical JSON, so equal content has an equal hash in every language.
- Every request carries W3C `traceparent` and `baggage`, so runs, spans and scores join up.

### 7.2 Run Protocol core

| Endpoint | Purpose |
| --- | --- |
| `GET /.well-known/promptlab.json` | Manifest: workflows, call sites and tunables, input and output JSON Schemas, capabilities, limits, convention versions |
| `POST /v1/runs` | Start a run. Scope is a workflow, a step or a single tunable. Returns 202 with a run handle, or streams |
| `GET /v1/runs/{id}` | Status and result |
| `POST /v1/runs/{id}:cancel` | Cooperative cancel |
| `GET /v1/runs/{id}/events` | SSE stream of run events |
| `POST /v1/runs:batch` | Many runs in one request |
| `POST /v1/work:lease`, `POST /v1/work/{id}:heartbeat`, `POST /v1/work/{id}:complete` | Pull mode for runners with no inbound ports |

Run states: queued, leased, running, input\_required, then succeeded, failed, canceled or timed\_out. They align with MCP tasks and A2A task states, so the façades are thin.

### 7.3 Core schemas

Manifest excerpt:

```json
{
  "protocol": "2026-10-01",
  "workflows": [{
    "id": "support-triage",
    "input_schema": {"$ref": "#/$defs/Ticket"},
    "output_schema": {"$ref": "#/$defs/Triage"},
    "call_sites": [{
      "id": "node:summarize", "kind": "llm_call",
      "tunables": [{"id": "triage.summarize", "type": "template",
                    "variables": ["context", "ticket"], "default_hash": "sha256:9f2c…"}]
    }]
  }],
  "capabilities": ["observe", "steer.model", "steer.template", "memo", "capture.io"],
  "limits": {"max_concurrency": 20, "max_run_seconds": 300}
}
```

Run request:

```json
{
  "workflow_id": "support-triage",
  "scope": {"kind": "workflow"},
  "input": {"ticket": {"id": "T-1042", "text": "…"}},
  "overlay": {"ref": "ovl_7c1a", "hash": "sha256:41be…"},
  "mode": "REPLAY",
  "cache": {"scope": "exp_91", "tape": "tape_88d0", "sample_index": 0, "on_miss": "fail"},
  "seed": 17,
  "budget": {"max_tokens": 40000, "max_seconds": 120},
  "effects": {"write": "deny"},
  "trace_context": {"traceparent": "00-…"},
  "idempotency_key": "exp_91/v2/item_14/rep_0"
}
```

Run result:

```json
{
  "run_id": "run_5b3e",
  "status": "succeeded",
  "output": {"category": "billing", "priority": "high"},
  "trace_id": "4bf92f35…",
  "mode_report": {"mode": "REPLAY", "cache_hits": 6, "cache_misses": 0, "replay_fidelity": 1.0, "capture_epoch": "2026-10-14"},
  "overlay_applied": {"hash": "sha256:41be…", "tunables": ["triage.summarize"]},
  "usage": {"input_tokens": 3120, "output_tokens": 410, "cost_usd": 0.0071, "latency_ms": 4180},
  "error": null
}
```

Overlay (immutable, content-addressed):

```json
{
  "overlay_id": "ovl_7c1a",
  "parent": "ovl_6a10",
  "entries": [
    {"call_site": "node:summarize",
     "patch": [{"op": "replace", "path": "/messages/0/template", "value": {"template_ref": "prompt://triage.summarize@v7"}}]},
    {"call_site": "agent:billing/llm",
     "patch": [{"op": "replace", "path": "/tools/refund_lookup/description", "value": "…"},
               {"op": "replace", "path": "/params/temperature", "value": 0}]}
  ]
}
```

**Steer verification.** `overlay_applied` is how the platform knows the override really happened. A run whose applied hash differs from the requested one is invalid, not a tie. This one rule catches the most common silent failure of prompt testing: measuring the old prompt and calling it a new one.

### 7.4 Event stream

```yaml
/v1/runs/{id}/events:
  get:
    responses:
      '200':
        content:
          text/event-stream:
            itemSchema:
              $ref: '#/components/schemas/RunEvent'
```

Event types: `run.started`, `step.started`, `step.finished`, `cache.hit`, `cache.miss`, `tool.denied`, `run.input_required`, `run.finished`. Spans themselves travel by OTLP, not through this stream.

### 7.5 Profiles: reuse the surfaces that already exist

| Profile | Start a run | Carry overlay and mode | Result |
| --- | --- | --- | --- |
| `langgraph-agent-server` | Create a run on a thread (wait or stream) | Per-run `context` or `configurable`, read by the middleware or wrapper | Final state; spans by OTLP |
| `adk-api-server` | Create a session with initial state, then `/run` or `/run_sse` | A session-state key holding the overlay reference, read by the plugin | Event list; spans by OTLP |
| `a2a` | Message send or stream | Message metadata or an A2A extension; `contextId` groups runs | Task artifacts |
| `mcp-tool` | `tools/call`, with a task handle for long runs | Tool arguments or request metadata | Tool result |
| `openai-chat` | Chat completions | Headers plus gateway rewrite | Response |
| `http-generic`, `cli`, `in-process` | POST to the endpoint, run the command, call the function | `traceparent` and `baggage` headers, environment variables | Body or exit output |

A profile is a small adapter in the runner gateway, so a team that already runs an Agent Server or `adk api_server` registers a URL and a profile and is done. The shim inside the workflow still applies the overlay.

### 7.6 Optimizer adapter, MCP facade and A2A binding

**GEPA adapter.** A GEPA candidate (a dictionary of component name to text) is the overlay’s text entries. `evaluate(batch, candidate, capture_traces)` becomes a batch of runs through the Run Protocol with that overlay, returning outputs, scores and, on request, trajectories built from traces. `make_reflective_dataset` slices the traces per component by `call_site_id` and adds evaluator feedback text. Any other optimizer implements the same two methods.

**MCP facade.** PromptLab is also an MCP server. Tools: `list_workflows`, `list_tunables`, `start_experiment` (returns a task handle; clients poll with `tasks/get` and stop with `tasks/cancel`), `get_results`, `compare_variants`, `propose_candidate` (registers a prompt version labeled proposed) and `request_promotion` (always needs a human approval). A coding agent can tune prompts conversationally and open a pull request.

**A2A binding.** A workflow’s agent card is generated from its manifest, and run states map one-to-one to task states.

### 7.7 Services with their own API contracts

Each workflow keeps its own API, and PromptLab adapts to it through a declarative endpoint adapter. No developer has to change a contract or a coding style to be testable.

**What an adapter declares**

- The test deployment’s base URL and a reference to its credentials in the vault.
- A request mapping from dataset item fields to the service’s own request body, path and headers.
- A response mapping that extracts the output and any extra fields to score.
- Optional multi-step handling (create a session, then post a message) and asynchronous patterns (poll a job id until done).

```yaml
# workflows/refund-assistant.yaml
workflow: refund-assistant
framework: adk
invoke:
  profile: http-generic
  base_url: https://refund-assistant.test.internal      # test deployment, read-only database role
  auth: {secret_ref: vault://promptlab/refund-assistant}
  request:
    method: POST
    path: /api/v2/refunds/assist
    body:
      customerId: "{{ item.input.customer_id }}"
      text: "{{ item.input.message }}"
      channel: web
  response:
    output: "$.result.reply"
    fields: {decision: "$.result.decision", order_id: "$.result.orderId"}
  async:                                                  # only if the API is asynchronous
    poll: {path: "/api/v2/jobs/{{ response.jobId }}", until: "$.status == 'done'", every_ms: 500}
openapi: https://refund-assistant.test.internal/openapi.json   # optional: draft the mapping from it
```

**How overlays reach the code.** PromptLab adds W3C `traceparent` and `baggage` headers (run id, overlay reference, mode) to each call. Inside the service, the interceptor loaded by the launcher at the container entrypoint reads that context from the incoming request and applies the overlay to the model calls made while serving it. This works when the service’s web framework is instrumented to extract incoming context (OpenTelemetry auto-instrumentation covers FastAPI, Flask and Django). Work handed to background tasks or queues must carry the context along, and `promptlab doctor` checks that a canary overlay survives the whole request path.

**Onboarding helpers**

- `promptlab adapter draft` proposes the mapping from the service’s OpenAPI document (FastAPI generates one automatically).
- `promptlab adapter test` sends one dataset item and shows the mapped request, the raw response and the extracted output.
- Datasets can be seeded from sampled real requests to the service, so items already arrive in the service’s own shape.

**Contract drift.** The adapter is versioned with the workflow version, and a run fails fast with a clear error when a response path stops matching, instead of scoring empty outputs.

### 7.8 Workflow inputs: the test-case envelope

Every test case stores a workflow’s inputs in one framework-neutral **envelope**, and each workflow’s adapter or native profile translates it into that service’s own request. One dataset format therefore serves every contract. Data the workflow fetches for itself is never faked: the workflow fetches it as usual, from the test database or from recorded results.

| Field | Holds | Used for |
| --- | --- | --- |
| `input` | Request fields (customer id, message, channel); in prompt-only experiments, the prompt’s variables | Building the request |
| `state` | Initial session or graph state (ADK session state, LangGraph initial state) | Creating the session or graph run |
| `turns` | Earlier conversation turns, each with an optional expected reply | Replaying multi-turn conversations |
| `context` | Per-case headers, tenant, locale, a named test identity, file references | Caller context |
| `clock` | “Now” and the time zone the run should see | Reproducible time |
| `expected` | Expected output and decision fields | Scoring only; never sent to the service |
| `tags` | Slices such as intent or customer tier | Stratified splits and reporting |
| `source` | Upload, playground, captured trace (with capture epoch) or generated | Provenance |
| `fixtures` | Records the case depends on (table and key) | The pre-run check |

```json
{
  "id": "item-0142",
  "input":   {"customer_id": "C-1029", "message": "Where is my refund for order 8812?", "channel": "web"},
  "state":   {"customer_tier": "premium"},
  "turns":   [],
  "context": {"headers": {"X-Tenant": "eu-1"}, "identity": "test-user-support",
              "files": ["promptlab-file://billing-inputs/v2/invoice-8812.pdf"]},
  "clock":   {"now": "2026-10-14T09:30:00Z", "tz": "Europe/Berlin"},
  "expected": {"decision": "refund_approved", "order_id": "8812"},
  "tags": ["refunds", "premium"],
  "source": {"kind": "captured", "trace_id": "4bf92f35…", "capture_epoch": "2026-10-14"},
  "fixtures": [{"table": "customers", "key": "C-1029"}, {"table": "orders", "key": "8812"}]
}
```

**How each kind of input reaches the workflow**

- **Request fields** map through the adapter into the service’s body, path, query or headers, and are validated before the run against the workflow’s input schema, taken from its OpenAPI request body or declared in the manifest.
- **State** goes into session creation for the `adk-api-server` profile, and into the initial state for the `langgraph-agent-server` profile. Custom APIs map it like any other field.
- **Turns** replay in order through the adapter’s multi-step mapping, keeping the session between turns.
- **Caller context:** fixed values and secrets come from the adapter and the vault. A case may name a test identity, which the adapter exchanges for a short-lived token, so no credential is ever stored in a dataset.
- **Clock** maps into the request or state, so “today” in the prompt comes from the case, not from the server’s clock.
- **Files** are uploaded into dataset storage, versioned with the dataset, and referenced by URI. The adapter sends them as a multipart upload, a signed URL or inline base64, whichever the service expects.
- **Fetched data** (Postgres rows, retrieval results, internal API responses) is not in the envelope. The case carries only the ids that drive those lookups.

```yaml
# adapter excerpt: envelope fields mapped into a custom contract
request:
  method: POST
  path: /api/v2/refunds/assist
  headers: {X-Tenant: "{{ item.context.headers.X-Tenant }}"}
  body:
    customerId: "{{ item.input.customer_id }}"
    text: "{{ item.input.message }}"
    tier: "{{ item.state.customer_tier }}"
    requestTime: "{{ item.clock.now }}"
  files:
    invoice: {from: "{{ item.context.files[0] }}", as: multipart}
auth: {identity_from: "{{ item.context.identity }}", exchange: vault://promptlab/test-identities}
```

**Pre-run check for referenced records.** A case that points at customer C-1029 fails for the wrong reason if that customer is missing from the test database. Before an experiment starts, PromptLab looks up each case’s `fixtures` through the test deployment’s read-only database settings. Missing records exclude the case and are reported, never scored as workflow failures. For captured cases, fixtures are inferred from the keys in the captured SQL reads, and a case is flagged as **drifted** when those rows have changed since the capture epoch.

**Captured requests become envelopes automatically.** The inbound request body is mapped back to `input` when the adapter’s mapping can be inverted (plain field-to-field paths). Otherwise the body is kept raw and passed through unchanged. ADK session state at creation becomes `state`, the request time becomes `clock`, and captured SQL reads become `fixtures`.

## 8. Domain model

| Entity | Key fields | Notes |
| --- | --- | --- |
| Org, Project, Environment | Tenancy, quotas, region | The isolation boundary (Section 14) |
| Workflow | Id, framework, invoke profile, capabilities | Registered from a manifest |
| WorkflowVersion | Git sha or image digest, manifest hash, workflow-model snapshot | Binds every result to code |
| CallSite | `call_site_id`, kind, parents | Comes from the Canonical Workflow Model |
| Tunable | Id, type, contract (variables, schema, max length), call site | Prompt, tool description, examples, parameters |
| PromptVersion | Content hash, text, variables, parent, author, lineage | Immutable and content-addressed |
| Alias | Name (prod, staging), target version, protection | Movable pointer; protected labels need approval |
| Overlay | JSON Patch entries per call site, hash | Immutable |
| Dataset, DatasetVersion, Item | Input, expected, metadata, split, tape reference | Versioned; splits fixed at creation |
| Tape | Named set of memo entries (key = hash of canonical request), capture epoch, storage URI | Content-addressed |
| EvaluatorVersion | Kind, config, judge model and prompt, calibration stats | Versioned; judges are prompts too |
| Experiment | Workflow version, variants, dataset version, evaluators, mode, budget, stats plan |  |
| Trial | Variant × item × repetition, run id, trace id, status | Failures are kept |
| Score | Trial, evaluator version, value, label, explanation, span id | Step-level when a span id is set |
| OptimizationJob | Algorithm, search space, objectives, splits, budget | Result lineage links to experiments |
| Promotion | From and to alias, evidence, approver, time | Audit record |

**Reproducibility tuple:** workflow version, overlay hash, dataset version, evaluator versions, model and parameters, tape ids, seed. Two results with the same tuple are comparable. Anything else is flagged in the UI. Langfuse’s documentation notes that its experiments always run on the latest dataset version, which is exactly the gap that versioned datasets close.

## 9. Experiment engine

An experiment is a durable workflow, so a crash, deploy or provider outage never loses progress or double-counts a trial.

- **Orchestration.** One durable workflow per experiment (Temporal by default; DBOS on Postgres or Restate are viable at smaller scale), one child workflow per trial, signals for cancel and pause, queries for live progress.
- **Scheduling.** Trials are variant × item × repetition, interleaved so provider drift and time-of-day load hit every variant alike. Seeds are fixed and model versions pinned. Repetitions exist for stochastic workflows and feed a hierarchical bootstrap (Section 12).
- **Concurrency.** Adaptive (AIMD) per runner within the runner’s declared limit, a per-tenant token bucket, and provider rate limits shared through the provider gateway.
- **Idempotency.** Each trial has a deterministic key (experiment, variant, item, repetition), so retries cannot double count.
- **Budgets.** Tokens, dollars and wall-clock. Soft warning at 80 percent, hard stop at 100. Partial results stay valid because the paired analysis uses only items finished by every variant, and coverage is reported.

**Failure taxonomy**

| Failure | Handling | Scored? |
| --- | --- | --- |
| Infrastructure (runner unreachable, lease expired) | Retry with backoff on another runner | No, until retries are exhausted |
| Workflow error or timeout | Recorded as is | **Yes, as a failure** |
| Invalid run (overlay not applied, replay miss, effect violation) | Flag and rerun; repeated invalidity stops the experiment | No, and surfaced loudly |
| Evaluator error | Retry the evaluator, never the run | Score missing, counted in coverage |

Experiment types: A/B, multi-variant, factorial (tunables × model), leave-one-out ablation, and online canary or shadow comparison on sampled production traffic. Testing granularity (one prompt, a subset, everything) follows the matrix in 6.7.

**What a test case can be.** Static input and output pairs cannot test an agent that asks follow-up questions or changes state, so the dataset model supports five kinds.

| Kind | Contents | Scored by |
| --- | --- | --- |
| Static item | Input, optional expected output | Output and trace evaluators |
| Captured run | A production or staging trace with its captured reads | Comparison against the original output, plus guard metrics |
| Conversation | Prior turns and the next user message | Turn-level evaluators |
| Scenario | A simulated user with a persona and hidden goal, an initial environment state, a success check on the final state | State check plus judge, reported as pass^k over k trials |
| Metamorphic group | An item, semantics-preserving transformations (paraphrase, reordering, added context) and the expected relation between outputs | Relation violations; no labels needed |

Simulated users are versioned and calibrated like judges, because the choice of simulator changes measured agent performance. A persona library (cooperative, terse, impatient, withholding) keeps tests from flattering the agent.

**Production traces as datasets.** Captured production runs are sampled and stratified by slice (intent, customer tier, language), then re-executed in FORK mode. The changed call runs on exactly the context production saw, with no curation step. Results are scored against the original output with pairwise judges and guard metrics, and human feedback that arrives later attaches as labels. Captures stay in the data plane under the tenant’s retention rules.

## 10. Evaluation service

Evaluators score outputs and also individual steps, because in a multi-prompt workflow a wrong final answer says little about which prompt failed.

| Evaluator kind | Examples | Runs where |
| --- | --- | --- |
| Deterministic | JSON schema validity, regex, exact match, retrieval precision and recall, tool-call trajectory rules, budget limits | Workers, or inside the customer plane |
| LLM judge | Rubric score, pairwise with order swap, reference-based faithfulness using the retrieved context from the trace | Workers via the provider gateway |
| Human | Annotation queues, gold-set labeling | UI |
| Composite | Weighted or constrained combinations of quality, cost, latency and safety | Stats engine |

**Evaluator contract.** `POST /v1/evaluate` receives the item, the output, the expected value, a trace reference and optional context. It returns scores, each with a name, value or label, an explanation, an optional `span_id` and the evaluator version. The shape deliberately matches the OpenTelemetry `gen_ai.evaluation.result` event, so scores can also be emitted into any OTel pipeline. Custom evaluators in any language implement this endpoint, or run in-process through the SDK.

**Judge governance**

- A judge is a versioned artifact: model, prompt, parameters, and its calibration record.
- Calibrate against a human gold set before trusting a judge, and report agreement (for example Cohen’s kappa) beside every judge-scored result.
- Mitigate known biases by default: order swap in pairwise comparison, length-controlled rubrics, and cross-family ensembles so a judge never grades its own family alone.
- When calibration data exists, report bias-corrected estimates and intervals, following the ICML 2026 method for correct reporting of judge-based scores.
- Judge prompts can themselves be optimized, but only against human labels, never against their own scores.
- Judge cost has its own budget line, tracked as a share of total experiment cost.

**Trace-aware scoring.** Evaluators can read spans: retrieved chunks, tool calls, handoffs. Step scores attach to the span id and call site, which feeds credit assignment in Section 11.

## 11. Optimization service

Optimizers are jobs that call one adapter (7.6) and never see the framework.

| Algorithm | Use it when |
| --- | --- |
| Manual with side-by-side diff | A person is iterating in the UI |
| Few-shot bootstrap | The tunable is an example set |
| Instruction search (MIPRO-style) | One or few prompts, a clean scalar metric |
| **Reflective evolution (GEPA), default** | Several text tunables, rich textual feedback, a small rollout budget, multiple objectives (Pareto) |
| Grid, Bayesian or bandit search | Numeric and enum tunables, or online allocation |

GEPA is the default because it learns from full traces and written feedback, keeps a Pareto set of candidates, and is already integrated by DSPy and MLflow, which is a sign of convergence. It also benefits from a strong reflection model, so the reflection model is a per-tenant setting.

**Spend the budget where it matters.** Candidates are first scored on a small subset, and only survivors advance to the full validation set (successive halving). Because the memo cache is shared, every candidate reuses the same upstream calls, so a generation of candidates costs little more than the calls they actually change.

**Data discipline (enforced by the platform, not by convention)**

- Every dataset has train, validation and test splits fixed at creation. Optimizers see train for reflection and validation for selection. The test split is touched once, by the promotion gate.
- A candidate that wins on validation but loses on test is recorded as overfit, and counts against that optimization run.

**Guardrails**

- Static checks before any run: required variables present, schema constraints kept, maximum length, no secrets or forbidden patterns.
- Length and cost are objectives, which counters prompt bloat from repeated mutation.
- A safety and regression suite is a hard constraint, never a tradeable objective.
- Traces fed to a reflection model are redacted first, and datasets, traces and retrieved text are treated as untrusted data inside reflection and judge prompts (delimited, no tool access).

**Multi-agent credit assignment.** The workflow model gives each agent and call site its own slice of every trace. Reflection and component selection work per tunable. A leave-one-out ablation then confirms that a proposed change actually carries the gain, rather than trusting the optimizer’s story.

**Output.** Each candidate becomes an immutable prompt version with lineage (parent, algorithm, experiment ids) and the label `proposed`. Nothing reaches an environment without passing the promotion gate in Section 12.

## 12. Statistics and promotion gates

A prompt change ships only when a paired comparison, a held-out confirmation and the guard metrics all agree.

### 12.1 The statistics engine

| Question | Method |
| --- | --- |
| Is variant B better than A on the same items? | Paired bootstrap interval on per-item differences, with a permutation test as a cross-check |
| Pass or fail metric | McNemar on discordant pairs (exact test for small counts) |
| Repetitions, or clustered items such as conversations | Hierarchical bootstrap: resample cases (or whole clusters), then repetitions |
| Several variants or metrics | Holm or Benjamini–Hochberg correction, with the primary comparison declared before the run |
| Can this dataset detect the effect that matters? | Power and minimum-detectable-effect planner computed from observed variance, shown before the run |
| Can I stop early? | Anytime-valid confidence sequences, so peeking does not inflate error |
| Judge-scored metric | Bias-corrected estimate when calibration data exists |
| Is the winner real, or a fluke of selection? | An untouched test split, evaluated once |

One practitioner source suggests that fewer than about 100 paired examples is below the resolution of a credible prompt A/B. Treat that as a rule of thumb: the planner computes the real answer from your data’s variance.

**Variance reduction comes from the engine.** Common random numbers (6.3) give both variants identical upstream samples, which tightens paired intervals at no extra cost. The power planner accounts for this, so the item count it recommends reflects the real design.

**Guard metrics** use non-inferiority tests: cost, latency, schema validity and the safety suite must not get worse by more than a stated margin, however much quality improves.

### 12.2 Promotion

Environments are aliases on prompt versions: `proposed`, `candidate`, `staging`, `canary`, `prod`.

1. **proposed**: created by a person or an optimizer and passed the static checks.
2. **candidate**: beats the baseline on validation with an interval above zero, and the guard metrics pass.
3. **staging**: passes the test split and the safety suite in LIVE mode.
4. **canary**: shadow traffic first, then a small share (1 to 5 percent), with online evaluators on a sampled fraction.
5. **prod**: ramps up, with automatic rollback when online scores or guard metrics degrade.

Protected aliases need an approver (Langfuse offers protected prompt labels in the same spirit). Every move writes a Promotion record that links to its evidence.

### 12.3 CI/CD

- `promptlab ci` runs on a pull request: a REPLAY regression against the committed tape set, a report with deltas and intervals, and a failing status on any guard violation.
- A scheduled LIVE run against fresh data catches tape staleness before it misleads anyone.
- In Git-native mode the optimizer opens the pull request itself, and the same gate decides whether it can merge.

## 13. Observability and data layer

**Ingest.** OTLP over gRPC or HTTP through an OpenTelemetry Collector. A normalizer accepts both OpenInference (`llm.*`) and OpenTelemetry GenAI (`gen_ai.*`) attributes, and stores the convention version with every batch. Our own attributes live under `promptlab.*`: workflow id and version, call site id, tunable id and version, overlay hash, experiment and trial ids, mode, tape hit.

**Content capture.** OpenTelemetry leaves prompt and completion content off by default. We turn it on for experiment runs only. A redaction processor runs in the customer plane before export, and large payloads go to object storage with a reference in the span.

**Storage**

| Store | Holds | Notes |
| --- | --- | --- |
| Postgres | Registry, overlays, datasets, experiments, promotions, tenancy | Row-level security per tenant |
| ClickHouse | Spans and scores | Denormalized and append-only, so dashboards need no join at query time (Langfuse v4 moved to this shape) |
| Redis (or Valkey) | Queues, caches, rate limits, leases | Ephemeral by design |
| Object storage | Payloads, tapes, dataset files, exports | Per-tenant prefix and keys |

This is the same four-store shape Langfuse v4 runs in production. At small volume Postgres alone can hold spans in Phase 0, with ClickHouse arriving when volume demands it.

**Retention.** Per tenant and per class: raw content shortest (a proposal is 30 days), metadata and scores longer, audit records longest. Tapes expire with the datasets that reference them.

**Views the data must support.** The trace tree by call site, a side-by-side diff of two runs, score slices by dataset tag and by call site, and the reproducibility tuple on every result.

**Ingest scale path.** Start with Redis and object-storage buffering, the pattern Langfuse uses. Move to a Kafka-compatible log only when sustained ingest requires it.

## 14. Security and multi-tenancy

| Area | Design |
| --- | --- |
| Isolation | Org, project and environment. Postgres row-level security; tenant key in the ClickHouse primary key plus row policies; per-tenant object prefix with envelope encryption (KMS); bring-your-own-bucket option |
| Identity | OIDC and SAML SSO, SCIM provisioning, roles (viewer, experimenter, approver, admin). API keys scoped to project, environment and role. Short-lived, audience-bound runner tokens; mTLS option for push runners |
| Runner trust | The control plane never executes customer code. It sends overlays and inputs and receives results. Managed sandboxes run in isolated containers with no ambient credentials and an egress allowlist |
| Network | Pull-mode agents are outbound-only. Push mode uses allowlisted runner URLs, blocks private and metadata address ranges (SSRF), and signs webhooks with timestamps |
| Data residency | Split plane: traces with content stay in the customer’s region or VPC. In metadata-only mode the control plane stores hashes, scores and metadata. Evaluators and the tape store can run in the customer plane |
| Prompt-injection hygiene | Datasets, traces and retrieved text are untrusted inside judge and reflection prompts: delimited, no tool access, outputs validated |
| Effects safety | Writes denied by default; per-experiment allowlist; sandbox targets for LIVE runs |
| Secrets | Provider keys in a vault (BYOK). Secrets never travel in overlays or traces, and a scanner checks prompts and traces |
| Audit | Append-only log of overlay creation, promotions, key use and data access, exportable to a SIEM |

This design supports SOC 2-style controls. It does not by itself make any certification claim.

## 15. Scalability and reliability

- **Stateless scale-out.** API, runner gateway, evaluator and optimizer workers scale horizontally. State lives in the four stores and in the durable orchestrator.
- **Sizing formula.** Spans per second = concurrent trials × spans per trial ÷ average trial seconds. Example only: 2,000 concurrent trials × 30 spans ÷ 20 s = 3,000 spans per second. Replace the inputs with your real numbers (open question 3).
- **Backpressure.** Runner leases, adaptive concurrency, per-tenant token buckets, circuit breakers on providers, and priority queues so CI runs are never starved by bulk optimization.
- **Caching.** An LLM response cache keyed by the canonical request for judge and reflection calls (record once, replay), and an SDK-side cache of prompts and overlays with ETag and stale-while-revalidate.
- **Reliability targets (proposal).** Control API availability 99.9 percent; trial dispatch p95 under 5 seconds; experiments survive a control-plane restart because they are durable; no span loss because ingest is queued and replayable.
- **Disaster recovery.** Postgres point-in-time recovery, ClickHouse replicas, object-store versioning. RPO and RTO to be set with you.
- **Regions.** One control plane per region with data pinned by tenant, which is also how residency requirements are met.

## 16. Technology choices

| Concern | Default | Alternatives and notes |
| --- | --- | --- |
| Services | A modular-monolith control plane in Python (evaluators, optimizers and statistics live there), with Go for the runner gateway, ingest and memo-store hot paths | TypeScript for the API layer if that matches your team |
| Contracts | Protobuf as the source; generated JSON Schema 2020-12, OpenAPI 3.2, gRPC stubs and SDKs |  |
| Orchestration | Temporal | DBOS on Postgres for a smaller footprint; Restate |
| Trace ingest | OpenTelemetry Collector with a custom processor |  |
| Analytics store | ClickHouse | Postgres alone in Phase 0 |
| Queue and cache | Redis or Valkey | Kafka-compatible log at higher ingest |
| Object store | S3-compatible | Any cloud blob store |
| Optimizers | `gepa` library, DSPy optimizers | Custom behind the same adapter |
| Provider gateway | LiteLLM-class proxy | In-house; Agent Lightning also uses a LiteLLM proxy |
| SDKs and shims | Python and TypeScript first | ADK ships in Python, TypeScript, Go and Java, so its plugin must follow usage across those |
| Web UI | React with a diff and trace viewer |  |

## 17. Roadmap

Durations are planning estimates for a team of roughly four to six engineers, not commitments.

| Phase | Scope | Exit gate |
| --- | --- | --- |
| **0. Standalone Prompt Lab** (about 4–6 weeks) | Prompt registry with versions and aliases, datasets from uploads, standalone prompt runs across Gemini and other models, deterministic and judge evaluators, paired statistics, model-upgrade experiments, lead approval for promotions, a playground UI on the same API. Postgres only (6.8) | A real ADK prompt is improved, or a model upgrade is decided, by a paired comparison on a few hundred items and approved by a lead |
| **1. Connect ADK services** (about 6 weeks) | Launcher with the Google GenAI interceptor and the ADK instrumentor (tracing from scratch), endpoint adapters for your service contracts (7.7), capture of real prompt inputs including Postgres reads, read-only test deployments (6.5), a trace view | Standalone tests run on inputs captured from real ADK calls, and no test run can write to production |
| **2. In-workflow testing** (about 8 weeks) | Memo cache with FORK and REPLAY, the ADK plugin for templates, write blocking with scored attempted writes, CI gate, LangGraph support (Agent Server profile, OpenInference instrumentor) | A prompt change is tested inside a full ADK workflow on real data, and one LangGraph workflow is onboarded |
| **3. Optimize and scale** (ongoing) | GEPA optimization, production traces as datasets, scenarios with simulated users, canary rollout, ClickHouse once the thresholds are crossed, MCP facade | The optimizer beats the baseline on the test split, and reliability targets are met |

## 18. Risks and open questions

| Risk | Mitigation |
| --- | --- |
| OpenTelemetry GenAI conventions keep evolving | Normalizer layer, version stored per batch, support both attribute families |
| Shims break as frameworks release | Conformance kit, published support matrix, pinned versions |
| The optimizer overfits the judge (Goodhart) | Test split, human gold set, adversarial probes, safety as a hard constraint |
| Tapes go stale and replay stops matching live | Tape keys include upstream hashes, scheduled LIVE confirmation, tape age shown in results |
| Side effects during LIVE runs | Default deny, effect tagging, sandbox targets |
| Sensitive data reaches reflection or judge models | Redaction, per-tenant model choice, metadata-only mode |
| Non-determinism hides small gains | Repetitions, paired design, power planner |
| Adoption friction | One-line shims, `init` and `doctor`, benchmark mode first |

**Risks this revision introduces, and their mitigations**

- **Interceptor drift.** Model SDKs change often. Mitigate with a conformance suite per SDK version, a published support matrix, and the gateway as a fallback.
- **False cache hits.** A missed volatile field or an unpinned model alias could return a stale answer. Mitigate with keyed model versions, `doctor` detection of unstable fields, and sampled live re-execution of hits (6.4).
- **Payload capture is sensitive.** Recording reads means storing customer data. Mitigate by capturing in experiment runs only, keeping captures in the data plane, encrypting per tenant and applying short retention.
- **Standards still settling.** OFREP is at 0.3.0 and the OpenTelemetry GenAI conventions are in Development status, so both sit behind adapter layers.

**Decisions recorded, and what is still open**

1. **Frameworks:** ADK first, LangGraph second.
2. **Data and traces:** data in PostgreSQL; no tracing today, so PromptLab provides it.
3. **Scale:** start on Postgres and measure; thresholds in Section 1.
4. **Starting point:** standalone prompt testing and model upgrades (6.8). Build Phase 0 on your own Postgres, and revisit embedding Langfuse or MLflow when a trace view is needed in Phase 1.
5. **Production writes:** yes, so test runs use read-only test deployments enforced in Postgres (6.5).
6. **Approvals:** leads approve promotions; no compliance regime assumed.

**Still open**

- Which candidate model is the first upgrade, and which prompts matter most?
- Is a staging copy or a read replica of the production database available for test deployments?
- Do any services hand work to background jobs or queues? This affects context propagation (7.7).
- How many workflows and prompts will the first teams bring? This sizes Phase 0.

## 19. Worked examples

All worked examples now live in Integration cookbook: getting-started examples for ADK, LangChain and LangGraph, the streaming refund service end to end, catalogs of ADK and LangGraph scenarios, and every supported service shape. Architecture, user journey, API flow and dataflow pictures are in Diagrams.

## 20. Sources

These pages surfaced in web searches in October 2026 and were read as excerpts, not in full. Treat versions and dates as approximate until you open the source.

- Observability standards: [OpenTelemetry GenAI conventions overview](https://www.truefoundry.com/blog/opentelemetry-genai-semantic-conventions) · [GenAI agent spans](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-agent-spans) · [OpenInference instrumentors](https://github.com/Arize-AI/openinference) · [Phoenix instrumentor list](https://pypi.org/project/arize-phoenix/20.0.0/)
- Optimization: [dspy.GEPA](https://dspy.ai/api/optimizers/GEPA) (paper: arXiv 2507.19457) · [GEPA adapters](https://www.mintlify.com/gepa-ai/gepa/api/adapters/overview) · [MLflow optimize\_prompts](https://mlflow.org/docs/3.5.1/genai/prompt-registry/optimize-prompts) · [Agent Lightning](https://pypi.org/project/agentlightning/)
- Eval platforms: [Braintrust remote evals](https://braintrust.dev/docs/evaluate/remote-evals) · [Langfuse core concepts](https://langfuse.com/docs/evaluation/core-concepts) · [Langfuse UI vs SDK experiments](https://langfuse.com/resources/engineering/migrate-from-openai-evals) · [Langfuse v4 stack](https://railway.com/deploy/langfuse-v4)
- Protocols: [MCP 2026-07-28 specification](https://blog.modelcontextprotocol.io/posts/2026-07-28/) · [A2A v1.0 builder guide](https://aaif.io/blog/a2a-v1-0-a-builder-s-guide-part-1-discovery-tasks-and-clients) · [OpenAPI 3.2 release notes](https://github.com/OAI/OpenAPI-Specification/releases)
- Frameworks: [ADK callbacks](https://google.github.io/adk-docs/callbacks) · [ADK plugins](https://google.github.io/adk-docs/plugins) · [ADK api\_server](https://google.github.io/adk-docs/get-started/testing) · [LangChain middleware and context engineering](https://docs.langchain.com/oss/python/langchain/context-engineering.md) · [LangGraph time travel](https://docs.langchain.com/oss/python/langgraph/use-time-travel) · [LangGraph Agent Server (A2A endpoint)](https://docs.langchain.com/langsmith/agent-server-api/a2a/a2a-json-rpc.md)
- Evaluation practice: [LLM-judge bias correction, ICML 2026](https://proceedings.mlr.press/v306/lee26bl.html) · [LLM-judge best practices](https://futureagi.com/blog/llm-as-judge-best-practices-2026) · [Paired comparison with intervals](https://oneuptime.com/blog/post/2026-08-31-compare-prompts-models-confidence-intervals/view) · [Anytime-valid intervals for judged evals](https://pypi.org/project/truescore/0.6.0/) · [A/B testing prompts](https://futureagi.com/blog/ab-testing-llm-prompts-best-practices-2026) · [Record and replay evals](https://github.com/Eval-core/evalcore) · [Agent eval harness with replay](https://www.kunalganglani.com/blog/agent-evaluation-harness-replay)
- Durable execution: [DBOS and agents on Postgres](https://dev.to/mr_manushukla/durable-ai-agents-without-temporal-exactly-once-workflows-on-postgres-with-dbos-2026-5a6n)

* Added in this revision: [OFREP overview](https://openfeature.dev/docs/reference/other-technologies/ofrep/) · [OFREP specification 0.3.0](https://openfeature.dev/docs/reference/other-technologies/ofrep/openapi/) · [UserProxyBench (simulated users)](https://arxiv.org/pdf/2609.38043) · [τ-bench overview](https://www.layer3labs.io/guides/tau-bench-explained) · [Simulated-user personas and pass^k](https://potatoannotator.readthedocs.io/en/latest/agent-evaluation/persona_simulation/) · [Semantic invariance (metamorphic testing)](https://arxiv.org/html/2603.13173) · [A2A protocol architecture](https://tyk.io/learning-center/a2a-protocol-architecture-and-technical-specification/)
