# Agent Creation & Multi-Agent Orchestration ("Nodes") for Caliban

*Research date: 2026-10-02. Scope: how Caliban should let customers build sub-apps ("nodes") on top of the gateway.*

**Summary.** The literature from 2023 to 2026 points the same way. Explicit, structured control flow beats free-form agent chatter. Caliban should therefore ship a typed graph runtime that is durable and budgeted, not a "society of agents" framework. Free-form multi-agent systems fail often: MAST measured 41–86.7% failure rates across 7 frameworks, mostly from system design and coordination problems rather than model limits. A 260-configuration scaling study found multi-agent setups range from +80.8% to −70% against single-agent baselines, depending on how well the coordination strategy fits the task. Plan-then-execute patterns (ReWOO, LLMCompiler) bring three gains together:
- **Cost:** up to 5× token efficiency and up to 6.7× cost reduction.
- **Latency:** up to 3.7× faster through parallel DAG execution.
- **Security:** fixing the plan before untrusted data arrives gives control-flow integrity, which is the basis of CaMeL and the 2025 prompt-injection design patterns.

Plans can also be cached (Agentic Plan Caching: −50% cost, −27% latency), which fits Caliban's vector-cache data plane. On protocols, the market has settled on **MCP for tools** (spec 2025-11-25: async Tasks, OAuth/OIDC) and **A2A v1.0 for agent-to-agent** (IBM's ACP merged into A2A in Aug 2025). Every Caliban node should be callable through both. The main risks are prompt injection through tool outputs and tool metadata (tool poisoning reaches >72% attack success), runaway loops and cost, cascading errors, and confused-deputy token handling across tenants. All of these must be enforced in the Rust core, not left to prompts.

---

## Key papers

| Paper | Year | Core idea | Why it matters for Caliban | Verdict |
|---|---|---|---|---|
| [ReAct](https://arxiv.org/pdf/2210.03629) | 2022 | Interleave reasoning traces with tool actions in one loop. | Baseline "agent loop" every customer expects; also the most expensive and injection-prone pattern (every observation re-enters the context). | **Adopt** as the default single-node loop, bounded by budgets |
| [Reflexion](https://arxiv.org/abs/2303.11366) | 2023 | Agents verbally reflect on feedback and store reflections in episodic memory for later trials. | A cheap "retry with critique" node type; reflections can be stored in node memory across runs. | **Prototype** |
| [Toolformer](https://www.semanticscholar.org/paper/Toolformer:-Language-Models-Can-Teach-Themselves-to-Schick-Dwivedi-Yu/53d128ea815bcc0526856eb5a9c42cc977cb36a7) | 2023 | Self-supervised training so an LM decides when and how to call APIs. | Shows that tool-calling quality comes from the model, which feeds the router's per-model tool-use scores. Caliban will not train this itself. | **Watch** |
| [Gorilla](https://arxiv.org/abs/2305.15334) | 2023 | Fine-tuned LLM + retriever over API docs cuts hallucinated API calls (APIBench). | Supports **retrieval over tool catalogs**: with hundreds of MCP tools per tenant, retrieve the top-k tool schemas instead of stuffing all of them into context. | **Adopt** (tool retrieval) |
| [Plan-and-Solve](https://arxiv.org/abs/2305.04091) | 2023 | Plan subtasks first, then execute them. | Conceptual root of "plan-and-execute" node types. | **Adopt** (pattern) |
| [ReWOO](https://arxiv.org/pdf/2305.18323) | 2023 | Planner → Workers → Solver; plan with variable placeholders, no observations during planning. 5× token efficiency on HotpotQA. | Planner output is a static plan that the Rust executor can run without further LLM calls. Big token savings, and the plan is cacheable. | **Adopt** |
| [LLMCompiler](https://arxiv.org/abs/2312.04511) | 2023/ICML'24 | Planner emits a DAG of function calls; task-fetching unit and executor run independent calls in parallel. Up to 3.7× latency, 6.7× cost, ~9% accuracy vs ReAct. | Maps directly onto a Rust/tokio DAG executor in the gateway. This should be the main "fast path" for multi-tool nodes. | **Adopt** |
| [AutoGen](https://arxiv.org/abs/2308.08155) | 2023 | Multi-agent conversation framework; agents converse via natural language and code. | Shows the conversational multi-agent paradigm, which MAST later found fragile. Useful as an interop target, not as a runtime model. | **Watch** |
| [MetaGPT](https://arxiv.org/abs/2308.00352) | 2023/ICLR'24 | Encode SOPs into roles; require structured intermediate artifacts. | Structured handoffs (typed outputs between nodes) reduce compounding errors. Use typed edges. | **Adopt** (typed artifacts) |
| [ChatDev](https://arxiv.org/abs/2307.07924) | 2023/ACL'24 | "Chat chain" of role agents with communicative dehallucination for software development. | Domain-specific; confirms the value of phase-structured pipelines over open chat. | **Skip** (as runtime) |
| [CAMEL](https://arxiv.org/pdf/2303.17760) | 2023 | Role-playing agents with inception prompting for autonomous cooperation. | Useful for synthetic data and eval generation, not for production orchestration. | **Skip** |
| [AgentVerse](https://arxiv.org/abs/2308.10848) | 2023 | Dynamic expert recruitment + collaborative decision-making. | "Recruit a team at runtime" is hard to budget and govern. Prefer static graphs with router nodes. | **Watch** |
| [DSPy](https://arxiv.org/abs/2310.03714) | 2023 | Declarative LM modules compiled and optimized against a metric. | Model for an **offline node optimizer**: tune prompts and few-shots per node against customer eval sets. Belongs in the SDK, not the core. | **Prototype** (SDK) |
| [GPTSwarm](https://www.alphaxiv.org/abs/2402.16823) | 2024/ICML | Agents as optimizable computational graphs; optimize node prompts and edges. | Matches the Caliban node-graph IR; edge pruning can lower token cost. | **Watch** |
| [ADAS](https://arxiv.org/abs/2408.08435) | 2024 | Meta-agent searches the space of agent designs written as code. | Long-term "auto-build my node" feature; too unconstrained for governed production. | **Watch** |
| [AFlow](https://arxiv.org/abs/2410.10762) | 2024/ICLR'25 oral | MCTS over code-represented workflows; smaller models beat GPT-4o at 4.55% of the cost. | Strong evidence for "optimize the workflow, then route to cheaper models", which is Caliban's thesis. Offline optimizer candidate. | **Prototype** (offline) |
| [MemGPT](https://arxiv.org/pdf/2310.08560) | 2023 | OS-style memory hierarchy; the LLM pages context in and out of external storage. | Basis for tiered node memory (core / recall / archival). | **Adopt** (concepts) |
| [Mem0](https://arxiv.org/pdf/2504.19413) | 2025 | Extract, consolidate and retrieve salient facts; 91% lower p95 latency, >90% token savings vs full context. | Shows that managed memory is a token- and latency-saving feature that fits Caliban's value proposition. | **Adopt** (pattern) |
| [A-MEM](https://arxiv.org/abs/2502.12110) | 2025 | Zettelkasten-style linked memory notes that evolve over time. | Richer memory graph; more LLM calls per write. Offer as an opt-in memory policy. | **Watch** |
| [Agentic Plan Caching](https://arxiv.org/abs/2506.14852) | 2025/NeurIPS | Extract and adapt plan templates from past runs for semantically similar tasks; −50.31% cost, −27.28% latency. | Direct fit for the vector-DB cache: cache **plans**, not answers, which avoids serving stale data. | **Adopt** |
| [SemanticALLI](https://arxiv.org/html/2601.16286v1) | 2026 | Cache reasoning artifacts rather than only responses in agentic systems. | Supports plan/reasoning-level caching. | **Watch** |
| [Workload-Aware Caching for MAS](https://arxiv.org/abs/2607.20495) | 2026 | Eviction using recompute cost, dependency counts and frequency; up to 64.7% latency reduction. | Eviction policy for the shared node-output / plan cache. | **Prototype** |
| [Why Do Multi-Agent LLM Systems Fail? (MAST)](https://arxiv.org/abs/2503.13657) | 2025 | 14 failure modes in 3 classes (system design, inter-agent misalignment, task verification); 41–86.7% failure rates. | Use MAST as the **failure-label taxonomy** in Caliban's trace UI and eval loop. | **Adopt** |
| [Towards a Science of Scaling Agent Systems](https://arxiv.org/abs/2512.08296) | 2025/26 | 260 configs × 6 benchmarks: multi-agent ranges from +80.8% to −70% vs single agent, depending on task/coordination fit. | Multi-agent must not be the default. The router should choose topology per task. | **Adopt** (as guidance) |
| [Rethinking Multi-Agent Collaboration: When More Is Less](https://arxiv.org/abs/2609.19759) | 2026 | Multi-agent helps on long-horizon, sparse-dependency tasks; single agent wins on sequential workflows. | Same conclusion; gives a heuristic for when to fan out. | **Watch** |
| [AI Agents That Matter](https://arxiv.org/abs/2407.01502) | 2024 | Agent evals must be cost-controlled; simple baselines (retry, warming) often match complex agents. | Every Caliban node eval reports an accuracy × $ Pareto, not accuracy alone. | **Adopt** |
| [τ-bench](https://arxiv.org/abs/2406.12045) | 2024 | Tool-agent-user benchmark with policy adherence; GPT-4o <50% success; pass^k reliability metric. | Use pass^k (consistency across k runs) as the node reliability metric. | **Adopt** (metric) |
| [AgentDojo](https://arxiv.org/abs/2406.13352) | 2024/NeurIPS | 97 tasks and 629 injection cases; more capable models are often easier to attack; tool isolation is the most effective defense. | Security regression suite for Caliban's runtime guardrails. | **Adopt** |
| [CaMeL: Defeating Prompt Injections by Design](https://arxiv.org/abs/2503.18813v1) | 2025 | Extract control and data flow from the trusted query; untrusted data cannot change program flow; capabilities block exfiltration. 67% of AgentDojo tasks with provable security. | Blueprint for a "secure mode" node: P-LLM plans, Q-LLM parses, Rust interpreter enforces taint/capability policy. | **Prototype** |
| [Design Patterns for Securing LLM Agents against Prompt Injections](https://arxiv.org/abs/2506.08837) | 2025 | Action-selector, plan-then-execute, map-reduce, dual-LLM, and other patterns; untrusted input must not trigger consequential actions. | Directly sets which node templates Caliban ships with security guarantees. | **Adopt** |
| [Adaptive Attacks Break Defenses Against Indirect Prompt Injection](https://arxiv.org/pdf/2503.00061) | 2025 | Adaptive attacks bypass the evaluated IPI defenses. | Do not rely on classifier or prompt-based defenses alone; enforce architecturally. | **Adopt** (as caution) |
| [MCPTox](https://arxiv.org/html/2508.14925v1) | 2025 | Benchmark for tool-poisoning on real MCP servers; >72% attack success. | Tool descriptions are an attack surface. Pin, hash and review MCP tool manifests per tenant. | **Adopt** |
| [MCP Threat Modeling & Tool Poisoning](https://arxiv.org/abs/2603.22489) | 2026 | Compares 7 MCP clients; tool poisoning is the most prevalent client-side vuln. | Caliban is an MCP client; it needs manifest signing and vetting. | **Adopt** |
| [ETDI](https://arxiv.org/pdf/2506.01333) | 2025 | OAuth-enhanced tool definitions + policy-based access control against tool squatting and rug pulls. | Pattern for signed, versioned tool definitions in Caliban's tool registry. | **Prototype** |
| [SoK: Security and Safety in the MCP Ecosystem](https://arxiv.org/pdf/2512.08290) | 2025 | Systematization of MCP security risks. | Checklist for MCP gateway hardening. | **Adopt** (reference) |
| [Survey of Agent Interoperability Protocols (MCP/ACP/A2A/ANP)](https://arxiv.org/abs/2505.02279) | 2025 | Compares 4 protocols; proposes a phased roadmap: MCP → ACP → A2A → ANP. | Confirms MCP for tools + A2A for agents. ACP has since merged into A2A. | **Adopt** (reference) |
| [A Survey of AI Agent Protocols](https://arxiv.org/pdf/2504.16736) | 2025 | Taxonomy of context-oriented vs inter-agent protocols. | Background for protocol strategy. | **Watch** |
| [Comparative Study of MCP and A2A for Inter-Agent Coordination](https://arxiv.org/abs/2607.23884) | 2026 | MCP can coordinate agents with a lighter implementation; A2A has richer native stateful interaction at higher complexity. | Ship MCP exposure first (cheap); add A2A for long-running, stateful, cross-org nodes. | **Adopt** |

Practitioner reference: [Anthropic, *Building Effective AI Agents* (Dec 2024)](https://www.anthropic.com/engineering/building-effective-agents). It defines five workflow patterns (prompt chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer) and advises using workflows for well-defined tasks and agents only for open-ended ones. These map one-to-one onto Caliban node templates.

---

## Open-source implementations & protocols

| Project | Language | Use | Notes |
|---|---|---|---|
| [Model Context Protocol spec (2025-11-25)](https://modelcontextprotocol.io/specification/2025-11-25/basic/utilities/tasks) | Spec | Tool/resource/prompt exposure | Experimental **Tasks** primitive (call-now, fetch-later; `execution.taskSupport`). [Authorization](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization) adds OIDC discovery and incremental scope consent. |
| [MCP Security Best Practices](https://modelcontextprotocol.io/specification/2025-11-25/basic/security_best_practices) | Spec | Hardening | Confused deputy, **token passthrough forbidden**, SSRF, session hijack, scope minimization. Mandatory reading. |
| [modelcontextprotocol/rust-sdk (`rmcp`)](https://github.com/modelcontextprotocol/rust-sdk) | **Rust** | MCP client + server | **Rust crate.** Official, tokio-based, `rmcp-macros` for tools. Use in the core to both consume customer MCP servers and expose nodes as MCP. |
| [A2A protocol](https://github.com/a2aproject/a2a) / [spec v1.0](https://a2a-protocol.org/latest/specification/) | Spec + SDKs | Agent-to-agent | Linux Foundation; Agent Cards; task states (SUBMITTED, WORKING, INPUT_REQUIRED, AUTH_REQUIRED, …); JSON-RPC / gRPC / HTTP+JSON; streaming + push. SDK list includes **Rust**. |
| [ACP (i-am-bee/acp)](https://github.com/i-am-bee/acp) | Spec | Agent messaging | **Merged into A2A** ([LF AI & Data, Aug 2025](https://lfaidata.foundation/communityblog/2025/08/29/acp-joins-forces-with-a2a-under-the-linux-foundations-lf-ai-data/)). Do not implement separately. |
| [Rig](https://github.com/0xPlaygrounds/rig) | **Rust** | LLM/agent library | **Rust crate.** MIT; 20+ providers, 10+ vector stores, agent loop, tool registry, GenAI-semconv-compatible tracing, checkpoints. Breaking changes still expected. Good reference and possibly a provider layer; not a durable orchestrator. |
| [Swiftide](https://github.com/bosun-ai/swiftide) | **Rust** | RAG pipelines + agents + task graphs | **Rust crate.** Streaming indexing/query pipelines and typed task graphs; pre-1.0. Strong candidate for the datasource-ingestion side. |
| [ADK-Rust](https://github.com/zavora-ai/adk-rust) / [`adk-graph`](https://crates.io/crates/adk-graph) | **Rust** | Agent dev kit, LangGraph-style graphs | **Rust crates.** Sequential/Parallel/Loop agents + graph workflows. Community project (not Google-official); evaluate maturity. |
| [AutoAgents](https://github.com/liquidos-ai/autoagents) | **Rust** | Multi-agent framework | **Rust crate.** Typed pub/sub between agents, pluggable memory and LLMs. Watch. |
| [Restate](https://github.com/restatedev/restate) | **Rust** (server) | Durable execution / workflows-as-code | **Server written in Rust**; markets "durable AI agents". Closest embeddable analog to Temporal in Rust. Check license before embedding. |
| [Temporal + OpenAI Agents SDK](https://temporal.io/blog/announcing-openai-agents-sdk-integration) | Go server; Py/TS SDKs | Durable agent execution | Agent loop as Workflow, LLM/tool calls as Activities; GA Mar 2026. Reference architecture for "workflow = deterministic, activity = side effect". |
| [LangGraph](https://github.com/langchain-ai/langgraph) | Python/JS | Stateful graph orchestration | MIT; Pregel-inspired; [checkpointers](https://docs.langchain.com/oss/python/langgraph/checkpointers) with durability modes (exit/async/sync), [interrupts](https://docs.langchain.com/oss/python/langgraph/interrupts) for HITL. Caveat: nodes after a checkpoint re-execute, including LLM calls. The de-facto concept model to mirror. |
| [LLMCompiler](https://github.com/SqueezeAILab/LLMCompiler) | Python | Parallel function-calling planner | Reference planner prompt + DAG format. |
| [AFlow](https://github.com/FoundationAgents/AFlow) | Python | Workflow search | Offline optimizer reference. |
| [DSPy](https://github.com/stanfordnlp/dspy) | Python | Prompt/program optimization | SDK-side optimizer for node prompts. |
| [Mem0](https://github.com/mem0ai/mem0) | Python | Agent memory layer | Reference for memory extract/consolidate API shape. |
| [CaMeL reference](https://github.com/google-research/camel-prompt-injection) | Python | Capability-based injection defense | Research code; port the interpreter and policy ideas to Rust. |
| [OTel GenAI agent spans](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/gen-ai-agent-spans.md) | Spec | Tracing | `invoke_agent`, `chat`, `execute_tool`, `create_agent`, `embeddings` ops; `gen_ai.*` attributes; content capture opt-in. Now lives in a dedicated repo. |
| [Dual LLM pattern (Willison)](https://simonwillison.net/2023/Apr/25/dual-llm-pattern/) | Pattern | Injection isolation | Privileged LLM (tools, trusted input) + quarantined LLM (no tools, untrusted data) + orchestrator that passes symbolic references. |

---

## Risks & failure modes

| Risk | Evidence | Mitigation in Caliban |
|---|---|---|
| **Indirect prompt injection via tool outputs / RAG** | AgentDojo: capable models are often *easier* to attack, and tool isolation works best ([2406.13352](https://arxiv.org/abs/2406.13352)). Adaptive attacks break published defenses ([2503.00061](https://arxiv.org/pdf/2503.00061)). | Architectural, not prompt-based: plan-then-execute and dual-LLM templates ([2506.08837](https://arxiv.org/abs/2506.08837)); CaMeL-style taint labels on every value ([2503.18813](https://arxiv.org/abs/2503.18813v1)); consequential tools require untainted args or human approval. |
| **Tool poisoning / rug pulls in MCP metadata** | >72% attack success ([MCPTox](https://arxiv.org/html/2508.14925v1)); the most prevalent client-side vuln ([2603.22489](https://arxiv.org/abs/2603.22489)). | Tool registry pins `{server, tool, schema hash, description hash}`; changes require re-approval ([ETDI](https://arxiv.org/pdf/2506.01333)); scan descriptions; never auto-trust `tools/list_changed`. |
| **Confused deputy / token passthrough across tenants** | MCP spec: proxies with static client IDs + DCR + consent cookies allow code theft; token passthrough is **forbidden** ([MCP security](https://modelcontextprotocol.io/specification/2025-11-25/basic/security_best_practices)). | Caliban is exactly an "MCP proxy server". Per-client consent registry; audience-bound tokens minted per (tenant, node, tool); token exchange with down-scoping on each delegation hop; session keys `<tenant>:<user>:<session>`; egress proxy that blocks private IPs (SSRF). |
| **Runaway loops & cost blowups** | SOTA agents are "needlessly complex and costly" ([AI Agents That Matter](https://arxiv.org/abs/2407.01502)); MAST lists step repetition and unaware-of-termination failures ([2503.13657](https://arxiv.org/abs/2503.13657)). | Hard budgets in the executor: max steps, max depth, max fan-out, tokens, $ and wall clock per run *and* per tenant; repeated-state detection (hash of node + inputs); circuit breakers per tool. |
| **Cascading / compounding errors across agents** | MAST: inter-agent misalignment and weak verification dominate; multi-agent can degrade performance by up to 70% ([2512.08296](https://arxiv.org/abs/2512.08296)). | Typed edges (JSON Schema) between nodes, as in MetaGPT's structured artifacts; verifier nodes on critical edges; single-agent by default; fan-out only for parallel, sparse-dependency subtasks ([2609.19759](https://arxiv.org/abs/2609.19759)). |
| **Non-determinism on replay** | LangGraph re-executes nodes after the last checkpoint, including LLM calls ([docs](https://docs.langchain.com/oss/python/langgraph/checkpointers)). | Journal every LLM/tool result (Temporal/Restate model); replay reads from the journal; idempotency keys on side-effecting tools. |
| **Memory poisoning / cross-user leakage** | Memory systems persist and evolve content ([A-MEM](https://arxiv.org/abs/2502.12110), [Mem0](https://arxiv.org/pdf/2504.19413)), so an injected fact becomes durable. | Memory is scoped (tenant / node / end-user), taint-labeled at write, PII-anonymized before storage, and deletable via a TTL/erasure API. |
| **Stale cached plans / answers** | Response caching fails for agents because outputs depend on live data ([APC](https://arxiv.org/abs/2506.14852)). | Cache *plans/templates*, re-execute tools; include tool-schema versions and datasource-scope hashes in the cache key. |

---

## Recommended design for Caliban

### 1. The node model (a node is declarative config, not code)

```yaml
node: invoice-triage@v7            # immutable, versioned, content-hashed
kind: agent | workflow             # agent = bounded loop; workflow = explicit graph
prompt: { system: ..., templates: ..., output_schema: <JSON Schema> }
model_policy:                      # handed to Caliban's router, not a fixed model
  intent_classes: [extraction, reasoning]
  candidates: [tier:small, tier:frontier]
  escalate_on: [schema_violation, low_confidence]
  max_cost_usd: 0.05
tools:                             # pinned MCP tools + other nodes (as tools)
  - mcp://erp/lookup_invoice#sha256:…   { effect: read }
  - mcp://erp/approve_invoice#sha256:…  { effect: write, requires: [untainted_args, human_approval>1000usd] }
  - node://vendor-risk@v3                { effect: read }
datasources: { scopes: [erp.invoices:read, docs.vendor_contracts:read] }   # ontology-layer scopes
memory: { short_term: run, long_term: { policy: mem0-style, scope: end_user, ttl: 90d } }
guardrails: { pii: anonymize, injection_mode: plan_then_execute | camel | none, egress: allowlist }
budgets: { steps: 20, depth: 3, fanout: 8, tokens: 200k, wall_clock: 120s }
exposure: { mcp_tool: true, a2a_agent: true, http: true }
eval: { suite: invoice-triage-golden, metrics: [pass^k, cost_per_success, mast_labels] }
```

- **Kinds of graph vertices:** `llm`, `tool`, `router` (intent classifier), `map` (parallel fan-out), `reduce`, `verify` (evaluator-optimizer), `human` (interrupt), `subnode` (call another node), `code` (WASM-sandboxed transform).
- **Templates shipped on day one:** the 5 Anthropic workflow patterns, plus ReAct-bounded, ReWOO/LLMCompiler plan-and-execute, dual-LLM, and CaMeL-secure.

### 2. Composition: graphs, not chat rooms

- A workflow is a **typed DAG with bounded cycles**: loops need an explicit `max_iterations` and are compiled to a state machine. Edges carry JSON-Schema-validated artifacts.
- **Planner-generated DAGs (LLMCompiler/ReWOO):** a `plan` vertex emits a DAG in a constrained grammar. The Rust executor validates it against the node's allowed tools and budgets, then runs independent branches concurrently. Plans go through **Agentic Plan Caching** (vector-keyed template reuse).
- **Multi-agent = node calls node.** There is no implicit shared chat. The router decides topology per request (single vs fan-out), guided by the scaling results ([2512.08296](https://arxiv.org/abs/2512.08296)).

### 3. Execution engine (Rust core): durable, resumable, budgeted

- **Event-sourced journal per run.** Every LLM call, tool call, interrupt and state transition is appended to the journal. Replay is deterministic and reads recorded results, following the Temporal "workflow vs activity" split and the Restate journal.
  - **Option A:** embed or partner with Restate (Rust; check its license).
  - **Option B:** build a minimal journal on Postgres/FoundationDB + tokio.
  - Do **not** take a Temporal (Go) dependency for an on-prem "sovereign router".
- **Idempotency keys** on all `effect: write` tools; at-least-once execution with dedupe.
- **Budgets as a first-class resource:** a hierarchical token/$ ledger at the tenant → node → run → subnode levels. A child inherits the remainder of its parent's budget. Overruns trigger graceful termination with partial results.
- **Interrupts / HITL:** `INPUT_REQUIRED` and `AUTH_REQUIRED` states that map 1:1 to A2A task states and MCP Tasks.
- **Loop/termination guard:** detect repeated (vertex, input-hash) pairs; cap tool retries; per-tool circuit breakers.

### 4. Exposure via MCP and A2A

- Every published node becomes an **MCP tool**: it serves `tools/list` through `rmcp`, `execution.taskSupport: optional` for long runs, and the input schema = the node input schema. It also becomes an **A2A agent** (Agent Card generated from node metadata; skills = node entry points).
- Caliban also acts as an **MCP client** to customer tool servers. It is the single policy enforcement point: auth, pinning, taint, PII redaction on the wire.
- Order: MCP first (lighter, already supported by every client), A2A second for long-running, cross-org and stateful nodes ([2607.23884](https://arxiv.org/abs/2607.23884)).

### 5. Per-node permissions (capability model)

- A node runs as a **workload identity** `(tenant, node@version, invoking principal)`. Effective permissions = the intersection of node grants, invoker grants, and tenant policy.
- Tokens to downstream tools are **minted per call**, audience-bound and down-scoped (no passthrough), following MCP security best practices. Incremental scope elevation uses `WWW-Authenticate` challenges.
- **Taint tracking** (CaMeL-lite): values derived from tool outputs, retrieved documents or other tenants are labeled. Policy rules such as "no `effect: write` with tainted args unless allowlisted" or "no egress of `pii` labels" are enforced in Rust, not in prompts.
- Cross-tenant node sharing (marketplace) runs the callee in its **owner's** tenant, with only explicitly declared output fields returned. This prevents confused deputy across tenants.

### 6. Observability

- Emit **OTel GenAI semconv** spans natively from the Rust core: `invoke_agent` (node run / subnode), `chat` (each model call, with routed model + cache hit), `execute_tool`, `embeddings`. Add Caliban attributes: `caliban.node.version`, `caliban.budget.remaining`, `caliban.taint`, `caliban.cache.plan_hit`.
- Message-content capture is opt-in and PII-redacted by default, which matters for on-prem compliance.
- Trace UI lets humans label failed runs with **MAST** categories. These labels feed the eval loop.

### 7. Evaluation loop

- **Per node:** golden sets plus simulated users (τ-bench style). Report **pass^k**, cost-per-success, p95 latency, and an accuracy × $ Pareto across routing policies ([AI Agents That Matter](https://arxiv.org/abs/2407.01502)).
- **Security CI:** an AgentDojo-style injection suite runs against every node version with write tools.
- **Offline optimization (SDK):** DSPy/AFlow-style search over prompts, few-shots, graph shape and model policy. The output is a *new node version* proposed for promotion; it is never auto-deployed.
- **Production feedback:** traces → MAST labels + user feedback → regression sets.

### 8. Rust core vs. higher-level SDK

| Rust core (data plane, also ships on-prem) | SDK / control plane (Python + TS) |
|---|---|
| Graph IR validation, DAG/state-machine executor (tokio) | Node authoring DSL / YAML, visual builder |
| Durable journal, replay, idempotency, interrupts | Eval harness, dataset management, MAST labeling UI |
| Budget ledger, loop guards, circuit breakers | DSPy/AFlow-style offline optimizers |
| MCP client/server (`rmcp`), A2A server/client | Template library (ReAct, ReWOO, dual-LLM, CaMeL) as configs |
| Policy engine: capabilities, taint, PII, egress, tool pinning | Framework adapters (LangGraph, OpenAI Agents SDK, AutoGen) that call Caliban nodes |
| Plan cache + memory store over the vector DB | Analytics/cost dashboards |
| WASM sandbox for `code` vertices (wasmtime) | |
| OTel GenAI span emission | |

Library choice: build on `rmcp` + tokio. Borrow provider abstractions and ideas from Rig and Swiftide (Swiftide for ingestion pipelines). Do not adopt ADK-Rust or AutoAgents as the core, because Caliban's executor *is* the product. Evaluate Restate for the journal.

---

## Open questions

1. **Journal backend:** embed Restate (license and operational fit for air-gapped on-prem?) or build a minimal journal on Postgres? What durability mode is the default: every step, or only at side effects?
2. **Plan DSL:** adopt the LLMCompiler format, a CaMeL-style restricted Python subset, or a custom JSON IR? This trades expressivity against verifiability.
3. **How much CaMeL?** Full capability/taint tracking costs utility (67% of AgentDojo tasks solved). Should it be the default for write-capable nodes or opt-in?
4. **Plan-cache keying:** which features (intent class, tool-schema hashes, datasource scope, tenant) give safe reuse without cross-tenant leakage? Should templates ever be shared across tenants?
5. **A2A vs MCP for node-to-node calls inside Caliban:** use the internal fast path only and expose the protocols at the edge?
6. **Memory governance:** right-to-erasure across derived memories and cached plans; how should PII anonymization interact with memory recall quality?
7. **Topology selection:** can the intent router reliably predict when multi-agent fan-out helps ([2512.08296](https://arxiv.org/abs/2512.08296), [2609.19759](https://arxiv.org/abs/2609.19759)), or should fan-out be author-declared only?
8. **Billing model** for nested nodes and marketplace nodes running in the owner's tenant.
9. **MCP Tasks** is still experimental in 2025-11-25. Track the next spec revision before relying on it for long-running nodes.
