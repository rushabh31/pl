# PromptLab requirements

Requirements for all four phases, written to be testable: each has an id, a priority and an acceptance check. Design rationale lives in the main tab; this tab says what must be built.

## 1. Conventions

- **Ids** are `AREA-NNN` and never reused. Areas: PR prompt registry, DS datasets, RUN prompt runs, EXP experiments, EV evaluators, ST statistics, GOV governance, CMP contract compatibility, INT integration, ADP adapters, SAFE production safety, MEMO memo cache, OVL overlays, WR write handling, WEXP workflow experiments, CI, LG LangGraph, OPT optimization, PROD production traces, SCN scenarios, ROLL rollout, NFR non-functional, UX user interface.
- **Priority:** MUST is required for the phase’s exit gate; SHOULD is expected unless a documented trade-off defers it; COULD is optional.
- **Phase** is the phase in which a requirement must first be met. Later phases keep it.
- **Roles:** *viewer* reads; *editor* creates prompts, datasets and experiments; *lead* approves promotions and alias moves; *admin* manages providers, keys, roles and retention.
- Numeric targets marked *(proposed)* are starting points to confirm with load tests and with your team.

## 2. API contract compatibility

PromptLab must test any workflow service as it is, whatever request and response shape its developer chose. ADK and LangGraph services get native profiles; everything else goes through the declarative endpoint adapter (main tab, 7.7).

**Support matrix**

| Contract style | How PromptLab supports it | Phase |
| --- | --- | --- |
| Synchronous JSON over HTTP, any shape | Request template + JSONPath response mapping | 1 |
| Asynchronous job (submit, then poll a job id) | Adapter `async.poll` with completion condition and timeout | 1 |
| Asynchronous with webhook callback | PromptLab callback URL injected into the request; correlation by run id | 2 |
| Multi-step session (create session, then send messages) | Adapter `steps` with values passed between steps | 1 |
| Streaming responses (SSE or JSON Lines) | Adapter aggregation rule (concatenate text, take final event) | 1 |
| Multi-turn conversations | Dataset item holds the turns; adapter replays them in order with the session | 2 |
| ADK `adk api_server` | Native `adk-api-server` profile: create session with state, then `/run` or `/run_sse` | 1 |
| ADK agent behind a custom API | `http-generic` adapter plus the ADK plugin inside the service | 1 |
| LangGraph Agent Server | Native `langgraph-agent-server` profile: threads, runs, per-run context | 2 |
| LangGraph graph behind a custom API | `http-generic` adapter plus PromptLab instrumentation inside the service | 2 |
| In-process Python callable | `in-process` profile: module path and function | 1 |
| gRPC service | `grpc` profile from the service’s proto descriptors | 3 |
| Queue-driven workflow | Publish a message and await a result topic, correlated by run id | 3 |

**Requirements**

| Id | Requirement | Priority | Phase | Acceptance |
| --- | --- | --- | --- | --- |
| CMP-001 | Impose no shape on a service’s request or response; any JSON-over-HTTP contract is supported through declarative mapping | MUST | 1 | Three services with different contracts are onboarded with no code change to the services |
| CMP-002 | Request mapping supports body, path, query and header values from dataset item fields, constants and secrets, including nested objects and arrays | MUST | 1 | Mapping unit tests cover every value location |
| CMP-003 | Response mapping extracts the primary output and named extra fields with JSONPath; a missing required path fails the trial with a contract-drift error | MUST | 1 | A renamed response field produces a drift error, not an empty score |
| CMP-004 | Auth types: API key header, OAuth 2.0 client credentials bearer token, mTLS; secrets referenced from the vault, never stored in adapter files | MUST | 1 | Each auth type passes an integration test |
| CMP-005 | Every call carries W3C `traceparent` and `baggage` (run id, overlay reference, mode); `doctor` verifies a canary overlay survives the full request path, including background work | MUST | 1 | Doctor reports pass or fail per hop |
| CMP-006 | Adapter results normalize to `{output, fields, usage, error, latency}`; non-2xx responses and timeouts are recorded as scored failures with the response body kept | MUST | 1 | A 500 response appears as a failed trial with body visible |
| CMP-007 | Draft an adapter from an OpenAPI 3.0, 3.1 or 3.2 document | SHOULD | 1 | FastAPI-generated specs produce a working draft |
| CMP-008 | `promptlab adapter test` sends one item and shows mapped request, raw response and extracted output | MUST | 1 | Available in CLI and UI |
| CMP-009 | Adapters are versioned with the workflow version; experiments record the adapter version | MUST | 1 | Changing an adapter creates a new version |
| CMP-010 | Per-service concurrency and rate limits are configurable and never exceeded; an idempotency-key header can be added per trial | MUST | 1 | Load test shows limits respected |
| CMP-011 | Webhook, streaming and multi-turn patterns from the matrix | SHOULD | 2 | One service of each pattern onboarded |
| CMP-012 | gRPC and queue-driven profiles | COULD | 3 | One service of each onboarded |
| CMP-013 | The adapter schema is published as JSON Schema so editors validate adapter files | SHOULD | 1 | Schema linked from docs and used by the UI editor |
| CMP-014 | SSE responses are aggregated by a declared final event, with output and fields extracted from it; time to first event and event counts are recorded as metrics; a stream that ends without the final event is a scored failure that keeps the events received | MUST | 1 | The refund service example (Integration cookbook, example E) runs a dataset end to end over SSE |

**Adapter features from the Integration cookbook**

| Id | Requirement | Priority | Phase | Acceptance |
| --- | --- | --- | --- | --- |
| CMP-015 | A placeholder that is an entire value keeps the item’s JSON type; a placeholder inside a string renders as text | MUST | 1 | Arrays, numbers and booleans arrive typed (cookbook A2) |
| CMP-016 | GET requests with query mapping and automatic URL encoding; warning above 8 KB of URL *(proposed)* | MUST | 1 | Cookbook A3 passes |
| CMP-017 | Plain-text responses (`format: text`, `output: $body`) | MUST | 1 | Cookbook A6 passes |
| CMP-018 | `accept_status`, `error_when` and `error_message` separate real failures from expected business outcomes | MUST | 1 | A 200 error body is a failure; an accepted 422 is scored against `expected` (A8) |
| CMP-019 | Batch endpoints: cases grouped per request, results matched back by id, same batch composition across variants | SHOULD | 2 | Cookbook A7 passes, paired |
| CMP-020 | Multi-step `steps` with saved values, `for_each` over turns, and per-turn scoring | MUST | 1 (steps), 2 (turns) | Cookbook B3 and B4 pass |
| CMP-021 | Interrupts: detect a pause, record it, and resume with a response scripted by the test case | SHOULD | 2 | Both approve and reject paths of B5 pass |
| CMP-022 | JSON Lines and chunked streams with `final_when` or `aggregate` | SHOULD | 2 | Cookbook B6 passes |
| CMP-023 | Results read from a database with a read-only role after a job completes (`result.from_sql`) | SHOULD | 2 | Cookbook B9 passes |
| CMP-024 | WebSocket and message-bus (Pub/Sub, Kafka) profiles with correlation ids and context in handshake headers or message attributes | COULD | 3 | Cookbook B7 and B8 pass |
| CMP-025 | Context helpers (`pl.context.inject`, `pl.context.attach`) for job records, custom queues and sockets; `doctor` verifies every hop with a canary overlay | MUST | 1 | A hop that drops context is named in the doctor report |
| CMP-026 | Call-site ids prefixed with the service name, so one overlay can span several services | MUST | 2 | Cookbook C6 overlay applies in both services |
| CMP-027 | Request-level variants (`request_patch`) for black-box and vendor services | SHOULD | 1 | Cookbook C9 runs a two-variant experiment |
| CMP-028 | Per-host effect rules for outbound HTTP (`mock`, `block`, `allow`), with mocked calls recorded for evaluators | MUST | 2 | Cookbook D2: email mocked and asserted, payments blocked |
| CMP-029 | Response-cache detection: a trial with no model call under an active overlay is invalid, not scored | MUST | 2 | Cookbook D3 with the cache on yields invalid trials |
| CMP-030 | Test identities exchanged for short-lived end-user tokens (OAuth token exchange or client credentials), with results sliceable by identity | MUST | 1 | Cookbook D4 passes for two roles |
| CMP-031 | Limits: concurrency, rate per minute, timeouts, retries on 429 and 503 honoring `Retry-After` | MUST | 1 | Load test respects every limit (D5) |

## 3. Phase 0: standalone Prompt Lab and model upgrades

**Goal:** test, refine and compare prompts and models outside any workflow, with statistically sound results and lead-approved promotion. **Exit gate:** a real ADK prompt is improved, or a model upgrade is decided, by a paired comparison on a few hundred items, and a lead approves it.

### Prompt registry

| Id | Requirement | Priority | Acceptance |
| --- | --- | --- | --- |
| PR-001 | Store prompt versions as immutable, content-addressed records: messages, typed variables (JSON Schema), optional output schema, model and parameters, change note, author, parent version | MUST | Saving an edit creates a new version; identical content yields an identical hash |
| PR-002 | Templates use ADK’s `{variable}` syntax; saving validates that every placeholder is declared and every declared variable is used | MUST | An undeclared placeholder blocks the save with a clear message |
| PR-003 | Movable aliases per prompt (`dev`, `staging`, `prod` and custom); `staging` and `prod` are protected and move only through an approved promotion | MUST | An editor cannot move a protected alias directly |
| PR-004 | Diff any two versions: text, variables, output schema, model and parameters | MUST | Diff view and API return all changed fields |
| PR-005 | Organize prompts by workflow, team and tags; search by name, text and tag | SHOULD | Search returns results within 1 second for 10,000 versions |
| PR-006 | `promptlab import adk` extracts instruction strings from ADK agent code into prompt versions, keeping agent names as prompt ids | SHOULD | Import of a sample multi-agent ADK app creates one prompt per agent |
| PR-007 | Resolve a prompt by alias through an API with ETag caching | MUST | A second fetch with the ETag returns 304 |

### Datasets

| Id | Requirement | Priority | Acceptance |
| --- | --- | --- | --- |
| DS-001 | Create datasets from CSV, JSON or JSONL uploads; an item holds variable values, an optional expected output and metadata tags | MUST | Upload with a column-mapping preview |
| DS-002 | Datasets are versioned and immutable; experiments pin a version | MUST | Editing items creates a new dataset version |
| DS-003 | Fixed train, validation and test splits assigned at creation, optionally stratified by a tag | MUST | Split membership never changes within a version |
| DS-004 | Validate items against the prompt’s variable schema before a run and list mismatches | MUST | A run with invalid items does not start until fixed or excluded |
| DS-005 | Save any playground run as a dataset item | SHOULD | One click from a result |
| DS-006 | Mark datasets as containing personal data; restrict access to named roles | SHOULD | Viewers without access cannot open the items |

### Prompt runs and models

| Id | Requirement | Priority | Acceptance |
| --- | --- | --- | --- |
| RUN-001 | `POST /v1/prompt-runs` runs a stored or inline prompt with variables and an optional model override, returning output, parsed output, schema validity, usage, cost, latency and finish reason | MUST | Contract tests pass against the OpenAPI document |
| RUN-002 | Model catalog with Gemini first and at least one other provider through the provider gateway; provider keys held in a vault | MUST | Same prompt runs on two providers from the UI |
| RUN-003 | Validate structured output against the output schema and record safety blocks and refusals | MUST | Schema-invalid and blocked outputs are flagged per item |
| RUN-004 | Provider rate limits respected with retries and backoff; exhausted retries recorded as failures | MUST | Provider 429s never abort an experiment |
| RUN-005 | Stream playground output over SSE | SHOULD | First tokens appear while the call runs |
| RUN-006 | Pass seed and sampling parameters where the provider supports them, and record when it does not | SHOULD | Run record shows seed support |
| RUN-007 | Optional response cache keyed by canonical request for repeated runs | SHOULD | Re-running an unchanged arm costs nothing |

### Experiments, evaluators and statistics

| Id | Requirement | Priority | Acceptance |
| --- | --- | --- | --- |
| EXP-001 | An experiment declares target prompt, dataset version and split, variants (prompt version, model, parameters), evaluators, repetitions and budget, through UI, API or YAML file | MUST | The same YAML runs from CLI and UI with identical results |
| EXP-002 | Experiments run durably: they survive restarts and can be paused, resumed and cancelled | MUST | Killing a worker mid-run loses no completed trial |
| EXP-003 | Trials are interleaved across variants | MUST | Trial order log shows interleaving |
| EXP-004 | Budgets in tokens and money: warning at 80 percent, hard stop at 100; partial results stay valid on completed paired items | MUST | A stopped experiment reports coverage |
| EXP-005 | A model-upgrade template with baseline, candidate model and optional tuned prompt | MUST | Created in under a minute from an existing prompt |
| EXP-006 | Store the reproducibility tuple (prompt versions, dataset version, evaluator versions, model and parameters, seed) | MUST | Re-running a tuple reproduces deterministic scores |
| EV-001 | Deterministic evaluators: schema validity, exact match, regex, contains, JSON field match, length, refusal detection | MUST | Each has unit tests |
| EV-002 | LLM-judge evaluators: rubric score and pairwise with order swap; judge prompts are versioned prompts | MUST | Pairwise results show both orders |
| EV-003 | Warn when the judge model shares a family with an arm being judged | SHOULD | Warning shown at experiment creation |
| EV-004 | Custom evaluators through the `POST /v1/evaluate` contract | SHOULD | A sample evaluator in another language works |
| EV-005 | Blind human review: rate or rank outputs side by side; labels kept as a gold set | SHOULD | Reviewer cannot see which variant is which |
| EV-006 | Judge agreement with human labels (Cohen’s kappa) shown once at least 50 labels exist *(proposed threshold)* | SHOULD | Agreement shown beside judge scores |
| ST-001 | Paired comparison per item with a bootstrap 95 percent interval on the mean difference | MUST | Matches a reference implementation on test data |
| ST-002 | McNemar test for pass or fail metrics | MUST | Matches a reference implementation |
| ST-003 | Repetitions handled with a hierarchical bootstrap | SHOULD | Intervals widen correctly with fewer items |
| ST-004 | Holm correction when more than two variants are compared | MUST | Report states the correction |
| ST-005 | Guard metrics (cost, latency, schema validity) tested for non-inferiority against declared margins | MUST | A cheaper but worse variant is not labelled better |
| ST-006 | Verdicts are `better`, `worse` or `inconclusive`, derived from intervals, never from point estimates alone | MUST | No verdict when the interval crosses zero |
| ST-007 | Power planner estimates the detectable effect before a run | SHOULD | Shown in the experiment builder |
| ST-008 | The test split is evaluated once per promotion and its use is recorded | MUST | A second test-split run is flagged in the audit log |

### Governance

| Id | Requirement | Priority | Acceptance |
| --- | --- | --- | --- |
| GOV-001 | Roles viewer, editor, lead, admin, scoped per project | MUST | Permission tests per role |
| GOV-002 | Promotion request bundles evidence (validation experiment, test-split result, guard metrics); a lead approves or rejects with a comment | MUST | Alias moves only after approval |
| GOV-003 | One-action rollback of an alias to its previous version | MUST | Rollback is audited |
| GOV-004 | Append-only audit log: version creation, alias moves, approvals, key use, dataset access | MUST | Export to CSV and to your log platform |
| GOV-005 | Single sign-on with OIDC | SHOULD | Users sign in with the company identity provider |

## 4. Phase 1: connect ADK services

**Goal:** observe real ADK services without code changes, capture the real inputs of each prompt (including Postgres reads) as test data, and invoke services through their own contracts, with no path to production writes. **Exit gate:** standalone tests run on inputs captured from real ADK calls, and no test run can write to production.

### Integration and tracing

| Id | Requirement | Priority | Acceptance |
| --- | --- | --- | --- |
| INT-001 | `promptlab run --` launcher loads the Google GenAI interceptor, the OpenInference ADK instrumentor and HTTP and SQL instrumentation with no code change | MUST | A sample ADK service is traced after changing only its entrypoint command |
| INT-002 | Traces show agent, tool and model spans with prompts and outputs, stored in Postgres and viewable in the trace view | MUST | A multi-agent request renders as one tree |
| INT-003 | Call-site ids derived from ADK agent names (`agent:<name>/llm`), stable across deployments | MUST | Two deployments of the same code yield identical call-site ids |
| INT-004 | The ADK plugin records, per model call, the instruction template and the session-state values filled into it | MUST | Captured variables reproduce the rendered instruction exactly |
| INT-005 | Capture Postgres reads (statement, parameters, rows up to a configurable limit) through psycopg, asyncpg and SQLAlchemy instrumentation | MUST | Retrieved rows appear on the span |
| INT-006 | Turn captured calls into dataset items for a chosen prompt: select traces, extract variables, create a dataset version | MUST | A dataset built from 100 traces runs in a Phase 0 experiment unchanged |
| INT-007 | Capture sampling and redaction (sampling rate, field drops, pattern masking) applied in-process before data leaves the service | MUST | Masked values never reach PromptLab storage |
| INT-008 | Fail open: if PromptLab is unreachable, the service behaves exactly as without it | MUST | Chaos test with PromptLab down shows unchanged responses |
| INT-009 | Interceptor overhead under 5 ms p95 per model call, with capture exported asynchronously *(proposed)* | MUST | Benchmark in CI |
| INT-010 | Optional runtime prompt resolution: a service can load a prompt by alias with caching and fall back to its code default | SHOULD | Moving `prod` changes the served prompt without a deploy |

### Endpoint adapters

| Id | Requirement | Priority | Acceptance |
| --- | --- | --- | --- |
| ADP-001 | Register a workflow with an endpoint adapter file (request mapping, response mapping, auth reference, limits) | MUST | Registration via UI, API and YAML |
| ADP-002 | Synchronous, asynchronous-poll and multi-step session interactions | MUST | One sample service of each kind passes |
| ADP-003 | Native `adk-api-server` profile: session created with state from item fields, then `/run` | MUST | A stock `adk api_server` app runs a dataset |
| ADP-004 | Run a workflow over a dataset end to end and score its output (benchmark mode, no prompt overrides yet) | MUST | A workflow experiment completes with scores and traces |
| ADP-005 | Contract-drift detection per CMP-003 | MUST | See CMP-003 |

### Workflow inputs (test-case envelope, main tab 7.8)

| Id | Requirement | Priority | Acceptance |
| --- | --- | --- | --- |
| DS-007 | Test cases use one envelope: `input`, `state`, `turns`, `context`, `clock`, `expected`, `tags`, `source`, `fixtures`. Phase 0 prompt datasets use `input` for prompt variables | MUST | One dataset runs against two services with different contracts through their adapters |
| DS-008 | Each workflow has an input schema, derived from its OpenAPI request body or declared in the manifest; `input` is validated before a run, with errors listed per item | MUST | A missing required field blocks the item before any call is made |
| DS-009 | `state` is mapped into ADK session creation by the `adk-api-server` profile, and into LangGraph initial state by the `langgraph-agent-server` profile (Phase 2); an optional state schema validates it | MUST | Captured ADK state reproduces the original rendered instruction |
| DS-010 | `expected` is never sent to a service; adapter validation rejects any request mapping that references it | MUST | A mapping that uses `item.expected` fails validation |
| DS-011 | Test cases never store secrets: a scanner rejects token-like values, and a case names a test identity that the adapter exchanges for a short-lived credential from the vault | MUST | An uploaded dataset containing a bearer token is rejected |
| DS-012 | Files are uploaded into dataset storage, versioned with the dataset and referenced by URI; adapters send them as multipart upload, signed URL or base64; 25 MB per file *(proposed)* | SHOULD | A PDF attachment reaches a service in each of the three forms |
| DS-013 | Pre-run check: `fixtures` are looked up through the test deployment’s read-only database settings; cases with missing records are excluded and reported, never scored as failures | MUST | A case referencing a deleted customer appears under excluded, with the missing key named |
| DS-014 | Fixtures are inferred for captured cases from the keys in captured SQL reads; a case is flagged drifted when those rows differ from the capture epoch | SHOULD | Changing a referenced row flags the case as drifted |
| DS-015 | Captured requests become envelopes: the inbound body maps back to `input` when the adapter mapping is invertible, otherwise it is kept raw and passed through; ADK session state at creation becomes `state`, the request time becomes `clock` | MUST | A captured request replayed through its adapter produces the original request body |
| DS-016 | `clock` maps into the request or state per adapter; cases without a clock use the dataset default, and the result says so | SHOULD | Two runs a week apart render the same date in the prompt |
| DS-017 | Multi-turn `turns` replay in order within one session, with optional expected replies per turn (Phase 2) | SHOULD | A five-turn conversation replays and scores each turn |

### Production safety

| Id | Requirement | Priority | Acceptance |
| --- | --- | --- | --- |
| SAFE-001 | Workflows register test deployments separately; PromptLab refuses to send test traffic to any URL tagged production | MUST | A production-tagged URL is rejected at registration and at run time |
| SAFE-002 | `doctor` verifies the test deployment’s database session is read-only (`default_transaction_read_only`, role grants) and blocks experiments otherwise | MUST | A writable role fails the check |
| SAFE-003 | Runners reach only allowlisted test URLs | MUST | Requests to other hosts are refused |
| SAFE-004 | Captured data inherits dataset access rules and a retention period (30 days for raw payloads, proposed) | MUST | Expired payloads are deleted by a scheduled job |

## 5. Phase 2: in-workflow testing and LangGraph

**Goal:** test prompt changes inside full workflows on real data, cheaply and deterministically, and onboard LangGraph. **Exit gate:** a prompt change is tested inside a full ADK workflow on real data, and one LangGraph workflow is onboarded.

| Id | Requirement | Priority | Acceptance |
| --- | --- | --- | --- |
| MEMO-001 | Memo cache keyed by canonical request (model version, sample index, capture epoch) for model calls and captured reads | MUST | Re-running an unchanged workflow is served entirely from cache |
| MEMO-002 | Modes LIVE, FORK, REPLAY and HYBRID; REPLAY misses fail the trial | MUST | Each mode has an integration test |
| MEMO-003 | Volatile-field masks; `doctor` detects fields that differ between identical runs | MUST | Timestamps in prompts no longer cause misses |
| MEMO-004 | Every result reports mode, hits, misses, replay fidelity and capture age | MUST | Shown in API results and UI |
| MEMO-005 | A sampled fraction of hits (1 percent, proposed) is re-executed live and compared | SHOULD | Mismatch rate is charted |
| MEMO-006 | Common random numbers: repetition r of every variant shares upstream samples | MUST | Paired intervals narrower than with independent samples on a test workflow |
| MEMO-007 | Named tapes (frozen memo sets) for CI | MUST | A tape replays identically a month later |
| OVL-001 | Overlays as JSON Patch per call site, propagated through baggage across service hops | MUST | An overlay applies in a two-service workflow |
| OVL-002 | Steer verification: a trial is invalid if the applied overlay hash differs from the requested one | MUST | A service without the interceptor yields invalid trials, never scores |
| OVL-003 | The ADK plugin applies template overlays while preserving session-state variables | MUST | Edited template still renders `{customer_tier}` per item |
| OVL-004 | Overrides for tool descriptions and model parameters | MUST | Both appear in the captured model request |
| WR-001 | SQL writes become recorded no-ops in FORK and REPLAY; attempted writes are visible to evaluators | MUST | An evaluator can assert the attempted UPDATE |
| WR-002 | LIVE write tests run only against a per-run disposable database or schema | SHOULD | Shared databases are rejected for LIVE write tests |
| WEXP-001 | Trace-aware evaluators: tool-call trajectory, retrieved rows, step-level scores per call site | MUST | A step score attaches to its span |
| WEXP-002 | Replay a single call from captured requests with a patch, without running the workflow | MUST | Result in seconds for 100 captured calls |
| CI-001 | `promptlab ci` runs a REPLAY experiment and posts a pass or fail status to GitHub or GitLab | MUST | A regression blocks a pull request |
| LG-001 | Native `langgraph-agent-server` profile: threads, runs, per-run context carrying the overlay reference | MUST | A stock Agent Server graph runs a dataset |
| LG-002 | LangGraph graphs behind a custom API through `http-generic` | MUST | One custom-API graph onboarded |
| LG-003 | Call-site ids from node names via the OpenInference LangChain instrumentor | MUST | Ids stable across runs |
| LG-004 | Template overlays through `pl.slot` or LangChain middleware | MUST | Template edits keep variables |
| LG-005 | Memo-based fork within graphs, independent of the checkpointer | MUST | Changing a late node replays earlier nodes from cache |

## 6. Phase 3: optimize and scale

**Goal:** propose better prompts automatically, test against production reality, roll out safely and scale storage. **Exit gate:** the optimizer beats the baseline on the test split, and reliability targets are met.

| Id | Requirement | Priority | Acceptance |
| --- | --- | --- | --- |
| OPT-001 | GEPA optimization jobs over selected tunables, using train for reflection and validation for selection, within a budget, returning a Pareto set | MUST | A job improves a validation score over its baseline |
| OPT-002 | Candidates become `proposed` prompt versions with lineage and pass static checks (variables kept, schema kept, maximum length) | MUST | No candidate drops a required variable |
| OPT-003 | Successive halving across candidates | SHOULD | Budget per candidate falls as rounds proceed |
| OPT-004 | Traces are redacted before they reach the reflection model | MUST | Masked values never appear in reflection prompts |
| PROD-001 | Build datasets from sampled production traces, stratified by tag | MUST | 300-item stratified dataset created from a week of traffic |
| PROD-002 | Re-execute captured production runs in FORK mode and compare against the original output | MUST | Counterfactual report per item |
| SCN-001 | Scenarios with versioned simulated users and a persona library; pass^k reporting | SHOULD | A scenario reports pass^3 |
| SCN-002 | Metamorphic groups with relation checks | SHOULD | Paraphrase invariance reported per prompt |
| ROLL-001 | Canary rollout by alias targeting, with an OFREP-compatible resolution endpoint | SHOULD | 5 percent of traffic receives a candidate version |
| ROLL-002 | Online evaluators on a traffic sample with automatic rollback on guard breach | SHOULD | Simulated degradation triggers rollback |
| SCALE-001 | Move span storage to ClickHouse when thresholds are crossed, without API changes | SHOULD | Trace queries under 2 seconds p95 at 10x volume |
| SCALE-002 | MCP server facade for coding agents | COULD | An MCP client starts and polls an experiment |

## 7. Non-functional requirements

| Id | Requirement | Priority | Phase | Acceptance |
| --- | --- | --- | --- | --- |
| NFR-001 | Control API availability 99.9 percent monthly *(proposed)* | MUST | 1 | Measured by synthetic checks |
| NFR-002 | Control API read latency under 300 ms p95; playground first token within provider latency plus 200 ms *(proposed)* | MUST | 0 | Load test report |
| NFR-003 | 1,000 trials dispatched within one minute of experiment start *(proposed)* | SHOULD | 1 | Load test report |
| NFR-004 | No experiment loses completed trials on a crash or deploy | MUST | 0 | Chaos test |
| NFR-005 | Encryption in transit (TLS 1.2 or later) and at rest; provider keys and service credentials only in a vault | MUST | 0 | Security review |
| NFR-006 | Project-level isolation with row-level security in Postgres | MUST | 0 | Cross-project access tests fail as expected |
| NFR-007 | Retention per data class: raw prompt and output payloads 30 days, scores and metadata 1 year, audit log 3 years *(proposed)* | MUST | 1 | Scheduled deletion verified |
| NFR-008 | Datasets, traces and retrieved text are treated as untrusted inside judge and reflection prompts (delimited, no tool access) | MUST | 0 | Prompt-injection test cases pass |
| NFR-009 | Point-in-time recovery for Postgres; recovery tested quarterly | MUST | 0 | Restore drill record |
| NFR-010 | Everything in the UI is also available through the API and CLI | MUST | 0 | API coverage check in CI |
| NFR-011 | Public API versioned by date; breaking changes only in a new version with 6 months’ notice *(proposed)* | MUST | 0 | Changelog and contract tests |
| NFR-012 | PromptLab’s own telemetry (OpenTelemetry), alerts on queue depth, error rate and provider failures | MUST | 0 | Dashboards and alert runbook exist |
| NFR-013 | Cost visibility per experiment, project and team, including judge cost | MUST | 0 | Cost report matches provider invoices within 5 percent |

## 8. UI/UX requirements

The UI exists to answer one question quickly and honestly: is this prompt or model change better, on what, and how sure are we? Every screen is built around comparison and evidence, not single outputs.

### Personas

| Persona | Main jobs | What they need most |
| --- | --- | --- |
| Workflow developer | Edit prompts, onboard services, run experiments, debug traces | Speed, keyboard flow, CLI parity, clear failures |
| Lead | Review evidence, approve or reject promotions, roll back | A one-screen verdict with the evidence attached |
| Domain expert or reviewer | Write test cases, review outputs, label gold sets | Plain language, blind side-by-side review, no YAML |
| Admin | Providers, keys, roles, budgets, retention | Safe defaults and an audit trail |

### Design principles

1. **Compare first.** Outputs appear side by side with differences highlighted; a single output is the exception.
2. **Evidence over point estimates.** Show intervals and the verdict words `better`, `worse`, `inconclusive`; never crown a winner from an average.
3. **Never hide how a number was made.** Mode, replay fidelity, dataset version, judge and sample size sit next to every result.
4. **One path from idea to production.** Playground, test case, dataset, experiment, promotion, approval: each step links to the next.
5. **Same truth everywhere.** Every screen has a shareable URL, and every action has an API and CLI equivalent shown on demand (“copy as CLI”).
6. **Safe by default.** Destructive or protected actions (alias moves, LIVE runs with writes) need confirmation and show their blast radius.

### Core journey (Phase 0)

1. Open a prompt in the library, or import one from ADK code.
2. Edit it in the playground and run it across two or three models side by side.
3. Save interesting inputs as test cases, or upload a dataset.
4. Build an experiment from the prompt (baseline vs candidate, or model upgrade).
5. Read the results: verdict, intervals, regressions first.
6. Request promotion; a lead approves from the approval queue.

### Screen requirements

| Id | Screen | Requirement | Phase | Priority |
| --- | --- | --- | --- | --- |
| UX-001 | Prompt library | List by workflow and team with alias badges (`dev`, `staging`, `prod`), last change, owner; search by name, text and tag | 0 | MUST |
| UX-002 | Prompt editor | Syntax highlighting for `{variables}`, inline validation of undeclared placeholders, output-schema editor, model and parameter panel, change note required on save | 0 | MUST |
| UX-003 | Playground | Variables form generated from the variable schema; run one prompt on up to four models in columns; streaming output; schema-valid badge; tokens, cost and latency per column | 0 | MUST |
| UX-004 | Playground | “Save as test case” and “Open as experiment” from any result | 0 | SHOULD |
| UX-005 | Version history | Timeline of versions with side-by-side diff of text, variables, schema, model and parameters | 0 | MUST |
| UX-006 | Datasets | Upload with column-mapping preview, validation errors per row, split view, item editor, tag filters | 0 | MUST |
| UX-007 | Experiment builder | Step-by-step form (target, dataset and split, variants, evaluators, repetitions, budget) with a live cost estimate and a power estimate; a toggle shows the equivalent YAML | 0 | MUST |
| UX-008 | Experiment run | Live progress by variant, coverage, spend against budget, pause, resume and cancel | 0 | MUST |
| UX-009 | Results summary | One verdict card per variant: delta with 95 percent interval, guard metrics with pass or fail, sample size; an interval chart across variants | 0 | MUST |
| UX-010 | Results detail | Per-item table sorted with regressions first; filter by tag and verdict; side-by-side outputs with diff highlighting; judge rationale; open the item’s run | 0 | MUST |
| UX-011 | Results detail | Slice breakdown by dataset tag, with a warning when slices are too small to trust | 0 | SHOULD |
| UX-012 | Human review | Blind A/B review with keyboard shortcuts, rubric guidance and progress; reviewer agreement shown to leads | 0 | SHOULD |
| UX-013 | Promotions | Promotion request pre-filled with evidence; lead approval queue; approve or reject with comment; history and one-click rollback | 0 | MUST |
| UX-014 | Workflows | Registration wizard: pick profile (`adk-api-server`, `langgraph-agent-server`, `http-generic`), draft adapter from OpenAPI, edit mapping with schema validation, “test with one item” showing request, response and extracted output | 1 | MUST |
| UX-015 | Workflows | `doctor` results as a checklist (observe, steer, memo, capture, read-only database, context propagation) with fix instructions per failure | 1 | MUST |
| UX-016 | Workflows | Call-site map: agents or nodes, their prompts and tunables, linked to the registry | 1 | MUST |
| UX-017 | Trace view | Span tree and timeline; model spans show rendered prompt, template and variables; SQL spans show statement and rows; cost and latency per span | 1 | MUST |
| UX-018 | Captured inputs | Select traces for a call site and turn them into a dataset in one flow, with redaction preview | 1 | MUST |
| UX-019 | Trace diff | Two runs aligned by call site, showing cache hits and misses and where outputs diverge | 2 | MUST |
| UX-020 | Results | Mode, replay fidelity and capture age shown on every in-workflow result; REPLAY results visually distinct from LIVE | 2 | MUST |
| UX-021 | Optimization | Candidate table with lineage tree, Pareto chart of quality against cost, and diff from parent | 3 | MUST |
| UX-022 | Admin | Providers and keys, roles, budgets, retention settings, audit log search | 0 | MUST |

### Cross-cutting UX requirements

| Id | Requirement | Priority |
| --- | --- | --- |
| UX-101 | WCAG 2.2 AA accessibility; full keyboard operation; never rely on color alone for verdicts | MUST |
| UX-102 | Desktop-first layout from 1280 px; read-only result and approval views usable on a phone | SHOULD |
| UX-103 | Light and dark themes | SHOULD |
| UX-104 | Every error says what happened and what to do next (for example, a contract-drift error names the missing response path) | MUST |
| UX-105 | Empty states teach the next step, with a sample prompt and dataset to try | SHOULD |
| UX-106 | Long operations never block the screen; progress persists across reloads and devices | MUST |
| UX-107 | “Copy as CLI” and “copy as API call” on every action that creates or runs something | SHOULD |
