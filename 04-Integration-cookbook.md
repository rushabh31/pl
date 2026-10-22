# Integration cookbook

Every service shape PromptLab supports, a catalog of ADK and LangGraph scenarios, and worked examples from first steps to a streaming multi-agent service. Each shows the exposed API and the adapter that connects it; no service changes its contract. Pictures of the architecture, the user journey, and the refund service’s API flow and dataflow are in Diagrams.

## How to read this

**Capability levels.** What PromptLab can do with a service depends only on how close it can get to the model calls, not on the shape of the API.

| Level | Requires | PromptLab can |
| --- | --- | --- |
| **Full** | The process that calls the model runs under `promptlab run` (Python ADK, LangGraph, LangChain, or plain Google GenAI) | Observe, steer prompts, templates, tool descriptions and parameters, memoize, capture data reads, fork from any step, score each step |
| **Gateway** | The model client’s base URL points at the PromptLab gateway (any language) | Observe model calls, steer static prompts and parameters, memoize model calls; capture data reads only where an OpenTelemetry agent instruments the service |
| **Black box** | Only the API is reachable | Run datasets, score outputs, compare versions or configurations of the service; no prompt overrides |

**The one rule.** An interaction pattern never lowers the level by itself. What matters is whether PromptLab’s run context (the W3C `traceparent` and `baggage` headers carrying run id, overlay and mode) reaches the process that makes the model calls. Synchronous HTTP carries it automatically. Async jobs, webhooks, queues and background workers must forward it, in the job record or the message attributes. `promptlab doctor` checks this with a canary overlay, and each case below says how context travels.

**What every case shares.** A test deployment with a read-only business database, test cases in the standard envelope (main tab 7.8), a versioned adapter file, and a passing `doctor` run. Every adapter has this skeleton; the cases below show only the parts that differ.

```yaml
workflow: <name>
framework: adk | langgraph | langchain | custom | external
invoke:
  profile: http-generic            # or a native profile
  base_url: https://<service>.test.internal
  auth: {secret_ref: vault://promptlab/<service>}
  request:  { ... }                # how a test case becomes the service's request
  response: { ... }                # how the answer and fields are extracted
  limits:   {concurrency: 8, timeout_ms: 60000}
```

## Coverage matrix

Thirty-three cases in four groups. Phase is when the adapter feature arrives (Requirements tab, CMP rows).

| Id | Case | Pattern | Profile | Level | Phase |
| --- | --- | --- | --- | --- | --- |
| A1 | Minimal JSON, one prompt | Synchronous | `http-generic` | Full | 1 |
| A2 | Renamed and nested fields | Synchronous | `http-generic` | Full | 1 |
| A3 | GET with query parameters | Synchronous | `http-generic` | Full | 1 |
| A4 | Context in path and headers | Synchronous | `http-generic` | Full | 1 |
| A5 | File upload | Synchronous, multipart | `http-generic` | Full | 1 |
| A6 | Plain-text response | Synchronous | `http-generic` | Full | 1 |
| A7 | Batch endpoint | Many items per call | `http-generic` | Full | 2 |
| A8 | Errors and business outcomes in the body | Synchronous | `http-generic` | Full | 1 |
| A9 | GraphQL | Synchronous | `http-generic` | Full | 1 |
| B1 | Submit, then poll | Asynchronous | `http-generic` | Full | 1 |
| B2 | Webhook callback | Asynchronous | `http-generic` | Full | 2 |
| B3 | Session first, then message | Multi-step | `http-generic` | Full | 1 |
| B4 | Multi-turn conversation | Multi-step | `http-generic` | Full | 2 |
| B5 | Human approval mid-run | Interrupt and resume | `http-generic` | Full | 2 |
| B6 | JSON Lines streaming | Stream | `http-generic` | Full | 2 |
| B7 | WebSocket | Bidirectional | `websocket` | Full | 3 |
| B8 | Queue-driven | Message bus | `pubsub` or `kafka` | Full | 3 |
| B9 | Batch job writing to a table | Job, no response body | `http-generic` + result query | Full | 2 |
| B10 | gRPC | RPC | `grpc` | Full | 3 |
| C1 | Stock `adk api_server` | Sessions, `/run` | `adk-api-server` | Full | 1 |
| C2 | LangGraph Agent Server | Threads and runs | `langgraph-agent-server` | Full | 2 |
| C3 | LangGraph behind your own API | Synchronous | `http-generic` | Full | 2 |
| C4 | No framework, direct model calls | Synchronous | `http-generic` | Full | 1 |
| C5 | Service in another language | Synchronous | `http-generic` | Gateway | 1 |
| C6 | Chain of services | Multi-hop | `http-generic` | Full on each hop | 2 |
| C7 | Agent with MCP tools | Synchronous | `http-generic` | Full | 2 |
| C8 | Remote A2A agent | Task protocol | `a2a` | Black box, or Full if it honors overlays | 3 |
| C9 | Closed vendor service | Synchronous | `http-generic` | Black box | 1 |
| D1 | Writes with no dry-run option | Synchronous | `http-generic` | Full | 1 |
| D2 | External side effects (email, payments) | Synchronous | `http-generic` | Full | 2 |
| D3 | Service-side response cache | Synchronous | `http-generic` | Full | 1 |
| D4 | End-user authentication | Synchronous | `http-generic` | Full | 1 |
| D5 | Slow or rate-limited service | Any | Any | Full | 1 |

The ADK and LangGraph scenario catalogs (K and G) follow the matrix, then the end-to-end refund service (E) and the getting-started examples (W).

## K. ADK scenarios

Every common way an ADK workflow is built and exposed as a service. All are Full level when the service runs under `promptlab run`; the adapter shape comes from groups A to D. Where code helps, the pattern is sketched in the pattern-code section below.

| Id | Scenario | Exposed as | Tunables PromptLab sees | How PromptLab tests it | Watch out |
| --- | --- | --- | --- | --- | --- |
| K1 | One `LlmAgent` with an instruction template | JSON API (A1) or stock server (C1) | Instruction, model parameters | Standalone in Phase 0; FORK in Phase 2 | Keep per-request values in session state, not in code |
| K2 | Agent with function tools reading Postgres | JSON API | Instruction, tool docstrings | FORK replays the tools’ SQL reads | `ORDER BY` in every tool query |
| K3 | Structured extraction with `output_schema` | JSON API returning the schema | Instruction, schema field descriptions | Field-level exact-match evaluators | Check whether your ADK version allows tools alongside `output_schema`; if not, split extraction from tool use |
| K4 | `SequentialAgent` pipeline with `output_key` handoffs | JSON API | Every stage’s instruction | Per-stage scores; fork from any stage | Downstream templates read state keys, so keep key names stable |
| K5 | `ParallelAgent` fan-out with a merger | JSON API | Branch and merger instructions | Changing one branch keeps the others’ cache hits | Give every branch a distinct `output_key` |
| K6 | `LoopAgent` draft-and-review with an exit tool | JSON API | Writer and critic instructions, exit criteria text | Loop count and cost as metrics; FORK hits until the drafts diverge | Always set `max_iterations` |
| K7 | Coordinator delegating to `sub_agents` | JSON or chat API | Coordinator instruction, sub-agent descriptions | Routing accuracy from the trace (which agent handled the case) | Descriptions drive routing, so they are tunables too |
| K8 | Agent used as a tool (`AgentTool`) | JSON or SSE API | Caller and callee instructions, callee description | Trajectory rule: was the tool agent called exactly when it should be | See example E |
| K9 | RAG over pgvector through a tool | JSON or SSE API | Retriever instruction, top-k, embedding model | Retrieval recall per step; top-k as a numeric tunable | The embedding call is memoized by query text |
| K10 | Built-in Google Search grounding | JSON API | Instruction | The memoized model call replays the grounded answer; LIVE is needed to test fresh search results | Search runs on the provider side, so its results are not captured separately and change over time |
| K11 | Built-in code execution | JSON API | Instruction | Score the executed result fields | Runs on the provider side, like K10 |
| K12 | MCP toolset | JSON API | Instruction; tool descriptions only if the MCP server reads them from PromptLab | MCP calls captured and replayed (C7) | Tool annotations decide write blocking |
| K13 | Long-running tool or human approval | Pause-and-resume API (B5) | Instruction, approval-policy text | Interrupt evaluators with a scripted human answer | Keep the run context with the paused run |
| K14 | Callbacks used as guardrails | Any | Instructions; guardrail messages if they are templates | Block rate and false-block rate evaluators | Guardrail logic in code is not tunable unless its text is |
| K15 | Multi-turn sessions on a database session service | Chat API with a conversation id (B3, B4) | All instructions | Per-turn scoring; a fresh session per test case | The session store must be writable in the test deployment |
| K16 | Long-term memory (memory service, `load_memory`) | Chat API | Instructions | Seed memory per test case as fixtures; capture memory searches | Memory contents are data: freeze them or results drift |
| K17 | Artifacts and file inputs | Multipart API (A5) | Instructions | Files versioned with the dataset | File content hash is part of the memo key |
| K18 | Multimodal input (images, audio) | Multipart or base64 JSON | Instructions | As K17 | Large payloads: watch retention limits |
| K19 | Streaming SSE pipeline | SSE API (example E) | All | Final-event aggregation, time to first token | Requirement CMP-014 |
| K20 | Stock `adk api_server` | ADK’s own server (C1) | All; templates need the plugin | `adk-api-server` profile | Plugin loading depends on the ADK version |
| K21 | Several model providers through ADK’s LiteLLM wrapper | Any | Per-agent model and parameters | Model-upgrade experiments per agent | The LiteLLM client needs its interceptor too |
| K22 | Instruction computed by a function | Any | Rendered instruction only | Template induction | Refactor into a static template with state values (developer guide, section 6) |
| K23 | Deployed on Cloud Run or GKE | Container | Full, through the launcher | As any other case | The image’s entrypoint changes to `promptlab run --` |
| K24 | Deployed on a managed agent runtime (for example Vertex AI Agent Engine) | Managed endpoint | Full only if the PromptLab plugin and initialization run inside the agent code and outbound network allows it; otherwise Gateway or black box | To validate in a spike | You do not control the entrypoint |

## G. LangGraph scenarios

The same for LangGraph, including LangChain agents that run on LangGraph. Call sites come from node, task and agent names through the OpenInference LangChain instrumentor.

| Id | Scenario | Exposed as | Tunables PromptLab sees | How PromptLab tests it | Watch out |
| --- | --- | --- | --- | --- | --- |
| G1 | Single-node prompt graph | Your own API (C3) | Node prompt | Standalone and FORK | — |
| G2 | Linear multi-prompt chain | Your own API or Agent Server | Each node’s prompt | Per-node scores; fork from a late node (W3) | — |
| G3 | Conditional routing: a router node, then branches | Your own API | Router prompt, branch prompts | Routing accuracy; branch-level scores | Route from the router’s structured output, never free text |
| G4 | Prebuilt tool-calling agent (LangChain `create_agent`) | Your own API | System prompt, tool descriptions | Trajectory checks | Middleware for template edits (W2) |
| G5 | Supervisor with worker agents | Agent Server or your own API | Supervisor prompt, worker prompts and descriptions | Per-agent scores, handoff accuracy | Call sites come from agent names; keep them stable |
| G6 | Swarm-style handoffs between agents | As G5 | Handoff tool descriptions, agent prompts | Handoff correctness from the trace | The active agent lives in state |
| G7 | Map-reduce with `Send` | Your own API | Map prompt, reduce prompt | Each branch is memoized separately; a reduce-only change reruns only the reduce | Sort branch outputs before the reduce, so its prompt is deterministic |
| G8 | Reflection loop with a maximum number of iterations | Your own API | Generator and critic prompts | Loop count as a metric; FORK hits until divergence | Cap the iterations |
| G9 | Human-in-the-loop with `interrupt` and resume | Agent Server or your own API with a thread id (B5) | Prompts | Scripted resume values per test case | Needs a checkpointer, so a writable checkpoint store in the test deployment |
| G10 | Checkpointed multi-turn threads (Postgres checkpointer) | Agent Server threads | Prompts | Per-turn scoring; a new thread per case | Checkpoint writes go to a separate writable database |
| G11 | Long-term memory store | Agent Server or your own API | Prompts | Seed the store per case | Freeze store contents |
| G12 | Subgraphs | Any | Prompts in nested nodes | Call sites include the path, for example `node:research/node:summarize` | Stable names at every level |
| G13 | Structured-output nodes | Your own API | Prompts, schema field descriptions | Field exact match | — |
| G14 | RAG with a pgvector retriever | Your own API | Prompts, k, filters | Retrieval recall; SQL captured | Deterministic ordering in the retriever query |
| G15 | Tool node with custom tools | Any | Tool docstrings, prompts | Tool-description experiments | Tool effects classified as read or write |
| G16 | Streaming through the Agent Server | Stream endpoint (SSE) | Prompts | Final-state aggregation | Requirement CMP-014 |
| G17 | Functional API (`entrypoint`, `task`) | Your own API | Prompts inside tasks | Call sites from task names | Name tasks explicitly |
| G18 | Graph exposed over A2A or MCP by the Agent Server | `a2a` or MCP profile | Prompts | Full, since you own the server | — |
| G19 | Background runs with a webhook (Agent Server) | Webhook pattern (B2) | Prompts | Callback correlation by run id | — |
| G20 | Different providers per node (`init_chat_model`) | Any | Per-node model and parameters | Per-node model upgrades | Each provider SDK needs its interceptor |

**Mixed workflows.** An ADK agent can call a LangGraph service through an HTTP tool, and a LangGraph node can call an ADK agent over A2A. Both are chains of services (C6): each hop runs under the launcher, call sites are prefixed with the service name, and one overlay can change prompts in both frameworks at once.

## Pattern code

Short sketches for the shapes that differ most from a plain agent. Each is served by an ordinary JSON endpoint, so its adapter is A1-style; what matters is what PromptLab sees inside. Library signatures follow current documentation; check them against your installed versions.

### K6 in code: ADK review loop

```python
from google.adk.agents import LlmAgent, LoopAgent
from google.adk.tools.tool_context import ToolContext

def approve_draft(tool_context: ToolContext) -> dict:
    """Call only when the draft meets every review criterion. Ends the review loop."""
    tool_context.actions.escalate = True
    return {}

writer = LlmAgent(name="writer", model=MODEL, output_key="draft",
    instruction="Write or revise the policy summary for {topic}. Reviewer feedback so far: {feedback?}")
critic = LlmAgent(name="critic", model=MODEL, output_key="feedback", tools=[approve_draft],
    instruction="Review this draft against the style guide:\n{draft}\n"
                "If it passes every check, call approve_draft. Otherwise list the fixes.")
review_loop = LoopAgent(name="review_loop", sub_agents=[writer, critic], max_iterations=3)
```

PromptLab sees `agent:writer/llm` and `agent:critic/llm`, counts iterations from the trace, and can score quality against cost per extra loop.

### K7 in code: ADK coordinator with specialists

```python
billing = LlmAgent(name="billing_agent", model=MODEL,
    description="Handles invoices, refunds and payment questions.",
    instruction="Resolve billing questions for {customer_tier} customers.")
tech = LlmAgent(name="tech_agent", model=MODEL,
    description="Handles login problems, app errors and outages.",
    instruction="Troubleshoot step by step. Ask for the app version first.")
coordinator = LlmAgent(name="coordinator", model=MODEL,
    instruction="Route the customer to the right specialist. Answer directly only for greetings.",
    sub_agents=[billing, tech])
```

A `routing_match` evaluator reads from the trace which agent produced the final answer and compares it with `expected.route`. The two `description` fields are tunables, because they decide the routing.

### G3 in code: LangGraph router and branches

```python
from typing import Literal
from pydantic import BaseModel

class Route(BaseModel):
    route: Literal["billing", "tech", "other"]

ROUTER = pl.slot("support.router", default="Classify this request: {message}", variables=["message"])
router_llm = llm.with_structured_output(Route)

def route(state: State):
    return {"route": router_llm.invoke(ROUTER.render(message=state["message"])).route}

builder.add_node("route", route)
builder.add_conditional_edges("route", lambda s: s["route"],
                              {"billing": "billing", "tech": "tech", "other": "fallback"})
```

Changing only the billing branch’s prompt replays the router from cache; changing the router reruns everything after it.

### G7 in code: LangGraph map-reduce

```python
import operator
from typing import Annotated, TypedDict
from langgraph.types import Send

class State(TypedDict):
    docs: list[dict]
    summaries: Annotated[list, operator.add]
    report: str

def fan_out(state: State):
    return [Send("summarize_doc", {"doc": d}) for d in sorted(state["docs"], key=lambda d: d["id"])]

def reduce(state: State):
    parts = sorted(state["summaries"], key=lambda s: s["doc_id"])      # deterministic reduce prompt
    return {"report": llm.invoke(REDUCE.render(parts=parts)).content}

builder.add_conditional_edges("load_docs", fan_out, ["summarize_doc"])
```

Each `summarize_doc` branch is memoized by its own request, so a change to the reduce prompt reruns only the reduce.

### G9 in code: LangGraph human approval

```python
from langgraph.types import interrupt, Command

def approve(state: State):
    decision = interrupt({"pending_refund": state["amount"]})     # pauses the run
    return {"approved": decision["approved"]}

graph = builder.compile(checkpointer=checkpointer)                # writable checkpoint store in tests

# POST /v1/refunds            → graph.invoke(inputs, {"configurable": {"thread_id": tid}});
#                               if the result carries an interrupt, respond {"status": "needs_approval", "runId": tid}
# POST /v1/runs/{tid}/decision → graph.invoke(Command(resume={"approved": body.approved}),
#                                             {"configurable": {"thread_id": tid}})
```

The adapter is exactly B5: the test case scripts the approver’s answer, and both paths are scored.

### G5 in code: LangGraph supervisor

```python
from langchain.agents import create_agent
from langgraph_supervisor import create_supervisor

billing_agent = create_agent(model=llm, tools=[lookup_invoice], name="billing_agent",
                             system_prompt="Resolve billing questions using invoice data.")
tech_agent = create_agent(model=llm, tools=[check_status], name="tech_agent",
                          system_prompt="Troubleshoot technical issues step by step.")
supervisor = create_supervisor([billing_agent, tech_agent], model=llm,
                               prompt="Send each request to the right agent; answer only after an agent finishes.").compile()
```

Call sites are the supervisor and each named agent. A handoff evaluator checks the sequence of agents in the trace against the expected route.

## E. End-to-end example: the streaming refund service

One realistic service from code to scored results: chained ADK agents with RAG behind one SSE API (E1), connected to PromptLab (E2), and one trial traced step by step (E3). Its API flow and dataflow are drawn in Diagrams.

### E1. The service: chained ADK agents with RAG behind one SSE API

A refund-resolution service: six LLM agents in a sequential pipeline with a parallel research stage, retrieval over pgvector in Postgres, an agent used as a tool, and one Server-Sent Events endpoint. It follows the Workflow developer guide: instruction templates over session state, stable agent names, tools as documented functions, the clock injected at the API, and the only write performed last by the API layer.

```
POST /v1/refunds/assist                                        (SSE stream)
        │
        ▼
refund_pipeline  (SequentialAgent)
 1. intake_agent ───────────────────────────────────────► state.intake        intent, order id (structured)
 2. gather  (ParallelAgent)
      ├─ policy_retriever ── search_policies: embed + pgvector ──► state.policy_context
      └─ account_researcher ── get_order, get_refund_history (SQL) ► state.account_facts
 3. resolver_agent ── tools: fraud_checker (agent as tool), get_refund_limits ──► state.decision (JSON)
 4. responder_agent ─────────────────────────────────────► state.reply         streamed to the client
        │
        ▼
API layer: persist the decision (idempotent; skipped when dryRun) → final SSE "result" event
```

**Agents**

```python
# refund_service/agents.py
import os
from pydantic import BaseModel
from google.adk.agents import LlmAgent, ParallelAgent, SequentialAgent
from google.adk.tools.agent_tool import AgentTool
from .tools import search_policies, get_order, get_refund_history, get_refund_limits

FAST = os.environ.get("REFUND_FAST_MODEL", "gemini-2.5-flash")
STRONG = os.environ.get("REFUND_STRONG_MODEL", "gemini-2.5-pro")

class Intake(BaseModel):
    intent: str                 # refund_request | refund_status | other
    order_id: str | None
    summary: str

intake_agent = LlmAgent(
    name="intake_agent", model=FAST,
    instruction=("Classify the customer's message and extract the order id if present.\n"
                 "Customer tier: {customer_tier}. Channel: {channel}."),
    output_schema=Intake, output_key="intake")

policy_retriever = LlmAgent(
    name="policy_retriever", model=FAST,
    instruction=("Find the refund policy passages that apply to this request: {intake}\n"
                 "Call search_policies with focused queries. Return each passage with its title."),
    tools=[search_policies], output_key="policy_context")

account_researcher = LlmAgent(
    name="account_researcher", model=FAST,
    instruction=("Collect the facts needed to decide this request: {intake}\n"
                 "Customer id: {customer_id}. Use get_order and get_refund_history. Report facts only."),
    tools=[get_order, get_refund_history], output_key="account_facts")

gather = ParallelAgent(name="gather", sub_agents=[policy_retriever, account_researcher])

fraud_checker = LlmAgent(
    name="fraud_checker", model=FAST,
    description="Assesses fraud risk for a refund from account facts. Returns low, medium or high with reasons.",
    instruction="Assess fraud risk from the facts you are given. Answer with a risk level and two reasons.")

resolver_agent = LlmAgent(
    name="resolver_agent", model=STRONG,
    instruction=("Decide the refund using only these policies:\n{policy_context}\n\n"
                 "Account facts:\n{account_facts}\n\nToday is {today}. Customer tier: {customer_tier}.\n"
                 "If the amount exceeds the tier's limit from get_refund_limits, call fraud_checker first.\n"
                 "Reply with JSON only, with the keys action (approve, deny or escalate), amount, "
                 "policy_titles (the titles you relied on) and reason."),
    tools=[AgentTool(agent=fraud_checker), get_refund_limits], output_key="decision")

responder_agent = LlmAgent(
    name="responder_agent", model=FAST,
    instruction=("Write the reply to the customer in a {reply_style} tone.\n"
                 "Decision: {decision}\nRelevant policies: {policy_context}\n"
                 "Never promise anything the decision does not grant."),
    output_key="reply")

refund_pipeline = SequentialAgent(
    name="refund_pipeline",
    sub_agents=[intake_agent, gather, resolver_agent, responder_agent])
```

The JSON shape is described in words rather than written with braces, because ADK treats `{...}` in an instruction as a session-state placeholder.

**Tools, including RAG over pgvector**

```python
# refund_service/tools.py
import os
from google import genai
from .db import read_pool          # psycopg 3 pool on DATABASE_READ_URL

EMBED_MODEL = os.environ.get("EMBED_MODEL", "gemini-embedding-001")
TOP_K = int(os.environ.get("POLICY_TOP_K", "6"))
_client = genai.Client()

def search_policies(query: str) -> list[dict]:
    """Search the refund and billing policies. Returns the most relevant passages with titles. Read-only."""
    vec = _client.models.embed_content(model=EMBED_MODEL, contents=query).embeddings[0].values
    with read_pool.connection() as conn:
        rows = conn.execute(
            "SELECT title, chunk FROM policy_chunks "
            "ORDER BY embedding <=> %s::vector, id LIMIT %s",      # id breaks ties: stable order
            (str(vec), TOP_K)).fetchall()
    return [{"title": t, "passage": c} for t, c in rows]

def get_order(order_id: str) -> dict:
    """Look up an order: items, amount, purchase date and status. Read-only."""
    ...   # SELECT ... FROM orders WHERE id = %s

def get_refund_history(customer_id: str) -> list[dict]:
    """List the customer's refunds in the last 12 months, newest first. Read-only."""
    ...   # SELECT ... FROM refunds WHERE customer_id = %s ORDER BY created_at DESC, id LIMIT 50

def get_refund_limits(customer_tier: str) -> dict:
    """Return the automatic refund limit for a customer tier. Read-only."""
    ...   # SELECT ... FROM refund_limits WHERE tier = %s
```

**One SSE endpoint for the whole pipeline**

```python
# refund_service/app.py
import json, os, uuid
from datetime import date
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from pydantic import BaseModel
from google.genai import types
from google.adk.runners import Runner
from google.adk.sessions import DatabaseSessionService
from google.adk.agents.run_config import RunConfig, StreamingMode
from promptlab.adk import PromptLabPlugin           # proposed PromptLab plugin
from .agents import refund_pipeline
from .repos import customers_repo, decisions_repo, parse_decision

app = FastAPI(title="Refund assistant")
sessions = DatabaseSessionService(db_url=os.environ["SESSION_DB_URL"])
runner = Runner(agent=refund_pipeline, app_name="refund_service",
                session_service=sessions, plugins=[PromptLabPlugin()])

class AssistRequest(BaseModel):
    customerId: str
    message: str
    channel: str = "web"
    replyStyle: str = "friendly"
    requestTime: str | None = None      # injected clock; defaults to today
    dryRun: bool = False

def sse(event: str, data: dict) -> str:
    return f"event: {event}\ndata: {json.dumps(data)}\n\n"

@app.post("/v1/refunds/assist")
async def assist(req: AssistRequest):
    request_id = f"req-{uuid.uuid4().hex[:8]}"
    customer = customers_repo.get(req.customerId)
    session = await sessions.create_session(
        app_name="refund_service", user_id=req.customerId,
        state={"customer_id": req.customerId, "customer_tier": customer.tier,
               "channel": req.channel, "reply_style": req.replyStyle,
               "today": (req.requestTime or date.today().isoformat())[:10]})

    async def stream():
        yield sse("started", {"requestId": request_id})
        current = None
        async for event in runner.run_async(
                user_id=req.customerId, session_id=session.id,
                new_message=types.Content(role="user", parts=[types.Part(text=req.message)]),
                run_config=RunConfig(streaming_mode=StreamingMode.SSE)):
            if event.author != current:
                current = event.author
                yield sse("stage", {"agent": current})
            for call in event.get_function_calls():
                yield sse("tool", {"agent": event.author, "name": call.name})
            if event.author == "responder_agent" and event.partial and event.content:
                text = "".join(p.text or "" for p in event.content.parts)
                if text:
                    yield sse("delta", {"text": text})
        final = await sessions.get_session(app_name="refund_service",
                                           user_id=req.customerId, session_id=session.id)
        decision = parse_decision(final.state["decision"])      # validates the JSON, raises on bad shape
        writes = [{"table": "refund_decisions", "op": "insert", "key": request_id}]
        if not req.dryRun:
            decisions_repo.save(request_id, req.customerId, decision)   # the only write, idempotent
        yield sse("result", {"requestId": request_id, "decision": decision,
                             "reply": final.state["reply"], "intendedWrites": writes,
                             "dryRun": req.dryRun})

    return StreamingResponse(stream(), media_type="text/event-stream")
```

What a client receives (abridged):

```
event: started
data: {"requestId": "req-3f9a12c0"}

event: stage
data: {"agent": "intake_agent"}

event: tool
data: {"agent": "policy_retriever", "name": "search_policies"}

event: tool
data: {"agent": "resolver_agent", "name": "fraud_checker"}

event: delta
data: {"text": "Good news: your refund for order 8812 "}

event: result
data: {"requestId": "req-3f9a12c0", "decision": {"action": "approve", "amount": 129.0,
       "policy_titles": ["Refunds within 60 days"], "reason": "…"}, "reply": "…",
       "intendedWrites": [{"table": "refund_decisions", "op": "insert", "key": "req-3f9a12c0"}],
       "dryRun": true}
```

### E2. Connecting the refund service to PromptLab

The service needs no PromptLab-specific code beyond the plugin line already shown. PromptLab drives it through its own SSE contract, sees all six agents as separate call sites, and can test any one of them inside the full pipeline while the rest replay from cache.

**1. Run it as a test deployment**

```dockerfile
ENTRYPOINT ["promptlab", "run", "--"]
CMD ["uvicorn", "refund_service.app:app", "--host", "0.0.0.0", "--port", "8080"]
```

| Setting | Test deployment value | Why |
| --- | --- | --- |
| `DATABASE_READ_URL` | Read replica or staging copy, read-only role | Real policies, orders and refunds; no production writes possible |
| `SESSION_DB_URL` | The test deployment’s own disposable database | ADK sessions are the service’s working state and must be writable; they are not business data |
| Decision writes | `dryRun: true` from the adapter (Phase 1); write blocking in the I/O plane (Phase 2) | The `refund_decisions` insert never reaches production |
| `PROMPTLAB_ENV` | `test` | Lets `doctor` and the adapter tell the environments apart |

**2. What `promptlab doctor` finds**

```
call sites   agent:intake_agent/llm                      instruction · output schema · params
             agent:policy_retriever/llm                  instruction · tools[search_policies]
             agent:policy_retriever/tool:search_policies/embed   embedding model
             agent:account_researcher/llm                instruction · tools[get_order, get_refund_history]
             agent:resolver_agent/llm                    instruction · tools[fraud_checker, get_refund_limits]
             agent:fraud_checker/llm                     instruction · description (used as tool description)
             agent:responder_agent/llm                   instruction · params
captured     sql: policy_chunks (vector search) · orders · refunds · refund_limits
capabilities observe ✓  steer.model ✓  steer.template ✓ (plugin)  memo ✓  capture.io ✓ (sql)
safety       read-only business database ✓   writable session store ✓   dryRun honored ✓
context      canary overlay reached all 6 agents, including the parallel stage ✓
```

To make the retrieval depth tunable as well, read it through PromptLab instead of the environment: `TOP_K = pl.param("refunds.policy_top_k", default=6)`. It then appears as a numeric tunable on `search_policies`.

**3. The adapter, with SSE aggregation**

```yaml
# workflows/refund-service.yaml
workflow: refund-service
framework: adk
invoke:
  profile: http-generic
  base_url: https://refund-service.test.internal
  auth: {secret_ref: vault://promptlab/refund-service}
  request:
    method: POST
    path: /v1/refunds/assist
    body:
      customerId: "{{ item.input.customer_id }}"
      message: "{{ item.input.message }}"
      channel: "{{ item.input.channel }}"
      replyStyle: "{{ item.state.reply_style }}"
      requestTime: "{{ item.clock.now }}"
      dryRun: true
  response:
    stream: sse
    final_event: result                       # the event that carries the answer
    output: "$.reply"
    fields:
      action: "$.decision.action"
      amount: "$.decision.amount"
      policy_titles: "$.decision.policy_titles"
      intended_writes: "$.intendedWrites"
    stream_metrics:                           # recorded on every trial
      time_to_first: delta
      count: [stage, tool]
  limits: {concurrency: 8, timeout_ms: 60000}
```

The adapter reads the stream until the `result` event, extracts the reply and decision fields from it, and records time to first `delta` as a metric. A stream that ends without `result` is a scored failure, with the events received so far kept for debugging. Because your services stream, SSE aggregation is needed in Phase 1 (requirement CMP-014).

**4. A test case**

```json
{
  "id": "refund-0031",
  "input":   {"customer_id": "C-1029", "message": "I returned order 8812 three weeks ago and still have no refund.", "channel": "web"},
  "state":   {"reply_style": "friendly"},
  "clock":   {"now": "2026-10-14T09:30:00Z"},
  "expected": {"intent": "refund_request", "action": "approve", "policy_titles": ["Refunds within 60 days"]},
  "tags": ["refund_request", "premium", "web"],
  "fixtures": [{"table": "customers", "key": "C-1029"}, {"table": "orders", "key": "8812"}]
}
```

**5. What to test, and what runs live in FORK mode**

| Goal | Change | Runs live | Replayed from cache |
| --- | --- | --- | --- |
| Tighter decision rules | `resolver_agent` instruction | resolver, fraud checker when called, responder | intake, both research agents, embeddings and SQL |
| Better retrieval queries | `policy_retriever` instruction | policy retriever and its new searches (embedding + SQL on the replica), resolver, responder | intake, account researcher |
| Retrieve more passages | `refunds.policy_top_k` 6 → 10 | the vector query with the new limit, then resolver and responder | intake, embeddings (same query text), account researcher |
| Clearer delegation to the fraud check | `fraud_checker` description | resolver onward | everything before the resolver |
| Cheaper reply model | `responder_agent` model | responder only | everything else |
| Release check of all approved changes | all of the above | everything, in LIVE mode on the replica with `dryRun` | nothing |

The parallel stage needs no special handling: each research agent’s calls are keyed by their own requests, so changing one branch leaves the other’s cache hits intact.

**6. Evaluators, including per-stage scores**

| Evaluator | Type | What it checks |
| --- | --- | --- |
| `action_match` | Deterministic | `action` equals `expected.action` |
| `intent_match` | Deterministic, step-level | intake’s structured `intent` equals `expected.intent` |
| `retrieval_recall` | Trace-aware, step-level | expected policy titles appear among passages returned by `search_policies` |
| `citation_grounded` | Trace-aware | every title in `policy_titles` was actually retrieved in this run |
| `fraud_trajectory_rule` | Trace-aware | `fraud_checker` was called whenever the amount exceeded the limit returned by `get_refund_limits` |
| `single_write_check` | Deterministic | `intended_writes` holds exactly one insert |
| `faithfulness_judge@v2` | LLM judge | the reply matches the decision and the retrieved passages, promising nothing more |
| `tone_judge@v1` | LLM judge | the reply matches `reply_style` |
| Stream metrics | Guard | time to first delta, total latency and cost do not regress beyond their margins |

Step-level scores show which stage fails. For example, if `action_match` drops while `retrieval_recall` holds, the regression is in the resolver, not in retrieval.

**7. An experiment**

```yaml
# experiments/resolver-rules-v3.yaml
experiment: resolver-rules-v3
workflow: refund-service
dataset: refund-requests@v4
split: validation
mode: FORK
repetitions: 3
variants:
  baseline: []
  rules_v3:
    - call_site: agent:resolver_agent/llm
      patch: [{op: replace, path: /system, value: {template_ref: "prompt://refunds.resolver.system@v3"}}]
evaluators: [action_match, citation_grounded, fraud_trajectory_rule, single_write_check, faithfulness_judge@v2]
stats:
  primary: action_match
  guards: {time_to_first_delta_ms: "+15%", cost_usd: "+10%", faithfulness_judge: "-0.02"}
```

The new resolver template must keep the variables `{policy_context}`, `{account_facts}`, `{today}` and `{customer_tier}`; the registry refuses a version that drops one, and the plugin fills them from session state on every trial.

**8. How this example lands across the phases**

| Phase | What you can do with this service |
| --- | --- |
| 0 | `promptlab import adk` copies all six instructions into the registry; each can be tested standalone with hand-written variables, and the responder can be compared across models |
| 1 | The adapter runs the full pipeline over a dataset with `dryRun`, scoring decisions and replies end to end; real requests are captured, so standalone tests of the resolver use the exact `{policy_context}` and `{account_facts}` the pipeline produced |
| 2 | FORK experiments from the table above: one agent changed, the rest replayed, with per-stage scores and common random numbers |
| 3 | GEPA proposes resolver and retriever instructions together, using per-stage feedback; production traffic becomes the test set |

### E3. One trial, step by step

The API flow of a single trial: variant `rules_v3`, test case `refund-0031`, repetition 0, in FORK mode.

1. **Schedule.** The experiment engine hands the trial to the runner gateway, interleaved with the baseline trial for the same case and repetition.
2. **Pre-run check.** The fixtures (customer C-1029, order 8812) are confirmed on the replica through a read-only query.
3. **Request.** The adapter renders the request body from the test case, adds `traceparent` and `baggage` (run id, overlay reference, mode FORK), and posts to `/v1/refunds/assist` with `dryRun: true`.
4. **Session.** The service’s web-framework instrumentation extracts the context; the API creates the ADK session (in the writable test session store) with tier, channel, style and date, and starts the pipeline.
5. **Intake.** The model plane finds no overlay entry for `intake_agent`; the request is identical to the baseline’s, so the memo cache answers (hit).
6. **Research, in parallel.** The policy retriever’s model call, its embedding call and its vector query all hit; the account researcher’s model call and SQL reads hit too.
7. **Resolver.** The overlay patches the instruction with template v3; the plugin fills `{policy_context}`, `{account_facts}`, `{today}` and `{customer_tier}`. The request differs, so it misses and goes live to Gemini. Its call to `fraud_checker` is a new request and runs live; `get_refund_limits` hits.
8. **Responder.** Its input (the decision) changed, so it misses and runs live, streaming `delta` events to the adapter.
9. **Result.** With `dryRun`, the API skips the write, lists the intended insert, and sends the final `result` event.
10. **Collect.** The adapter extracts reply and decision fields, records time to first delta, and confirms that the applied overlay hash matches the requested one; otherwise the trial is invalid.
11. **Score.** Spans arrive over OTLP; evaluators score the output and the steps (using the retrieved passages from the trace); the statistics engine pairs the score with the baseline trial for the same case and repetition.

**Where the data goes**

| Data | Produced by | Stored in | Leaves the customer’s environment? |
| --- | --- | --- | --- |
| Test cases | Dataset module | Postgres and object storage (control plane) | Already in the control plane |
| Prompt versions and overlays | Registry | Postgres (control plane); cached by the SDK | Read by the service |
| Business reads (policies, orders, refunds) | Read replica | Memo and capture store, encrypted per tenant | Configurable; metadata only if required |
| Model requests and responses | Gemini | Memo and capture store | Configurable, redacted |
| ADK session state | The service | Test session database | No |
| Spans | The service, over OTLP | Postgres, later ClickHouse | Yes, after in-process redaction |
| Scores and results | Evaluators, statistics engine | Postgres | Yes |
| Intended writes | Service response | Trial result | Yes; the production write never happens |

## W. Getting-started examples

All frameworks connect to PromptLab in the same four steps, and only the first step differs by framework. The workflow code stays as it is: its real tools, databases and vector stores keep running.

| Step | What you do | Effort |
| --- | --- | --- |
| 1. Connect | Start the app under `promptlab run`, or add the one-line shim when you need template edits | Zero code, or one line |
| 2. Verify | `promptlab doctor` checks observe, steer, memo and capture, and lists the call sites it found | One command |
| 3. Define | Write an experiment file: dataset, variants as patches on call sites, evaluators, mode | One YAML file |
| 4. Run and read | `promptlab experiment run` returns paired deltas, intervals and guard checks | One command |

*Framework code uses each framework’s documented API. Everything named `promptlab` (`pl`) is the proposed PromptLab interface, and model names are placeholders.*

### W1. Google ADK: a support agent with a tool

The agent answers billing questions, looks orders up through an internal HTTP API, and personalizes its instruction from session state.

```python
# support_agent/agent.py
from google.adk.agents import LlmAgent

def lookup_order(order_id: str) -> dict:
    """Look up an order's status and refund eligibility."""
    return orders_api.get(f"/orders/{order_id}").json()   # real HTTP call

root_agent = LlmAgent(
    name="support",
    model="gemini-2.5-flash",
    instruction="You are a billing assistant for {customer_tier} customers. "
                "Always confirm the order id before discussing refunds.",
    tools=[lookup_order],
)
```

**Connect with zero code.** Serve the agent with ADK’s own API server, wrapped by the launcher:

```bash
promptlab run -- adk api_server            # interceptors load around the google-genai client
promptlab init --profile adk-api-server --app support_agent
promptlab doctor
```

`doctor` reports what it found, for example:

```
call sites   agent:support/llm        system · tools[lookup_order] · params
capabilities observe ✓  steer.model ✓  memo ✓  capture.io ✓ (http)  steer.template ✗
note         system prompt varies across runs (customer_tier); add the ADK plugin for template edits
```

**Add the plugin for template edits.** `{customer_tier}` is filled from session state before the model sees the instruction, so the model plane only sees the rendered text. The ADK plugin, registered on the Runner, supplies the template and its variables, so an edited template keeps `{customer_tier}` intact.

```python
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from promptlab.adk import PromptLabPlugin          # proposed: wraps before_model_callback

runner = Runner(agent=root_agent, app_name="support_agent",
                session_service=InMemorySessionService(),
                plugins=[PromptLabPlugin()])
```

The `adk-api-server` profile starts each trial by creating a session whose initial state carries the test case’s fields (here `customer_tier`) and the overlay reference, then calls `/run`. The `lookup_order` HTTP call is captured on the first run and replayed in FORK and REPLAY runs. ADK plugins are not supported in the ADK web UI, so run through a Runner or `adk api_server`.

### W2. LangChain v1: a RAG agent

The agent searches a hosted vector store for policy passages and answers with citations. LangChain v1 agents run on LangGraph.

```python
# kb_agent.py
import promptlab as pl
from langchain.agents import create_agent
from langchain.tools import tool

@tool
def search_policies(query: str) -> str:
    """Search the refund and billing policy knowledge base."""
    hits = vector_store.similarity_search(query, k=4)       # hosted store, called over HTTP
    return "\n\n".join(h.page_content for h in hits)

agent = create_agent(
    model="openai:gpt-4.1-mini",
    tools=[search_policies],
    system_prompt="Answer billing questions using only retrieved policy text. Cite the policy name.",
    middleware=[pl.langchain.middleware()],    # optional: template edits; uses wrap_model_call
)

result = agent.invoke({"messages": [{"role": "user", "content": "Can I get a refund after 45 days?"}]})
```

**Connect.** Run it as an in-process workflow, or serve it through the LangGraph Agent Server:

```bash
promptlab run -- python serve_kb_agent.py   # or: promptlab init --profile in-process --entry kb_agent:agent
promptlab doctor
```

Call sites come from the OpenInference LangChain instrumentor: `agent:kb_agent/llm` (system prompt, `tools[search_policies]`, parameters). Because the system prompt here is static, the model plane can replace it without the middleware; the middleware matters once the prompt contains variables. The vector-store query travels over HTTP, so the I/O plane captures the retrieved passages. With an in-process index such as a local FAISS file, mark the search with `@pl.boundary('policy_index', effect='read')` so retrieval is still captured.

### W3. LangGraph: a multi-prompt pipeline over Postgres

Three steps, two prompts: retrieve account history from Postgres, draft a reply, then critique and revise it.

```python
# reply_graph.py
from typing import TypedDict
import promptlab as pl
from langgraph.graph import StateGraph, START, END
from langchain.chat_models import init_chat_model

llm = init_chat_model("openai:gpt-4.1-mini")

DRAFT = pl.slot("reply.draft",            # template with variables: declare it once
    default="Write a reply to the customer.\nHistory:\n{history}\nMessage:\n{message}",
    variables=["history", "message"])
CRITIQUE = "Check the draft for policy errors and tone. Return the corrected reply only."

class State(TypedDict):
    message: str
    history: str
    draft: str
    reply: str

def retrieve(state: State):
    rows = db.execute("SELECT note FROM account_notes WHERE customer_id = %s", (cid(state),))
    return {"history": "\n".join(r.note for r in rows)}          # real SQL, captured

def draft(state: State):
    msg = llm.invoke(DRAFT.render(history=state["history"], message=state["message"]))
    return {"draft": msg.content}

def critique(state: State):
    msg = llm.invoke([("system", CRITIQUE), ("user", state["draft"])])
    return {"reply": msg.content}

builder = StateGraph(State)
builder.add_node("retrieve", retrieve)
builder.add_node("draft", draft)
builder.add_node("critique", critique)
builder.add_edge(START, "retrieve")
builder.add_edge("retrieve", "draft")
builder.add_edge("draft", "critique")
builder.add_edge("critique", END)
graph = builder.compile()
```

**Connect.** Serve the graph with the LangGraph Agent Server (`langgraph.json` points at `reply_graph.py:graph`) and register the profile:

```bash
promptlab run -- langgraph dev
promptlab init --profile langgraph-agent-server --url http://localhost:2024 --assistant reply_graph
promptlab doctor
```

`doctor` finds two call sites, `node:draft` (template `reply.draft`) and `node:critique` (static system prompt), and one captured SQL read in `node:retrieve`. Changing the critique prompt in FORK mode replays the SQL read and the draft call from the cache and runs only the critique live, which is fork-from-step without touching LangGraph’s checkpointer.

### W4. One experiment file for all three

Only `workflow` and the call-site paths change between frameworks. This one tests the LangGraph pipeline’s two prompts together and separately.

```yaml
# experiments/reply-v2.yaml
experiment: reply-v2
workflow: reply_graph
dataset: support-messages@v3
split: validation
mode: FORK                        # replay unchanged calls, run changed ones live
repetitions: 3
variants:
  baseline: []
  draft_only:
    - call_site: node:draft
      patch: [{op: replace, path: /messages/0/template, value: {template_ref: "prompt://reply.draft@v8"}}]
  critique_only:
    - call_site: node:critique
      patch: [{op: replace, path: /system, value: {template_ref: "prompt://reply.critique@v3"}}]
  both: [draft_only, critique_only]   # combine overlays: a 2x2 factorial
evaluators: [policy_correctness_judge@v4, tone_judge@v2, schema_valid]
stats:
  primary: policy_correctness_judge
  guards: {cost_usd: "+10%", latency_p95_ms: "+15%"}
```

For the ADK agent the same file would use `workflow: support_agent` and `call_site: agent:support/llm`. For the LangChain agent it would use `agent:kb_agent/llm`, with a second entry patching `/tools/search_policies/description`.

```bash
promptlab experiment run experiments/reply-v2.yaml
```

The same thing from a notebook:

```python
import promptlab as pl
result = pl.experiment.run("experiments/reply-v2.yaml")
result.compare("baseline", "both").show()
```

### W5. Reading the result

An illustrative report (the numbers are made up to show the format):

```
reply-v2 · reply_graph@a41f9c · support-messages@v3 (validation, 240 items × 3 reps) · mode FORK
replay fidelity 0.97 · cache hits 61% · cost $4.10 (baseline run alone: $3.80)

variant         policy_correctness Δ vs baseline [95% CI]   guards            verdict
draft_only      +0.021 [-0.004, +0.046]                     ok                inconclusive
critique_only   +0.038 [+0.015, +0.061]                     ok                better
both            +0.044 [+0.020, +0.068]                     cost +6% ok       better
interaction     draft × critique: +0.006 [-0.020, +0.031]   no evidence of interaction

next: confirm critique_only on the test split in LIVE mode before promotion
```

The report shows that the critique change carries the gain, while the draft change has not shown an effect at this sample size. The next step is a single confirmation on the held-out test split.

### W6. Other everyday tasks

| Task | Command |
| --- | --- |
| Test one prompt alone on captured inputs, no workflow run | `promptlab call replay --call-site node:critique --tape tape_88d0 --patch critique-v3.json` |
| Let GEPA propose better prompts | `promptlab optimize --workflow reply_graph --tunables node:draft,node:critique --algorithm gepa --budget 400` |
| Turn last week’s production traffic into a test set | `promptlab dataset from-traces --workflow reply_graph --since 7d --stratify intent --sample 300` |
| Block a regression in a pull request | `promptlab ci --experiment experiments/reply-v2.yaml --mode REPLAY` |
| Promote a winner | `promptlab promote reply.critique@v3 --to staging` (moves the alias after the test-split gate passes) |

## A. Request and response shapes

All nine are synchronous HTTP, so run context travels in the request headers with no work from the service.

### A1. Minimal JSON, one prompt

An ADK summarizer: one agent, one endpoint, JSON in and out.

```python
@app.post("/v1/summarize")
async def summarize(req: SummarizeRequest) -> SummarizeResponse:      # {"text": str, "maxBullets": int}
    session = await sessions.create_session(app_name="summarizer", user_id="api",
                                            state={"max_bullets": req.maxBullets})
    summary = await run_to_text(runner, session, req.text)            # runner.run_async, keep the final text
    return SummarizeResponse(summary=summary)                        # {"summary": str}
```

```yaml
request:
  method: POST
  path: /v1/summarize
  body: {text: "{{ item.input.text }}", maxBullets: "{{ item.input.max_bullets }}"}
response:
  output: "$.summary"
```

The smallest possible integration. If the agent is served by the stock `adk api_server`, use C1 instead and skip the adapter mapping.

### A2. Renamed and nested fields

A triage service with its own conventions: camelCase, a JSON:API-style envelope, arrays.

```json
POST /v1/tickets:triage
{"data": {"type": "ticket", "attributes": {"subjectLine": "…", "bodyText": "…", "labels": ["web", "eu"]}}}
→ 200
{"data": {"attributes": {"triage": {"queue": "billing", "priority": "P2", "rationale": "…"}}},
 "meta": {"requestId": "r-81"}}
```

```yaml
request:
  method: POST
  path: /v1/tickets:triage
  body:
    data:
      type: ticket
      attributes:
        subjectLine: "{{ item.input.subject }}"
        bodyText: "{{ item.input.body }}"
        labels: "{{ item.input.labels }}"         # a whole-value placeholder keeps its JSON type (array)
response:
  output: "$.data.attributes.triage.rationale"
  fields:
    queue: "$.data.attributes.triage.queue"
    priority: "$.data.attributes.triage.priority"
    request_id: "$.meta.requestId"
```

A placeholder that is the entire value keeps the item’s type (array, number, boolean, object); one embedded in a string renders as text. Evaluators score `queue` and `priority` exactly and judge `rationale`.

### A3. GET with query parameters

```
GET /v1/sentiment?text=Where%20is%20my%20refund&lang=en   →   {"label": "negative", "score": 0.91}
```

```yaml
request:
  method: GET
  path: /v1/sentiment
  query: {text: "{{ item.input.text }}", lang: "{{ item.input.lang }}"}
response:
  output: "$.label"
  fields: {score: "$.score"}
```

Query values are URL-encoded automatically. PromptLab warns when a rendered URL exceeds 8 KB *(proposed)*, since long texts in a GET can be cut by proxies.

### A4. Context in path and headers

Tenant and ticket in the path, region and language in headers.

```yaml
request:
  method: POST
  path: /v1/tenants/{{ item.context.tenant }}/tickets/{{ item.input.ticket_id }}/triage
  headers:
    X-Tenant-Region: "{{ item.context.headers.X-Tenant-Region }}"
    Accept-Language: "{{ item.context.locale }}"
  body: {note: "{{ item.input.note }}"}
response:
  output: "$.reply"
```

Test tenants and tickets must exist in the test database, so they belong in the case’s `fixtures`. Results can be sliced by locale or region through tags.

### A5. File upload

An invoice extractor: a PDF and a currency field in, structured fields out. The ADK agent receives the file as an inline data part.

```yaml
request:
  method: POST
  path: /v1/invoices/extract
  multipart:
    file: {from: "{{ item.context.files[0] }}", filename: invoice.pdf, content_type: application/pdf}
    currency: "{{ item.input.currency }}"
response:
  output: "$"                              # the whole JSON object is the output
  fields: {total: "$.total", vendor: "$.vendor", due_date: "$.dueDate"}
```

Files live in dataset storage, versioned with the dataset. The model plane sees the file part, and its memo key includes the file’s content hash, so a changed file is never served from cache. Evaluators compare `total`, `vendor` and `due_date` exactly.

### A6. Plain-text response

```
POST /v1/draft-email   {"customerName": "…", "issue": "…"}   →   200 text/plain   "Dear Ms Weber, …"
```

```yaml
request:
  method: POST
  path: /v1/draft-email
  body: {customerName: "{{ item.input.name }}", issue: "{{ item.input.issue }}"}
response:
  format: text
  output: "$body"                          # the raw body is the output
```

### A7. Batch endpoint

One request classifies many texts.

```
POST /v1/classify:batch   {"items": [{"id": "1", "text": "…"}, …]}   →   {"results": [{"id": "1", "label": "billing"}, …]}
```

```yaml
batch:
  size: 25                                 # test cases per request
  item_template: {id: "{{ item.id }}", text: "{{ item.input.text }}"}
  request: {method: POST, path: /v1/classify:batch, body: {items: "{{ batch_items }}"}}
  response: {items: "$.results", match_by: "$.id", output: "$.label"}
```

Each test case is still scored on its own, and a failed batch fails each case in it. If the service puts several items into one prompt, neighbours can influence answers, so PromptLab keeps the same batch composition for every variant to keep the comparison paired.

### A8. Errors and business outcomes in the body

The service returns `200` with `{"status": "error", "error": {"code": "CONTEXT_TOO_LONG"}}` on failure, and `422` when it correctly refuses an unsupported request.

```yaml
response:
  accept_status: [200, 422]                # 422 is a valid outcome to test, not a failure
  error_when: "$.status == 'error'"
  error_message: "$.error.code"
  output: "$.answer"
  fields: {outcome: "$.status"}
```

A real failure is scored as a failure with its code; an expected refusal is scored against `expected.outcome`. Without this, a `200` error body would be scored as an empty answer.

### A9. GraphQL

```yaml
request:
  method: POST
  path: /graphql
  body:
    query: |
      query Answer($q: String!, $customerId: ID!) {
        answer(question: $q, customerId: $customerId) { text citations { title } }
      }
    variables: {q: "{{ item.input.question }}", customerId: "{{ item.input.customer_id }}"}
response:
  error_when: "$.errors"                   # GraphQL reports errors with status 200
  output: "$.data.answer.text"
  fields: {citations: "$.data.answer.citations[*].title"}
```

## B. Interaction patterns

Each case ends with **Context travels**: how the run context reaches the process that calls the model. Helper names under `pl.context` are proposed PromptLab SDK functions.

### B1. Submit, then poll

A research-report pipeline that takes minutes. The API accepts the job, a worker runs the ADK agents, and a status endpoint returns the report.

```
POST /v1/reports {"accountId": "A-77", "topic": "churn risk"}   →   202 {"jobId": "j-41"}
GET  /v1/reports/j-41   →   {"status": "running"}   …   {"status": "done", "report": {"summary": "…", "sections": […]}}
```

```yaml
request:
  method: POST
  path: /v1/reports
  body: {accountId: "{{ item.input.account_id }}", topic: "{{ item.input.topic }}"}
async:
  job_id: "$.jobId"
  poll: {path: "/v1/reports/{{ job_id }}", every_ms: 2000, until: "$.status in ['done', 'failed']", timeout_ms: 600000}
  failed_when: "$.status == 'failed'"
response:
  output: "$.report.summary"
  fields: {sections: "$.report.sections[*].title"}
```

**Context travels** on the job record: the API stores the incoming `traceparent` and `baggage` with the job, and the worker restores them before running the agents. OpenTelemetry instrumentation for common task queues (such as Celery) does this automatically; for a custom queue, call `pl.context.inject(job.headers)` when enqueuing and `pl.context.attach(job.headers)` in the worker.

### B2. Webhook callback

The service accepts a document, analyses it, and later posts the result to a callback URL.

```yaml
request:
  method: POST
  path: /v1/analyses
  body:
    document: "{{ item.input.document_text }}"
    callbackUrl: "{{ callback.url }}"            # a PromptLab endpoint reachable from the test network
    correlationId: "{{ run.id }}"
async:
  callback:
    correlate_by: "$.correlationId"
    timeout_ms: 900000
    verify: {hmac_header: X-Signature, secret_ref: vault://promptlab/analyses-webhook}
response:
  output: "$.result.summary"
  fields: {risk: "$.result.risk"}
```

Callbacks are matched by correlation id and rejected when the signature fails. **Context travels** as in B1, on the job record.

### B3. Session first, then message

A chat service that opens a conversation before accepting messages.

```yaml
steps:
  - name: open
    request: {method: POST, path: /v1/conversations, body: {customerId: "{{ item.input.customer_id }}"}}
    save: {conversation_id: "$.conversationId"}
  - name: ask
    request:
      method: POST
      path: /v1/conversations/{{ steps.open.conversation_id }}/messages
      body: {text: "{{ item.input.message }}"}
response:
  from_step: ask
  output: "$.reply"
```

**Context travels** in the headers of every step’s request.

### B4. Multi-turn conversation

The same chat service, tested over a whole conversation. The test case holds the user turns.

```json
"turns": [
  {"user": "I want to return my order."},
  {"user": "It's order 8812.", "expected": {"mentions": ["8812"]}},
  {"user": "Can I get the money back on my card?", "expected": {"action": "refund_to_card"}}
]
```

```yaml
steps:
  - name: open
    request: {method: POST, path: /v1/conversations, body: {customerId: "{{ item.input.customer_id }}"}}
    save: {conversation_id: "$.conversationId"}
  - name: turn
    for_each: "{{ item.turns }}"
    request:
      method: POST
      path: /v1/conversations/{{ steps.open.conversation_id }}/messages
      body: {text: "{{ turn.user }}"}
response:
  from_step: turn
  output: "$.reply"
  per_turn: true                           # score each turn and the conversation as a whole
```

In FORK mode, earlier turns replay from cache until the changed prompt first affects a reply, so late-turn changes are cheap to test. Fully dynamic conversations with a simulated user arrive with scenarios in Phase 3. **Context travels** in each turn’s request headers.

### B5. Human approval mid-run

A refund agent pauses for approval above a limit and resumes on a decision.

```
POST /v1/refunds/assist   →   {"status": "needs_approval", "runId": "r-9", "pendingAction": {"type": "refund", "amount": 420}}
POST /v1/runs/r-9/decision {"approved": true}   →   {"status": "completed", "reply": "…"}
```

```yaml
interrupts:
  - when: "$.status == 'needs_approval'"
    record: {pending_action: "$.pendingAction"}           # scored: did it pause when it should?
    respond:
      request:
        method: POST
        path: /v1/runs/{{ response.runId }}/decision
        body: {approved: "{{ item.state.approver_decision | default(true) }}"}
response:
  output: "$.reply"
  fields: {final_status: "$.status"}
```

The test case scripts the human’s answer, so both the approve and the reject paths can be tested. An evaluator checks that the pause happened exactly when expected (for example, only above the tier limit). **Context travels** in both requests; the service must keep it with the paused run, as in B1.

### B6. JSON Lines streaming

```
POST /v1/answer?stream=true   →   application/x-ndjson
{"type": "delta", "text": "Your refund "}
{"type": "delta", "text": "was approved."}
{"type": "final", "answer": "Your refund was approved.", "sources": ["Refunds within 60 days"]}
```

```yaml
response:
  stream: jsonl
  final_when: "$.type == 'final'"
  output: "$.answer"
  fields: {sources: "$.sources"}
  stream_metrics: {time_to_first: "$.type == 'delta'"}
```

For a stream with no final event, `aggregate: {concat: "$.text"}` joins the chunks instead. **Context travels** in the request headers.

### B7. WebSocket

A chat assistant over a WebSocket: the client sends a message, the server streams deltas and a final frame.

```yaml
invoke:
  profile: websocket
  url: wss://assistant.test.internal/ws
  connect_headers: {Authorization: "Bearer {{ auth.token }}"}
  send:
    - {type: user_message, text: "{{ item.input.message }}"}
  receive:
    final_when: "$.type == 'done'"
    output: "$.reply"
```

**Context travels** in the handshake headers; the service must attach it to the connection so that messages handled later on the same socket keep it. `doctor` checks this with a canary overlay.

### B8. Queue-driven

The service has no request API: it consumes `refund-requests` from Pub/Sub and publishes to `refund-decisions`.

```yaml
invoke:
  profile: pubsub
  project: payments-test
  publish:
    topic: refund-requests-test
    data: {customerId: "{{ item.input.customer_id }}", message: "{{ item.input.message }}"}
    attributes: {correlation_id: "{{ run.id }}"}       # traceparent and baggage are added automatically
  result:
    subscription: refund-decisions-promptlab
    correlate_by: "attributes.correlation_id"
    timeout_ms: 120000
  response:
    output: "$.data.reply"
    fields: {action: "$.data.decision.action"}
```

Only topics registered as test topics are accepted. A Kafka profile works the same way with record headers. **Context travels** in message attributes; OpenTelemetry messaging instrumentation restores it in supported consumers, otherwise call `pl.context.attach(message.attributes)`.

### B9. Batch job writing to a table

A nightly job summarizes accounts and writes rows to `daily_summaries`. Nothing comes back in a response body.

```yaml
request:
  method: POST
  path: /v1/jobs/daily-summaries
  body: {date: "{{ item.clock.now[:10] }}", accountIds: ["{{ item.input.account_id }}"]}
async:
  job_id: "$.jobId"
  poll: {path: "/v1/jobs/{{ job_id }}", every_ms: 5000, until: "$.state == 'finished'"}
result:
  from_sql:
    connection: vault://promptlab/summaries-test-db-readonly
    query: >
      SELECT summary, risk_flags FROM daily_summaries
      WHERE job_id = %(job_id)s AND account_id = %(account_id)s
    params: {job_id: "{{ job_id }}", account_id: "{{ item.input.account_id }}"}
  output: "$.rows[0].summary"
  fields: {risk_flags: "$.rows[0].risk_flags"}
```

This job writes by design, so it runs against its own disposable test database, never the production replica; PromptLab reads the results through a read-only role on that database. One account per job keeps test cases independent. **Context travels** on the job record, as in B1.

### B10. gRPC

```yaml
invoke:
  profile: grpc
  target: refund-grpc.test.internal:443
  descriptors: file://protos/refunds_v1.desc       # or server reflection
  method: refunds.v1.RefundAssistant/Decide
  request: {customer_id: "{{ item.input.customer_id }}", message: "{{ item.input.message }}"}
  metadata: {x-tenant: "{{ item.context.tenant }}"}
  response:
    output: "$.reply"
    fields: {action: "$.decision.action"}
```

Server-streaming methods use `final_when` as in B6. **Context travels** as gRPC metadata, extracted by OpenTelemetry gRPC server instrumentation.

## C. Frameworks and runtimes

### C1. Stock `adk api_server`

No custom API at all: the agent is served by ADK’s own server.

```yaml
invoke:
  profile: adk-api-server
  base_url: https://support-agent.test.internal
  app: support_agent
  session:
    user_id: "{{ item.input.customer_id }}"
    state: "{{ item.state }}"             # becomes the session's initial state
  message: "{{ item.input.message }}"
  endpoint: run                           # or run_sse
  response: {output: final_text}          # text of the final response event
```

Start it as `promptlab run -- adk api_server`. Model-plane steering works at once. Template edits need the PromptLab plugin on the Runner; if your ADK version’s server cannot load plugins, either serve the agent through your own Runner (A1) or rely on template induction over the rendered instruction.

### C2. LangGraph Agent Server

```yaml
invoke:
  profile: langgraph-agent-server
  base_url: https://reply-graph.test.internal
  assistant: reply_graph
  thread: per_case                        # a new thread for every test case
  input: {message: "{{ item.input.message }}", customer_id: "{{ item.input.customer_id }}"}
  overlay_via: context                    # overlay reference passed in the run's context
  response: {output: "$.reply"}           # read from the final graph state
```

Call sites come from node names (`node:draft`, `node:critique`) through the OpenInference LangChain instrumentor.

### C3. LangGraph behind your own API

```python
@app.post("/v1/replies")
async def reply(req: ReplyRequest):
    state = await graph.ainvoke({"message": req.message, "customer_id": req.customerId})
    return {"reply": state["reply"], "draft": state["draft"]}
```

```yaml
request: {method: POST, path: /v1/replies, body: {message: "{{ item.input.message }}", customerId: "{{ item.input.customer_id }}"}}
response: {output: "$.reply", fields: {draft: "$.draft"}}
```

Returning intermediate state (here `draft`) is optional; it lets evaluators score a step even when tracing is off.

### C4. No framework, direct model calls

A plain FastAPI service calling Gemini through the Google GenAI client.

```python
import promptlab as pl
from google import genai
from google.genai import types

client = genai.Client()
CLASSIFY = pl.slot("triage.classify",
    default="Classify this ticket as billing, technical or other. Reply as JSON with a label key.\nTicket: {ticket}",
    variables=["ticket"])

@app.post("/v1/classify")
def classify(req: ClassifyRequest):
    resp = client.models.generate_content(
        model=os.environ["TRIAGE_MODEL"],
        contents=CLASSIFY.render(ticket=req.text),
        config=types.GenerateContentConfig(temperature=0, response_mime_type="application/json"))
    return json.loads(resp.text)
```

Without framework spans, the call site is named by the slot (`triage.classify`); without a slot, by a fingerprint of the prompt’s static text. Everything else (steering, memo, capture) is identical to an ADK service.

### C5. Service in another language

A Java, Go or Node service calling a model through its own SDK or plain HTTP.

- **Steer and observe through the gateway:** set the model client’s endpoint or base URL to the PromptLab gateway (most model SDKs and HTTP clients allow a custom endpoint). Add one header, `X-PromptLab-Call-Site: triage.classify`, for stable call-site names; otherwise the gateway names call sites by fingerprint.
- **Traces:** OpenTelemetry’s language agents (for example the Java agent) trace inbound HTTP, outbound HTTP and JDBC with no code change, and propagate `traceparent` and `baggage`.
- **Limits:** static prompts and parameters can be changed; templates with per-request values are best effort through template induction; data reads are traced but not captured for replay, so FORK replays model calls only.
- **The adapter is plain `http-generic`;** nothing about the contract changes. ADK also ships in Java and Go, so native shims can follow if teams adopt them.

### C6. Chain of services

A front-door API (Node) calls an ADK intake service, which calls a LangGraph drafting service, all over HTTP.

```
client → api-gateway (Node) → intake-service (ADK) → drafting-service (LangGraph)
```

The adapter targets the front door only. Each Python hop runs under the launcher, and the Node hop forwards the headers through OpenTelemetry’s Node auto-instrumentation. Call-site ids are prefixed with the service name, so one overlay can change prompts in two services at once:

```yaml
variants:
  both_changed:
    - call_site: svc:intake-service/agent:intake_agent/llm
      patch: [{op: replace, path: /system, value: {template_ref: "prompt://intake.system@v6"}}]
    - call_site: svc:drafting-service/node:draft
      patch: [{op: replace, path: /messages/0/template, value: {template_ref: "prompt://drafting.draft@v4"}}]
```

`doctor` reports, hop by hop, whether a canary overlay arrived; a hop that drops the headers is named.

### C7. Agent with MCP tools

An ADK agent whose tools come from an internal MCP server (for example an orders service).

```python
from google.adk.agents import LlmAgent
from google.adk.tools.mcp_tool.mcp_toolset import MCPToolset

orders_agent = LlmAgent(
    name="orders_agent", model=MODEL,
    instruction="Answer order questions for customer {customer_id} using the orders tools.",
    tools=[MCPToolset(connection_params=ORDERS_MCP)])      # internal orders MCP server
```

The I/O plane captures MCP tool calls at the client (OpenInference ships MCP instrumentation), so they replay like any data read. Write blocking uses the tools’ MCP annotations: tools not marked read-only, or marked destructive, are treated as writes. Tool descriptions served by the MCP server are tunables only if that server reads them from PromptLab; otherwise they are observed but fixed.

### C8. Remote A2A agent

A partner or platform agent reachable only through the A2A protocol.

```yaml
invoke:
  profile: a2a
  agent_card: https://partner-agent.example.com/.well-known/agent-card.json
  message: {role: user, parts: [{text: "{{ item.input.message }}"}]}
  response: {output: "$.artifacts[0].parts[0].text"}
```

Black box by default: datasets, scoring and version comparison work, prompt overrides do not. If the remote agent runs PromptLab and accepts the overlay reference in message metadata, it becomes Full.

### C9. Closed vendor service

A SaaS assistant you cannot instrument. PromptLab still benchmarks it, and if the vendor API exposes settings (a persona, a custom instruction field, a model choice), variants can patch the request itself:

```yaml
variants:
  baseline: []
  concise_persona:
    - request_patch: [{op: replace, path: /body/settings/persona, value: concise}]
  custom_instruction:
    - request_patch: [{op: add, path: /body/settings/instructions, value: {template_ref: "prompt://vendor.instructions@v2"}}]
```

Request-level variants give black-box services real experiments over whatever the vendor lets you configure, with the same statistics and promotion flow.

## D. Safety and environment

### D1. Writes with no dry-run option

A service that writes to Postgres on every request and has no `dryRun` flag. Against a read-only role, its write fails and the request may return an error, which hides the answer you wanted to score.

| Option | Phase | How it works | Trade-off |
| --- | --- | --- | --- |
| Disposable writable copy | 1 | The test deployment points at a database restored from a snapshot for each experiment | Real writes, fully isolated; costs a restore per experiment |
| Write blocking in the I/O plane | 2 | Write statements become recorded no-ops, so the service carries on and responds normally | No service change; the attempted write becomes a scored signal |
| Add a dry-run flag | Any | The team adds `dryRun` and returns intended writes (Workflow developer guide, section 4) | A small code change; the clearest results |

The production primary is never an option: `doctor` blocks experiments when the business database role can write.

### D2. External side effects

A service that sends email, opens tickets or calls a payments API. Outbound calls are classified like database writes, and per-host rules decide what happens outside LIVE mode:

```yaml
effects:
  write: deny
  http:
    - match: {host: api.sendgrid.com, method: POST}
      action: mock                                  # the service gets a plausible success
      response: {status: 202, body: {}}
      record_as: email_sent                         # visible to evaluators
    - match: {host: tickets.internal, method: POST}
      action: mock
      response: {status: 201, body: {id: "T-TEST"}}
      record_as: ticket_opened
    - match: {host: payments.internal}
      action: block                                 # fail loudly; never mock money
```

Evaluators can then assert, for example, that exactly one email was attempted and that it went to the customer on file. LIVE tests use each provider’s sandbox endpoint instead.

### D3. Service-side response cache

A service that caches answers (in Redis, or a semantic cache keyed by similar questions) returns the baseline answer to every variant, so all variants look identical. This is a silent failure, and PromptLab catches it: no model call happens, so steer verification fails and the trials are marked invalid instead of scored.

The fix is to disable the cache in the test deployment, or to honor a bypass header that the adapter sends:

```yaml
request:
  headers: {Cache-Control: no-cache, X-Bypass-Response-Cache: "true"}
```

### D4. End-user authentication

A service that requires an end-user token and behaves differently by role or tenant (an agent-assist tool where tier-2 agents may approve larger refunds).

```yaml
auth:
  identity_from: "{{ item.context.identity }}"       # for example test-agent-tier2
  exchange:
    type: oauth_token_exchange
    token_url: https://idp.test.internal/oauth/token
    secret_ref: vault://promptlab/test-idp
```

Test identities exist only in the test identity provider, each with fixed claims. Datasets name identities, never tokens, and results can be sliced by role to show, for example, that a prompt change helps tier-1 agents but hurts tier-2.

### D5. Slow or rate-limited service

```yaml
limits:
  concurrency: 4
  rate_per_minute: 60
  timeout_ms: 180000
  retry: {on_status: [429, 503], max: 3, backoff: exponential, honor_retry_after: true}
```

Retries cover transient failures only; a trial that still times out is a scored failure, never silently dropped. Because trials are interleaved across variants, a slow afternoon affects every variant equally, and latency guards compare variants within the same experiment.
