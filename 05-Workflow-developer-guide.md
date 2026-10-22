# Building AI workflows for PromptLab

How to write ADK and LangGraph workflows so PromptLab can test, compare and optimize their prompts with no special code. Each rule says why it matters, so teams keep their own style everywhere else.

## 1. The rules at a glance

1. **Keep prompts as templates with named variables.** Never assemble instructions by concatenating strings in code.
2. **Give every agent, node and prompt a stable, unique name.** PromptLab’s call-site ids and history are built from them.
3. **Call models through supported clients, with the model id in configuration.** A model upgrade then becomes a config change that can be tested.
4. **Ask for structured output** wherever a later step or an evaluator reads the result.
5. **Read Postgres through standard drivers, with connection strings from configuration,** and keep reads and writes in separate code paths.
6. **Make writes idempotent and put them in one place,** ideally the last step of the workflow.
7. **Return structured API responses:** the answer, the decision fields and a request id, at stable paths.
8. **Publish an OpenAPI document** for every service.
9. **Start every service through the PromptLab launcher.** It adds tracing and testability, and it fails open.
10. **Inject time and per-request data;** never read the clock or globals while building a prompt.

Everything else (folder layout, frameworks for the API, coding style) is each team’s choice.

## 2. Prompts as data

**Why it matters:** PromptLab can only test and swap a prompt it can see as a template. A prompt built by string concatenation is invisible as a unit: it cannot be versioned, diffed, edited with its variables kept, or optimized.

**Rules**

- **One prompt per model call site.** If two branches need different instructions, use two prompts, or one prompt with a variable, not code that splices sentences together.
- **Stable ids** in the form `<workflow>.<agent or node>.<purpose>`, for example `refunds.assistant.system`.
- **Instructions static, data in variables.** Retrieved rows, user details and dates go in variables; the instruction text stays fixed.
- **Typed, documented variables.** Each variable has a type and one line saying what fills it.
- **Few-shot examples as their own variable** (a list), so they can be tuned separately.
- **No secrets or personal data written into templates.**
- **Prompt text in files or the registry,** not scattered through code: `prompts/*.yaml` beside the code works until runtime resolution is adopted.

**Don’t**

```python
instruction = ("You are helping " + user.name + ". Today is " + str(datetime.now()) +
               ". " + load_rules() + (" Be brief." if channel == "sms" else ""))
```

**Do** (ADK template, values supplied through session state)

```python
instruction = ("You are helping {user_name}. Today is {today}. "
               "Follow these refund rules:\n{refund_rules}\nReply style: {reply_style}.")
```

## 3. Model calls

**Why it matters:** PromptLab steers and records prompts at the model client. Calls that bypass supported clients are not seen, and model ids buried in code make upgrade tests a code change.

- **Use supported clients.** ADK calls Gemini through the Google GenAI client already. In LangGraph, create models with `init_chat_model` or the provider chat-model packages, which sit on the provider SDKs. A hand-written HTTP call to a model endpoint bypasses the interceptor; if one is unavoidable, send it through the PromptLab gateway URL.
- **Model ids and parameters in configuration,** for example `SUPPORT_MODEL=gemini-2.5-flash`, with temperature and output limits set explicitly.
- **Structured output with a schema** wherever the result is parsed: `output_schema` on an ADK `LlmAgent`, or `with_structured_output` on a LangChain chat model. Evaluators then score fields, not prose.
- **Let model errors surface.** A swallowed error becomes a fake answer that scores as a success.
- **Keep calls on the request’s context.** When work moves to threads or tasks, copy the context (`contextvars.copy_context()` in Python) so overlays and traces follow.

## 4. Data access and PostgreSQL

**Why it matters:** your workflows read context from Postgres and write results to production. PromptLab must capture the reads (so tests use real context) and must never let a test reach the writes.

- **Use standard, instrumented drivers:** psycopg 3, asyncpg or SQLAlchemy. Uncommon drivers or raw sockets are not captured.
- **Connection strings from configuration,** with separate read and write settings (`DATABASE_READ_URL`, `DATABASE_WRITE_URL`). A test deployment then points at a replica or staging copy with a read-only role, with no code change.
- **Separate read and write paths.** Put writes behind a small repository layer with explicit write methods, so they are easy to find, classify and block.
- **Writes idempotent, with a key** derived from the request, so a retried or replayed run cannot double-apply.
- **Writes last.** Compute the decision first and persist it in a final step. Tests can then score the decision even when the write is blocked.
- **Support a dry-run option** in the service: skip writes and return the intended writes in the response. Tests in Phase 1 can then score “would it have refunded the right order” before write blocking exists.
- **Handle a refused write cleanly.** In a test deployment the read-only role makes writes fail; return the decision with a clear error field instead of crashing.
- **Deterministic queries.** Add `ORDER BY` to every query whose rows reach a prompt, and an explicit row limit. Unordered rows change the prompt from run to run, which causes flaky results and cache misses.
- **Retrieval settings in configuration.** For pgvector or full-text retrieval, keep k, thresholds and filters in config so they can be tuned like prompts.

## 5. Service API contracts

Any request and response shape works: PromptLab adapts to your contract with a mapping file. These conventions make that mapping short and the results scoreable.

| Convention | Why | Example |
| --- | --- | --- |
| JSON in, JSON out over HTTP | The adapter maps fields by path | `POST /api/v2/refunds/assist` |
| Publish an OpenAPI document | PromptLab drafts the adapter from it | FastAPI serves `/openapi.json` automatically |
| The primary answer at a stable path | Response mapping stays valid across releases | `result.reply` |
| Decision fields returned separately from prose | Evaluators score fields exactly instead of judging text | `result.decision`, `result.orderId` |
| A request id in every response | Links results to traces and logs | `meta.requestId` |
| Errors as non-2xx with a problem-details body | Failures are scored as failures, not as empty answers | `422` with `{"type", "title", "detail"}` |
| Long work returns a job id and a status endpoint | Supported by the adapter’s poll pattern | `202` then `GET /jobs/{id}` |
| Optional dry-run flag | Intended writes can be scored without writing | `"dryRun": true` returns `result.intendedWrites` |
| A `/version` endpoint with the build’s git commit | Results are tied to the exact code tested | `{"commit": "a41f9c"}` |
| Accept and keep `traceparent` and `baggage` headers | Prompt overrides reach your code; gateways must not strip them | Check API gateway header rules |

A response that follows these conventions:

```json
{
  "result": {
    "reply": "Your refund for order 8812 was approved and will arrive in 5 business days.",
    "decision": "refund_approved",
    "orderId": "8812",
    "intendedWrites": [{"table": "refunds", "op": "insert", "key": "8812-req-77"}]
  },
  "meta": {"requestId": "req-77", "model": "gemini-2.5-flash", "commit": "a41f9c"}
}
```

## 6. ADK specifics

- **Unique, stable agent names.** Call-site ids are `agent:<name>/llm`. Renaming an agent splits its history, so treat a rename as a migration and record it.
- **Instructions as string templates with `{state_variables}`.** ADK fills them from session state. Avoid computing the instruction in a Python function or by concatenation: the plugin then sees only rendered text and cannot edit the template safely. If a value must be computed, compute it into session state and keep the template static.
- **Per-request data in session state at session creation.** This is exactly what the `adk-api-server` profile maps dataset fields into, so test items and real requests travel the same path.
- **Tools as plain functions with typed parameters and precise docstrings.** The docstring becomes the tool description the model sees, which PromptLab can test and tune. Name read tools and write tools so their effect is obvious.
- **Descriptions for sub-agents.** In multi-agent apps, each sub-agent’s `description` guides delegation and is itself a tunable.
- **Structured results** through `output_schema`, and `output_key` to store an agent’s result in state for later agents.
- **Register `PromptLabPlugin` on the Runner** in every environment. Run tests through a Runner or `adk api_server`; ADK plugins are not supported in the ADK web UI.
- **Model id from configuration.**

**Do**

```python
import os
from google.adk.agents import LlmAgent

MODEL = os.environ.get("REFUND_MODEL", "gemini-2.5-flash")

def lookup_order(order_id: str) -> dict:
    """Look up an order by id. Returns status and refund eligibility. Read-only."""
    return orders_repo.get(order_id)

def issue_refund(order_id: str, amount: float, idempotency_key: str) -> dict:
    """Issue a refund for an order. Writes to the payments database."""
    return payments_repo.refund(order_id, amount, idempotency_key)

root_agent = LlmAgent(
    name="refund_assistant",
    model=MODEL,
    description="Handles refund questions for existing orders.",
    instruction=("You help {customer_tier} customers with refunds. Today is {today}. "
                 "Confirm the order with lookup_order before calling issue_refund."),
    tools=[lookup_order, issue_refund],
    output_key="reply",
)
```

**Don’t**

```python
def build_instruction(ctx):
    rules = load_rules_from_db()                    # hidden data read inside prompt building
    return f"You help {ctx.state['tier']} customers. Today is {datetime.now()}. {rules}"

agent1 = LlmAgent(name="agent1",                     # generic name, unstable history
                  model="gemini-2.5-flash",          # model hard-coded
                  instruction=build_instruction,     # opaque prompt: no template to test
                  tools=[do_stuff])                  # vague tool, read and write mixed
```

The fix for the “don’t” version: load the rules and the date into session state when the session is created, and keep the instruction a static template with `{refund_rules}` and `{today}`.

## 7. LangGraph specifics

- **Unique, stable node names.** Call-site ids are `node:<name>`.
- **One model call per node where practical, with one declared prompt.** Declare templates with `pl.slot` or keep them in prompt files; LangChain agents can use the PromptLab middleware instead.
- **Models from `init_chat_model` or provider chat-model packages,** with the model id from configuration.
- **Typed state** (TypedDict or Pydantic). Keep retrieved context in its own state key, so traces and evaluators can see what each prompt was given.
- **Deterministic routing.** Conditional edges decide from structured outputs or state, never from randomness or free text parsing.
- **Writes in a dedicated final node** (for example `persist`). FORK runs then block one clearly marked step, and evaluators can read the intended write from state.
- **Request fields mapped into initial state explicitly** when the graph sits behind your own API, so the adapter’s fields and real requests follow one path.
- **Checkpointer optional.** PromptLab forks through its memo cache and does not need one; keep it if your workflow needs human-in-the-loop resumes.

**Do**

```python
import os
from typing import TypedDict
import promptlab as pl
from langgraph.graph import StateGraph, START, END
from langchain.chat_models import init_chat_model

llm = init_chat_model(os.environ.get("REPLY_MODEL", "openai:gpt-4.1-mini"))

DRAFT = pl.slot("replies.draft.system",
    default="Write a reply to the customer.\nAccount history:\n{history}\nMessage:\n{message}",
    variables=["history", "message"])

class State(TypedDict):
    message: str
    customer_id: str
    history: str
    reply: str

def retrieve(state: State):
    rows = notes_repo.recent(state["customer_id"], limit=20)   # ORDER BY created_at DESC inside
    return {"history": "\n".join(r.note for r in rows)}

def draft(state: State):
    return {"reply": llm.invoke(DRAFT.render(history=state["history"],
                                             message=state["message"])).content}

def persist(state: State):
    replies_repo.save(state["customer_id"], state["reply"])    # the only write, last
    return {}

builder = StateGraph(State)
for name, fn in [("retrieve", retrieve), ("draft", draft), ("persist", persist)]:
    builder.add_node(name, fn)
builder.add_edge(START, "retrieve")
builder.add_edge("retrieve", "draft")
builder.add_edge("draft", "persist")
builder.add_edge("persist", END)
graph = builder.compile()
```

**Don’t:** one node that queries the database, builds the prompt with f-strings, calls the model and writes the result. It cannot be forked, its prompt cannot be edited as a template, and its write cannot be isolated.

## 8. Determinism, observability and deployment

**Determinism.** Identical inputs should produce identical prompts. That keeps tests stable and lets the memo cache reuse unchanged steps.

- **Inject the clock:** pass “now” in the request or state instead of calling the clock while building a prompt.
- **Stable ordering:** `ORDER BY` in queries, and sorted keys when serializing lists or dicts into a prompt.
- **No random ids in prompts.** If one is unavoidable, add it to the volatile masks in `promptlab.yaml`.
- **Retries through library settings,** never a loop that silently changes the prompt and tries again.

**Observability**

- Logs include the trace id, so a log line and its PromptLab trace link up.
- Background work (task queues, thread pools) carries the request context; prefer instrumented task libraries or copy the context explicitly.
- Redaction rules for personal fields are declared in `promptlab.yaml` and applied before anything leaves the service.

**Deployment**

```dockerfile
# Same image in every environment; the launcher fails open if PromptLab is unreachable
ENTRYPOINT ["promptlab", "run", "--"]
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8080"]
```

- Every service has a **test deployment**: same image, `PROMPTLAB_ENV=test`, database settings pointing at a replica or staging copy through a read-only role.
- API gateways in front of services pass `traceparent` and `baggage` headers through unchanged.
- `/version` reports the deployed git commit, and `/health` reports readiness.

## 9. Readiness checklist

Complete before onboarding a workflow; `promptlab doctor` verifies the items marked (doctor).

**Prompts and models**

- [ ] Every model call uses a template with named variables; no concatenated instructions
- [ ] Agent and node names are unique and stable
- [ ] Prompt ids follow `<workflow>.<agent or node>.<purpose>`
- [ ] Model ids and parameters come from configuration
- [ ] Structured output schemas exist wherever results are parsed

**Data and writes**

- [ ] Postgres is accessed through psycopg 3, asyncpg or SQLAlchemy
- [ ] Read and write connection strings come from configuration
- [ ] Writes sit in one repository layer, run last, and are idempotent
- [ ] Queries feeding prompts have `ORDER BY` and a row limit
- [ ] A dry-run option returns intended writes (recommended)

**API contract**

- [ ] OpenAPI document published
- [ ] Answer, decision fields and request id at stable response paths
- [ ] Errors return non-2xx status codes with a problem-details body
- [ ] `/version` and `/health` endpoints exist

**Runtime**

- [ ] Service starts through `promptlab run` in every environment
- [ ] ADK: `PromptLabPlugin` registered on the Runner
- [ ] A test deployment exists with a read-only database role (doctor)
- [ ] A canary prompt override survives the full request path, including background work (doctor)
- [ ] Volatile fields are masked and repeated runs produce identical prompts (doctor)
- [ ] Redaction rules for personal data are configured
