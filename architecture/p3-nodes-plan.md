# P3 plan: nodes

*Approved 2026-10-10. Built from the reference architecture ([§5.3](caliban-reference-architecture.md#53-nodes-agents--sub-apps), [§8](caliban-reference-architecture.md#8-phased-roadmap)), research note [04](../research/04-agent-orchestration.md), and the state of `core` at `8c1af74`.*

**Goal (from the roadmap):** "Build AI sub-apps." A customer declares a node (prompt, tools, datasource scopes, budgets), publishes a version, and calls it over HTTP or MCP, or lets `caliban/auto` pick it.

**Exit criteria (from the roadmap):**
- pass^k and cost per success are reported per node.
- An AgentDojo-style injection suite runs in CI.

## Where we start

| Piece | State today |
|---|---|
| Node spec | `schemas/node.schema.json` and `NodeSpec` in `caliban-nodes`. Static checks: MCP tools must be pinned, workflows need a graph, and every cycle needs `max_iterations`. |
| Node storage | The CP stores node versions (`NodeRecord`: tenant, name, version, spec, soft delete) behind `NodesRead` / `NodesWrite`. The routers never see them. |
| Budgets | An in-memory hierarchical ledger (steps and tokens only). Nothing persists it, and no executor uses it. |
| MCP | Tool-manifest pinning (sha256 over name, description and schema). No client and no server. |
| Executor, journal, templates, plan cache, evals | Not started. |
| Things P3 builds on and which are already done | SSO and roles, per-tenant DEKs, Idempotency-Key, `caliban/auto` with per-tenant intents and exemplars, semantic cache, PII surrogates, metering with the flat price, CQIR datasource lanes. |

## Principles

1. **Every model call a node makes goes back through the gateway pipeline.** PII, cache, routing, quotas, metering and tracing then apply to node traffic for free, and a node can never reach a provider by a side door.
2. **Graphs, not chat rooms.** Explicit typed graphs and a bounded agent loop. Multi-agent only means "a node calls a node". Fan-out only where the author declares it.
3. **Security is enforced in Rust, never in prompts:**
   - pinned tools;
   - taint labels;
   - no write with tainted arguments without an allowlist entry or a human approval;
   - minted, down-scoped tokens and no passthrough;
   - egress allowlist.
4. **Durable by default.** Every step is checkpointed in Postgres. A replayed step never repeats a charged model call or an external write.

## Milestones

Each milestone is one batch of work that can be shipped and tested. They are in dependency order.

### M1. Node versions become deployable
- Versions are immutable and content-hashed, with `draft → published → retired` states and a promotion pointer per node (`invoice-triage` → `@v7`). Promotion is audited and needs a new `NodesPublish` permission.
- Published versions travel to routers in the signed snapshot, in tenant-sealed form, like datasources.
- Validation at publish time. On top of today's checks:
  - tool refs must resolve to approved, pinned manifests;
  - datasource scopes must exist in the tenant's ontology;
  - budgets must sit inside the tenant's node caps.
- Add a `NodesRun` permission. API keys get a node allowlist (default: all published nodes of the tenant).

### M2. Journal and executor (the core of P3)
- **Postgres journal** in `caliban-nodes`, following the decided Absurd model:
  - Tables:
    - `node_run`: run id, tenant, node@version, invoker, input, status, budget snapshot, lease;
    - `node_step`: run id, step id, attempt, vertex, input hash, result, tokens, USD, timings;
    - `node_event`: awaited events and human answers.
  - Workers claim runnable rows with `FOR UPDATE SKIP LOCKED`, under a lease and heartbeat. Sleeps and awaited events are durable.
  - Results are sealed with the tenant DEK, like other tenant data.
  - Idempotency key on every side effect = hash(run id, step id). Model calls reuse the gateway's Idempotency-Key, so a replay hits the stored response instead of paying twice.
  - Durability: checkpoint every step (one row write costs about 1 ms, against model calls in the hundreds of ms).
- **Executor** (tokio):
  - Vertex kinds in this order: `llm`, `tool`, `router`, `map`/`reduce`, `verify`, `subnode`, `human`. `code` (WASM) comes last, in M8.
  - Replay reads recorded results.
  - Run modes: sync (with a wall-clock cap), streaming (SSE of step events and the final output), and async (returns a run id to poll).
- **Where it runs:** a `worker` role of the same binary (decision 1).

### M3. Budgets and guards
- Extend the ledger with USD, wall clock, depth and fan-out. Persist it in the journal, so a resumed run keeps what it already spent.
- Spend caps at every level: tenant (daily and monthly) → node → run → subnode. A child gets at most what its parent has left.
- On overrun, the run ends gracefully with partial results and a clear reason.
- Loop guard: repeated (vertex, input hash) pairs; capped tool retries; a circuit breaker per tool.
- Metering: each model call inside a run is billed exactly as it is today (flat `caliban/auto` price or the pinned model's price, cache-hit discount included). Usage events gain `node`, `node_version` and `run_id`, so cost per run and per node fall out of the existing usage data.

### M4. Tools: Caliban as the MCP client
- An `rmcp` client to the customer's MCP servers, plus a tool registry per tenant:
  - pinned manifests;
  - a manifest change needs re-approval, and `tools/list_changed` is never auto-trusted;
  - descriptions are scanned for injection on approval.
- Built-in tools that need no MCP server:
  - the tenant's datasources through the CQIR lanes (read only, scoped by the node's datasource scopes and the invoker's roles);
  - RAG search;
  - calling another node.
- Per-call tokens minted for (tenant, node, tool), bound to an audience and down-scoped. Client tokens are never passed through.
- Egress allowlist with private IPs blocked (SSRF).
- Taint (CaMeL-lite):
  - values from tool outputs, retrieved documents and other tenants are labelled;
  - `effect: write` with tainted arguments needs an allowlist entry or a human approval;
  - `pii`-labelled values never leave through an egress the policy does not allow.
- PII surrogates stay in place across tool calls. Values are rehydrated only for tools the tenant marks as trusted.

### M5. Exposure: HTTP and MCP server
- HTTP API:
  - `POST /v1/nodes/{name}/runs` (sync, stream, async);
  - `GET /v1/runs/{id}` and its events;
  - `POST /v1/runs/{id}/input` for human answers;
  - Idempotency-Key supported.
- OpenAI-compatible shortcut: `model: "node/<name>"` on chat completions runs the node, so existing SDK users need no new client.
- MCP server (`rmcp`): `tools/list` lists the published nodes the key may run, and a node's input schema is the tool's input schema. Long runs use MCP Tasks behind a flag, because Tasks is still experimental in the spec.
- A2A comes in M8 (decision 5).

### M6. `caliban/auto` picks a node
- A tenant maps intents to nodes (`[routing.tenants.<id>.routes] triage = "node/triage@published"`). This uses the per-tenant intents and exemplars that already exist.
- When a `caliban/auto` request classifies into an intent that has a node, the node runs. Below the confidence threshold, or if the node is over budget, the request falls back to a plain model call, and the response says which path was taken.
- This is the "classify, then hand off to the right sub-agent" flow, and it adds no new classifier, because it reuses the kNN step already measured at about 5 ms.

### M7. Templates and the plan cache
- Templates ship as ready-made node configs that a tenant copies and edits:
  - the five Anthropic workflow patterns (chain, routing, parallelization, orchestrator-workers, evaluator-optimizer);
  - bounded ReAct;
  - plan-and-execute (ReWOO / LLMCompiler);
  - dual-LLM.
- CaMeL-secure is opt-in (decision 3).
- Plan format: a JSON plan IR validated in Rust (decision 2). The executor checks every plan against the node's allowed tools and budgets before running it, then runs independent branches in parallel.
- Plan cache: plans are cached, never answers.
  - Key: tenant, node version, intent, hash of the tool schemas, hash of the datasource scopes.
  - Lookup: by vector on the request embedding the gateway already computed.
  - Never shared across tenants. Invalidated on any tool or scope change.

### M8. Evals, console, SDK, and the rest
- Evals:
  - golden sets per node;
  - pass^k over k runs, cost per success and p95 latency, with an accuracy × cost chart across routing policies;
  - a version can be promoted only if its eval passes the bar set on the node.
- Security CI: an AgentDojo-style injection suite runs against every node version that has write tools. The pass rate is a promotion gate.
- Console:
  - node list, versions and diff;
  - run list and trace view (OTel GenAI spans: `invoke_agent`, `chat`, `execute_tool`);
  - inbox for human approvals;
  - MAST failure labels on failed runs.
- SDK (TypeScript and Python): author and validate nodes, run them, stream events, run evals.
- A2A agent cards and task states (decision 5).
- `code` vertices in a wasmtime sandbox (no network, no filesystem, fuel and memory limits).
- A third AWS run (about $5): executor throughput, journal write cost, plan-cache hit rate, the overhead of a node step against a plain call, and the injection suite against an on-prem model.

**Out of P3:**
- long-term memory (Mem0-style);
- marketplace nodes shared between tenants;
- offline optimizers (DSPy / AFlow) in the SDK.

Each needs P3 to exist first, and none of them is needed for the exit criteria.

## Decisions (made 2026-10-10)

| # | Question | Decision |
|---|---|---|
| 1 | **Where do nodes run?** | A `worker` role of the same binary, with access to the Postgres journal. Routers forward node runs to workers. Standalone mode runs everything in one process. Routers still need no database, so the "data plane holds no authoritative state" rule holds. |
| 2 | **Plan format** | Our own JSON plan IR, validated in Rust. LLMCompiler's text format is harder to validate, and a CaMeL Python subset is a much bigger build. |
| 3 | **CaMeL by default?** | No. Every node with write tools gets taint labels and the "no tainted writes without approval" rule by default, which covers the main attack. Full CaMeL-secure (capabilities on every value) is a template you opt into: it solves fewer tasks (67% of AgentDojo). |
| 4 | **Billing for node runs** | No per-run fee for now. A run costs the sum of its model calls at today's prices, and usage shows the cost per run and per node. A node fee can be added later, once we see real run costs. |
| 5 | **A2A in P3?** | MCP and HTTP first, because every client already speaks them. A2A at the end of P3, in M8, mostly for long-running and cross-organization nodes. |
| 6 | **First reference node** | Build one real node alongside M2 to M6 as the test case: a triage node (classify the case, ask a clarifying question through the human step, recommend services from a catalogue tool, with PII masked throughout). It doubles as the demo for the health platform conversation. |

## Order of work

1. M1 and M2 together (the spec, versions, journal and executor).
2. M3 and M4 (budgets, tools and security).
3. M5 and M6 (exposure and auto-routing to nodes). At this point a customer can build and call a node, so the reference node and an internal demo come here.
4. M7 (templates and the plan cache).
5. M8 (evals, console, SDK, A2A, WASM), then the AWS run.

Migrations continue from the next free number after the pre-P3 fixes. Old migrations are never edited.
