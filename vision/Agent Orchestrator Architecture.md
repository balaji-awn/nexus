# Nexus: Agent Orchestrator Architecture

Nexus is a standalone service that takes a user prompt, dynamically selects from thousands of specialized agents based on intent, executes them, and synthesizes the results.

## Tech choices

- **Language:** Python (async, FastAPI)
- **Orchestration loop:** LangGraph
- **Registry and retrieval:** hand-built on Postgres + pgvector
- **Embeddings:** a small embedding model run inside Nexus (e.g. `bge-small-en-v1.5` via sentence-transformers or ONNX). It only produces vectors, never text, so it does not conflict with "no direct LLM calls"
- **Cache:** Redis
- **Observability:** OpenTelemetry and/or LangSmith
- **Agents and LLM access:** only through the AI Platform Service (see below)

Principle: use a framework for the plumbing (state, retries, checkpoints) and own the routing logic (registry, retrieval, re-ranking, evaluation).

## AI Platform Service

The **AI Platform Service** is a REST API running in Kubernetes. It hosts every agent Nexus uses. Nexus cannot make raw LLM calls; it can only invoke agents that already exist on the platform.

- **Agents are pre-created on the platform.** Each one defines its own system prompt, input schema, output schema and model. The model is fixed when the agent is created, so Nexus never chooses or passes model details.
- **Invoking an agent is async.** Nexus submits with the agent name in the URL path, the tenant ID in a header, and a JSON body that matches the agent's input schema. The platform returns an **execution ID**, and Nexus polls it with exponential backoff until it finishes. The final output matches the agent's output schema. The assumed contract is in [AI Platform Service API.md](AI%20Platform%20Service%20API.md).
- **Catalog API:** returns every agent with its full details (description, schemas, model and so on). This is the source of truth for what agents exist. The catalog is the same for all tenants.
- **Tenant:** every incoming Nexus request carries a tenant ID, and Nexus passes it through on every agent call it makes for that request. The tenant decides whose data an agent works on, not which agents exist.
- **No usage reporting:** the platform does not return token usage or cost per call. Nexus tracks only what it can observe (call count, latency, failures).
- **No agent-management API:** agents cannot be created or updated through an API; they are created manually on the platform.

### Two kinds of agents

1. **Domain agents:** the thousands of specialized agents Nexus routes to (IP reputation, user activity search, ...). Nexus selects and runs these.
2. **Nexus system agents:** agents we create on the platform to perform Nexus's own LLM steps, since Nexus cannot call an LLM directly:

| System agent | Used in | Input (sketch) | Output (sketch) |
| --- | --- | --- | --- |
| `nexus-intent` | Intent analysis | user prompt, conversation context | list of sub-tasks, or a clarifying question |
| `nexus-scorer` | Retrieval stage 2 | sub-task + ~50 candidate agent summaries | relevance score / rank per candidate |
| `nexus-planner` | Planning | sub-tasks + top candidates with input schemas | plan: steps with agent name, input, dependencies |
| `nexus-reflector` | Reflection | original prompt, plan, results so far | done / new information needs |
| `nexus-synthesizer` | Synthesis | original prompt + all agent results | final answer with per-agent citations |
| `nexus-example-generator` | Offline only (registry + evaluation) | one domain agent's name, description, schemas | realistic user prompts that agent should handle |

System agents are tagged as such in the registry and are always excluded from retrieval, so Nexus never routes a user's sub-task to one of them.

Because system agents are created by hand on the platform, the repo keeps a spec for each one (system prompt, input schema, output schema) as the reference copy. At startup, Nexus reads the catalog and checks that every system agent exists and that its schemas match the spec; if not, it refuses to start and reports the mismatch. This catches a prompt or schema that was changed on the platform without updating Nexus, or vice versa.

Because input and output schemas are fixed per agent, the system agents' schemas must handle variable-size data (lists of candidates, lists of results), with each domain agent's output carried as a JSON value or string inside the system agent's input.

### Platform client

Nexus has one internal client module that every platform call goes through, both system agents and domain agents. It owns the tenant header, submit-then-poll, timeouts, retries, schema validation, latency tracking and tracing spans. LangGraph is used for the graph only; nodes call this client rather than any LangChain model integration.

- **One shared poller** per Nexus process tracks all open executions, rather than each call running its own polling loop.
- **Execution IDs go into the LangGraph checkpoint** as soon as they're returned. After a restart, Nexus resumes polling instead of resubmitting, so no agent call is repeated.
- **Submits are not retried blindly.** A submit that timed out may already have created an execution.

### Latency

Every system-agent step is a full submit-and-poll round trip, and a simple request already has five (intent, scorer, planner, reflector, synthesizer). So Nexus keeps system-agent calls per request low:

- one `nexus-scorer` call covers all sub-tasks, not one call per sub-task
- the reflection round is skipped when the plan finished with no failures and no new information needs flagged by the planner
- a short initial poll interval (0.5 s) so fast agents aren't held back by the backoff

## Request flow

1. API Gateway → 2. Intent Analysis → 3/4. Registry + Two-Stage Retrieval → 5. Planner/Orchestrator → 6. Agent Execution → 7. Synthesis → response

## 1. API Gateway

- FastAPI service receiving prompts from a **backend service** (the only caller)
- Service-to-service authentication, rate limiting
- Tenant ID taken from the incoming request and carried through the whole request

### Nexus API (async, mirrors the platform)

A request runs several platform round trips, so it takes seconds to minutes. Nexus therefore uses the same submit-and-poll pattern as the platform, with an optional callback.

| Call | Purpose |
| --- | --- |
| `POST /v1/requests` | Submit a prompt. Header `X-Tenant-Id`; body `{ "prompt", "actor_id"?, "callback_url"?, "limits"? }`. Returns `202 { "request_id", "status": "queued" }`. |
| `GET /v1/requests/{request_id}` | Status and, when finished, the result. |
| `POST /v1/requests/{request_id}/input` | Answer a clarifying question: `{ "answer" }`. |
| `DELETE /v1/requests/{request_id}` | Cancel. |

Request statuses: `queued`, `running`, `needs_input` (a clarifying question is pending), and the final states `succeeded`, `partial` (answered, but some steps failed), `failed` and `cancelled`.

A finished request returns the answer, citations (agent name + execution ID per piece of information) and a list of what's missing, if anything. If `callback_url` was given, Nexus POSTs to it on every change to `needs_input` or a final state, so the caller doesn't have to poll.

### Identity (v1)

The caller of Nexus is trusted, and identity is just **tenant ID + an optional actor ID**:

- **Tenant ID** (required): whose data the agents act on.
- **Actor ID** (optional, opaque string): who asked. This could be a local user of that tenant or an MSP operator working on behalf of one of its tenants. An MSP is handled naturally, because it sends a different tenant ID per request.

In v1 the actor ID is used only for audit logs, traces and feedback, not for deciding which agents are allowed. Every tenant can use every agent. Per-user or per-role permissions can be added later as a Stage 1 filter without changing the rest of the design.

## 2. Intent Analysis

- Calls the `nexus-intent` system agent to classify what the user wants
- Decomposes compound prompts into sub-tasks
- Example: "Investigate this IP and check if any users touched it" → (a) IP reputation lookup, (b) user activity search

## 3. Agent Registry

A searchable index of every domain agent, kept in Postgres. The platform catalog is the source of truth; Nexus syncs it on a schedule and on demand, and adds its own data on top.

From the platform catalog:

- Name and plain-language description
- Input and output schemas
- Model and version

Nexus-owned, added on top:

- Tags/metadata: domain, data source, system vs. domain agent
- Example prompts / intents it handles well
- Observed latency and failure rate (from execution tracking)
- Feedback signals (how often it actually helped)

The registry is shared across tenants, matching the catalog.

Indexed two ways: embeddings (from the in-process model, computed over the description plus example prompts) for semantic search, structured fields for filtering. When the catalog sync sees a new or changed agent, it re-indexes that agent. An admin API lets teams edit the Nexus-owned fields.

## 4. Two-Stage Retrieval

**Stage 1: Recall (fast, broad)**

- Hybrid search: vector similarity + keyword (BM25)
- Metadata filters: domain, system agents excluded (applied inside the search, not after it)
- Thousands → \~50 candidates in milliseconds
- Goal: don't miss the right agent

**Stage 2: Precision (slower, accurate)**

- The `nexus-scorer` system agent scores the ~50 candidates against the sub-task (one call covers all sub-tasks of a request)
- \~50 → top 5–10 agents per sub-task
- Only these are passed to the planner

## 5. Planner / Orchestrator (LangGraph)

- Calls the `nexus-planner` system agent, which returns a structured plan: which agents, with what inputs, in what order, what runs in parallel
- Nexus validates the plan (agents exist, inputs match their schemas) and executes it as a stateful graph
- Handles retries and failures
- Calls `nexus-reflector` after each round; loops back to retrieval if results reveal new information needs
- Checkpointing for long-running tasks

## 6. Agent Execution Layer

- Every domain agent is on the AI Platform Service, so there is one transport: submit (agent name in path, tenant in header, input in body), then poll the execution ID with exponential backoff
- Input validated before submitting, output validated when the execution succeeds, both against the agent's catalog schemas
- Timeouts, circuit breaking
- Latency and failure tracking per call (the platform does not report cost)

## 7. Synthesis

- Calls the `nexus-synthesizer` system agent to merge all agent outputs into one coherent response
- Can cite which agent produced which piece of information

## Runtime walkthrough: from intent to answer

Example prompt: "Is IP 1.2.3.4 malicious, and did any of our users connect to it?"

1. **Decompose.** `nexus-intent` splits the prompt into sub-tasks: (a) check IP reputation, (b) find users who connected to the IP.
2. **Retrieve per sub-task.** Query the registry once per sub-task, then have `nexus-scorer` narrow the candidates; get the top few agents for each, with their input schemas.
3. **Plan.** `nexus-planner` receives the sub-tasks plus candidate agents (names, descriptions, input schemas) and returns a plan with agents and inputs. Independent calls run in parallel; dependent calls (e.g. users first, then each user's devices) run in sequence.
4. **Execute.** The execution layer submits each agent call to the AI Platform Service (agent name in the path, tenant ID in the header, input in the body), polls each execution ID until it finishes, and validates each output against the output schema.
5. **Reflect.** Results go to `nexus-reflector`, which decides whether there is enough. If not (e.g. IP is malicious and 3 users touched it → need their recent activity), Nexus loops back to retrieval for the new need. Cap the number of loops.
6. **Synthesize.** `nexus-synthesizer` merges all results into the answer.

Core loop: retrieve → plan → call → reflect, until answered or the loop limit is hit.

### Calling agents

- All agents live on the AI Platform Service behind one REST API → no A2A needed; plain HTTP with agent name in the path, tenant ID in the header and a JSON body.
- Only the tenant ID is sent to the platform; the actor ID stays inside Nexus (logs, traces).
- Per-agent timeouts and concurrency limits so one slow agent can't degrade the rest.

### Design details for the workflow

- **Data passing between steps:** explicit orchestrator state carries outputs forward and maps them into the next agent's input schema fields (LangGraph state, designed deliberately).
- **Budgets:** per-request limits on loop count, total agent calls and wall-clock time (no token or cost data is available). System-agent calls count toward these too.
- **Partial failure:** if an agent fails or times out, return an answer from what succeeded and state what is missing, rather than failing the whole request.
- **Ambiguous prompts:** `nexus-intent` can return a clarifying question instead of guessing which agents to call. The request moves to `needs_input` and resumes when the caller posts the answer.

## 8. Cross-Cutting Concerns

- **Caching:** Redis. The catalog and routing decisions are shared across tenants; cached agent results are always keyed by tenant, since they contain tenant data
- **Tracing:** end-to-end spans across every stage, including every platform call
- **Feedback store:** records which agents actually helped, used to tune retrieval and scoring over time
- **Guardrails:** all agents are read-only in v1, so they run without approval. If action-taking agents are added later: tag them in the registry (untagged = action-taking), pause the graph at a checkpoint with an `awaiting_approval` request status, and resume when the caller posts an approve/reject decision

## 9. Evaluation

There are no existing example prompts, so the first test set is generated from the catalog:

1. **Generate.** Offline, `nexus-example-generator` writes about 10 realistic prompts for each domain agent from its name, description and schemas. It runs as a batch job and reruns for any agent that's new or changed.
2. **Split.** For each agent, about 5 prompts go into the registry as its example prompts, where they feed embeddings and keyword search. The other prompts are held back as the test set and never indexed, so retrieval isn't tested on text it was built from.
3. **Compound prompts.** Pairs of related agents are combined into prompts with two sub-tasks, to test intent decomposition.
4. **Measure.** Stage 1: is the right agent in the top 50? Stage 2: is it in the top 5? Intent: does it find the right sub-tasks? This runs on every change to retrieval, the scorer prompt or the embedding model.
5. **Replace with real data.** Generated prompts are cleaner than real ones, so the scores they give are optimistic. Once Nexus is in use, real prompts from traces and the feedback store become the main test set.

## 10. Developing without LLM access

Until the AI Platform Service is available, Nexus is built and tested against a **fake platform**: a small FastAPI app that implements the assumed contract ([AI Platform Service API.md](AI%20Platform%20Service%20API.md)). Nexus can't tell it apart from the real one; switching is a base-URL change.

### Fake platform

- Catalog served from a fixture file
- Async executions with execution IDs, configurable latency, and fault injection (failures, timeouts, `429`s, output that doesn't match its schema, executions that never finish)

### Fake domain agents

- A generated catalog of ~2,000 agents (domain × data source × operation) with realistic descriptions and schemas, so retrieval is tested at real scale
- A small set of hand-written "scenario" agents with fixed outputs, e.g. IP reputation, user connection search, device lookup
- All other agents return data generated from their output schema

### Fake system agents

Two modes:

- **Scripted:** the output for each test scenario is written into its fixture, so exact paths can be tested: one round, a reflection loop, partial failure, a clarifying question, the budget being hit. CI uses this mode.
- **Heuristic:** deterministic rules that work for any prompt, for demos and manual testing.
  - Intent: split on conjunctions; pull out IPs, users and hosts with regexes
  - Scorer: embedding similarity
  - Planner: best agent per sub-task, inputs filled by matching extracted values to schema field names
  - Reflector: done after one round, unless the scenario asks for more
  - Synthesizer: a template that lists each result with its citation

### What can be validated before LLM access

- The whole pipeline end to end: API, async polling, checkpoints and resume, parallel and sequential steps, loops, budgets, partial failure, clarifying questions
- Stage 1 retrieval quality: it uses the in-process embedding model, not an LLM, so how often the right agent lands in the top 50 can be measured now against a hand-written test set
- System-agent contracts: every fake output is validated against the system agent specs, so when the real agents exist, the same tests check that they return the agreed shapes

### What has to wait for LLM access

- How well the system agents perform: intent decomposition, scorer precision, planner input mapping, and answer quality
- Real latency per round trip, and tuning of the polling intervals

### When access arrives

- Create the system agents from the repo specs and point Nexus at the real platform
- Run the same scenario suite in live mode, checking structure (valid plan, correct agents picked, fields present) rather than exact text
- Record live responses so CI can replay them without calling the platform

## Decisions log

- **Embeddings:** small embedding model run inside Nexus.
- **Catalog:** the same for all tenants; no per-tenant agent visibility.
- **Identity:** tenant ID + optional actor ID; no per-user permissions in v1.
- **System agents:** created manually on the platform; specs kept in the repo and checked against the catalog at startup.
- **Usage/cost:** not reported by the platform; budgets use loop count, call count and wall-clock time.
- **Platform calls:** async; submit returns an execution ID, polled with exponential backoff. Contract assumed in [AI Platform Service API.md](AI%20Platform%20Service%20API.md) until the real one is available.
- **Caller:** a backend service. The Nexus API is async too (submit, poll or callback), with a `needs_input` state for clarifying questions.
- **Agents:** all domain agents are read-only in v1; no human-approval flow.
- **Example prompts and test data:** none exist; generated from the catalog by `nexus-example-generator`, then replaced by real traffic over time.

## Key idea

The registry and retrieval decide what is **possible**. The orchestrator decides what to **actually do**.
