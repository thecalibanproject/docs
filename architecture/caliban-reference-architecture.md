# Caliban Reference Architecture (v0.1)

*Draft, 2026-10-02. Built from the research notes in [`../research/`](../research/). Each decision links to the note that supports it, and the notes cite the original papers.*

## 1. What Caliban is

Caliban is the **single AI endpoint for a company**. A customer points its apps at Caliban instead of at OpenAI or Anthropic, connects its datasources once, and builds AI sub-apps ("nodes") on top. Caliban provides six capabilities:

| # | Capability | One-line promise | Research |
|---|---|---|---|
| 1 | **Ontology layer** | Connect any datasource. AI answers are grounded in a governed business model of your data, not raw schemas. | [03](../research/03-ontology-and-datasource-layer.md) |
| 2 | **Intent classification & routing** | Every request goes to the cheapest model, agent and datasource set that meets the quality floor. | [02](../research/02-intent-classification-and-routing.md) |
| 3 | **Agents ("nodes")** | Customers compose versioned, budgeted and secure AI sub-apps. Each one is callable over HTTP, MCP and A2A. | [04](../research/04-agent-orchestration.md) |
| 4 | **Anonymization** | Personal data is pseudonymized before it leaves the trust boundary and restored in the response, including streamed responses. | [05](../research/05-anonymization-and-privacy.md) |
| 5 | **Fewer tokens, faster answers** | Provider prompt-cache steering, tenant-scoped exact and semantic caching, RAG plus compression, and plan caching. | [01](../research/01-semantic-caching-and-rust-vector-stack.md), [06](../research/06-rag-and-token-efficiency.md) |
| 6 | **Sovereign router** | The same Rust binary runs as SaaS, in a customer VPC, or air-gapped with on-prem open-weight models. | [07](../research/07-gateway-serving-and-sovereign-deployment.md) |

### Where Caliban wins

Speed alone will not differentiate Caliban. Two Rust gateways already exist: Helicone claims under 5 ms P95 and TensorZero under 1 ms P99. Hosted routers (OpenRouter, Cloudflare) cannot run on-prem. Python/TS gateways (LiteLLM, Portkey) keep governance behind paywalls, and LiteLLM's PyPI package was compromised in March 2026. Infra proxies (Agent Router, formerly Envoy AI Gateway; Kong; Plano) target platform teams. **No product combines all four of the following** ([07](../research/07-gateway-serving-and-sovereign-deployment.md#competitive-landscape)):

1. reversible PII pseudonymization that works on streamed output,
2. routing and grounding that understand the *tenant's own data* (ontology),
3. a real multi-tenant control plane with nodes and budgets,
4. one signed binary for SaaS, VPC and air-gap.

That combination is the product. Each part on its own is a feature that a competitor can copy.

---

## 2. Evidence that should shape every design decision

These are the research findings that change what we build. Each is checked against its source paper.

| Finding | Consequence for Caliban |
|---|---|
| Text-to-SQL on raw enterprise schemas scores about 10–16% (BEAVER, EntSQL, data.world). The same questions over an ontology or KG score 54–72%. Within a governed semantic layer, dbt measured 98–100% (vs 84–90% for text-to-SQL on the same modeled schema, but only 11 questions). | The **ontology is the core asset.** Never run free-form SQL over raw schemas in production. |
| The LLM picks from a bounded menu (metric, dimension, filter, approved join) and deterministic code compiles the query: 97.4% vs 55.3% (SPC, small eval). | The *compiler* enforces permissions, never the prompt. |
| Simple routers match complex ones. Most of the routing gain comes from the **task type** ("Most of the LLM Routing Gap Is Task Type", 2026). | Start routing with intent → model tables plus embedding kNN. Add learned routing later. |
| Static-threshold semantic caching (GPTCache-style) gives too many false hits. vCache learns a threshold per entry and reports up to 12.5× more hits and 26× fewer errors. | Semantic cache only on allowlisted intents, with per-entry learned thresholds and an error budget. |
| Gateways that pool provider keys break the provider's per-org prompt-cache isolation. All 5 OSS gateways tested leaked. On OpenRouter, 12 of 28 tested model labels (33.7% of volume) showed cross-account reads (KeyPooling, 2026). | **Each tenant gets its own upstream identity** (key, project or workspace) or a tenant-HMAC `prompt_cache_key`. Never share a semantic cache across tenants. |
| Provider prompt-cache reads cost 0.05–0.1× the normal input price. | The biggest saving per unit of engineering is **steering provider caches** (stable prefix ordering), not building our own cache. Compression must never rewrite the cached prefix. |
| Presidio alone misses most partial or obfuscated PII. Placeholders like `[PERSON_1]` hurt answer quality. Type-consistent surrogates recover about 13 pts of BERTScore (SurrogateShield). | Tiered detector (patterns → small NER → optional local LLM). **Realistic surrogates**, not placeholders. |
| Embeddings can be inverted back to text (vec2text: 92% exact recovery on 32-token inputs). Noise defenses fail. | Treat cached vectors as personal data. Embed only pseudonymized text, keep per-tenant collections and keys, and crypto-shred on deletion. |
| Free-form multi-agent systems fail 41–87% of the time (MAST). Plan-then-execute reports up to 6.7× lower cost and 3.7× lower latency, and gives control-flow integrity against prompt injection. | Nodes are **typed, budgeted graphs**, not agent chat rooms. Security is enforced in the Rust core. |
| Tool-description poisoning (MCPTox) succeeds over 72% of the time, and published prompt-level defenses fall to adaptive attacks. | Pinned and hashed tool manifests, taint tracking, and per-call down-scoped tokens. Treat every MCP server as untrusted. |
| Adding retrieved context makes quality rise and then fall. Benchmark compression gains (LLMLingua-2: 2–20×) lose details in practice. PoisonedRAG reaches 97% attack success with 5 injected texts. | Use a token budget with a rerank-score cutoff, not "stuff the window". Compression is off by default. Key every cached item to document versions. Ingestion is a security boundary. |
| BIRD and Spider 2.0 have 50–60% annotation errors in audited samples. | Build **per-tenant golden sets** inside the product. Public leaderboards are not a target. |

---

## 3. System overview

```
                          ┌──────────────────────── CONTROL PLANE (Rust, Postgres) ────────────────────────┐
                          │ tenants/keys/BYOK vault · policies · model registry (licence, region, price)   │
                          │ ontology studio (CSM) · connectors · node registry · evals · billing · audit   │
                          └───────────────┬──────────────── signed, versioned config snapshots ────────────┘
                                          │ (xDS-style push / file / embedded; DP is fail-static)
 Client apps ──HTTPS──▶ ┌─────────────────▼──────────── DATA PLANE: caliban-router (single Rust binary) ─────────────┐
 OpenAI SDK             │  1 auth → 2 tenant policy → 3 quota reserve → 4 parse to CalibanIR                         │
 Anthropic SDK          │  5 ANONYMIZE ──▶ 6 CACHE (T1 exact → T2 semantic) ──hit──────────────────────┐             │
 MCP / A2A clients      │  7 CLASSIFY + ROUTE (intent, lane, model, node) ──▶ 8 GROUND (ontology/RAG)  │             │
                        │  9 EXECUTE: single call | node graph (durable journal, budgets, taint)       │             │
                        │ 10 PROVIDER ADAPTER (per-tenant upstream identity, prefix steering)          │             │
                        │ 11 STREAM ◀──────────────────────────────────────────────────────────────────┘             │
                        │ 12 REHYDRATE (streaming hold-back) → 13 METER → 14 TRACE (OTel GenAI)                     │
                        └───────┬─────────────┬──────────────┬──────────────┬────────────────┬──────────────────────┘
                                │             │              │              │                │
                         Valkey/Redis    Qdrant/usearch   DataFusion     PII vault       Providers:
                         (quotas)        (cache, RAG,     federation     (per-tenant      OpenAI, Anthropic, …
                                          routing kNN)    → customer DBs  envelope keys)  or on-prem vLLM/SGLang
```

**Rule:** the data plane holds no authoritative state. It runs on a signed config snapshot and keeps serving the last good one if the control plane is unavailable ([07 §1](../research/07-gateway-serving-and-sovereign-deployment.md#1-control-plane--data-plane-split)).

---

## 4. Request lifecycle (data plane)

Each stage is a `tower::Layer`, so each deployment SKU composes its own stack from config. Latency figures are **design targets to benchmark, not measurements.**

| # | Stage | What happens | Key crates / tech | Target |
|---|---|---|---|---|
| 1 | Auth | API key, JWT or mTLS → `tenant_id`, principal, scopes | axum/hyper, in-memory key index | <0.2 ms |
| 2 | Tenant policy | Allowed models, regions and licences; ZDR; PII mode; budgets | Compiled decision table from the snapshot | <0.1 ms |
| 3 | Quota | GCRA request rate plus a token-budget *reservation* (local tokenizer estimate) | `governor`, Valkey Lua, `tokenizers` | <1 ms |
| 4 | Parse → IR | OpenAI Chat/Responses or Anthropic Messages → `CalibanIR`, parsed once, zero-copy `Bytes` | serde, simd-json | <0.5 ms |
| 5 | **Anonymize** | L0 regex, validators and tenant dictionaries from the ontology → L1 GLiNER2-PII via ONNX → (L2 local LLM, strict mode only) → vault surrogates | `regex`, `aho-corasick`, `ort`, `fpe` (FF1) | L0 <1 ms, L1 5–30 ms/1k tok |
| 6 | **Cache** | T1 exact (BLAKE3 key incl. ACL fingerprint and datasource epochs) → T2 semantic (allowlisted intents, per-entry threshold, slot equality) | moka, foyer, usearch, Qdrant, fastembed-rs | T1 <5 ms, T2 <15 ms |
| 7 | **Classify + route** | Rules → embedding kNN → ModernBERT multi-head (intent, difficulty, OOS, jailbreak, PII-presence) → Arch-Router-class LLM fallback for ≤5–10% of traffic → pick a model within the route | `ort`/`candle`, shared embedding | p50 <10 ms without LLM fallback |
| 8 | **Ground** | Ontology retrieval (filtered by policy *before* retrieval) → lane A (metric compiler), B (validated SQL) or C (doc RAG) | DataFusion, sqlparser-rs, tantivy, Qdrant | Depends on the source |
| 9 | Execute | A single model call, or a node graph run by the durable executor | tokio, rmcp, Restate or a Postgres journal | — |
| 10 | Provider adapter | IR → wire format. Per-tenant credentials. Canonical prefix ordering plus `cache_control` / `prompt_cache_key`. Retry budget and circuit breaker. No retry after the first byte. | reqwest/hyper HTTP/2 pools | — |
| 11 | Stream | SSE passthrough with backpressure, translated to the client's dialect | bounded channels | — |
| 12 | **Rehydrate** | Aho-Corasick over active surrogates. Hold-back buffer of (longest surrogate − 1) characters. Fuzzy fallback. Miss telemetry. | `aho-corasick` | A few tokens of delay |
| 13 | Meter | Settle the reservation against actual usage (incl. cached and reasoning tokens). Idempotent UsageEvent → WAL → bus. | NATS/Kafka → ClickHouse | async |
| 14 | Trace | OTel GenAI span tree. Content capture opt-in only, and never in ZDR mode. | `opentelemetry` | async |

**Gateway overhead budget (excluding models, classifiers and RAG): p50 < 3 ms, p99 < 10 ms.**

**The one-embedding rule:** compute one embedding per request (of the *pseudonymized* text) and reuse it for semantic cache lookup, routing kNN, RAG retrieval and drift monitoring.

---

## 5. Subsystem designs (summaries; full detail in the research notes)

### 5.1 Ontology layer: the Caliban Semantic Model (CSM)

- **Connectors.** One Rust `Connector` trait. Every source is exposed as Arrow `RecordBatch` streams behind a DataFusion `TableProvider`.
  - Five tiers: SQL/warehouses, NoSQL/graph, APIs/SaaS (OpenAPI ops → table functions), files/lakes, and MCP servers. MCP servers are an **untrusted** tier.
  - Each connector exposes `introspect`, `profile`, `query_log` and `change_feed`. The change feed drives cache invalidation.
- **Model.** A typed, versioned property graph stored in Postgres and serialized as YAML for review. Object types: `Entity`, `Attribute`, `Relation` (with cardinality), `Metric`, `Dimension`, `GlossaryTerm`, `Policy` (row predicates, column masks, PII class) and `VerifiedQuery`.
  - Import/export: MetricFlow/OSI, Cube, dbt.
  - Do not use RDF/OWL internally. Borrow Ontop's *mapping* idea, not its stack.
- **Bootstrap.** Introspect, then profile, then mine query logs, then an LLM proposes entities, synonyms, PII classes and implicit joins, then humans curate.
  - Relations, metrics and policies **always** need human approval.
  - The curation queue is sorted by query frequency × uncertainty.
- **Query lanes.**
  - **A:** the LLM fills a JSON plan constrained to retrieved IDs, and a deterministic compiler builds the query.
  - **B:** exploratory SQL over *logical* entity views, then parse, bind to the CSM, apply a policy rewrite (DataFusion optimizer rule), and dry-run.
  - **C:** document RAG, with entities linked into the CSM.
  - If the ontology has no match, ask a clarifying question or say "I don't know".
- **Feeds the rest of Caliban.**
  - Ontology coverage drives routing (lane A → small model, lane B → reasoning model).
  - PII classes drive anonymization dictionaries and model-tier restrictions.
  - Source epochs and ontology versions drive cache invalidation.
- **Build vs buy.** Spice.ai OSS (Rust, DataFusion federation + acceleration + MCP + text-to-SQL) and WrenAI (DataFusion) are the closest prior art. **Decide whether to fork, embed or compete before writing connector code.**

### 5.2 Intent classification & routing

- **What a route is.** A route = (intent, node set, datasources, model, reasoning effort). Tenants declare routes and constraints, and Caliban learns which concrete model serves each route best.
- **Pipeline.**
  - **Stage 0:** compiled rules.
  - **Stage 1:** kNN over per-tenant exemplars in the shared vector index.
  - **Stage 2:** ModernBERT/SetFit multi-head in-process.
  - **Stage 3:** LLM fallback under a hard timeout.
  - **Stage 4:** score = quality − λ·cost − λ·latency, with LinUCB exploration inside a budget.
- **BYO models.** Run a probe set of 1–2k prompts stratified by intent cluster to get a quality vector per cluster (UniRoute). The model is routable immediately, without retraining.
- **Learning loop.** Log propensities on every decision. Retrain offline every night and promote only when off-policy evaluation beats both the current router *and* the "one model per intent" baseline.
- **Security.** Routers can be gamed with adversarial suffixes. Mitigations: spend caps per tenant and route, audit sampling, and alarms when traffic shifts toward the top-tier model.
- **Privacy.** The cost function includes a **privacy penalty**. Sensitive requests prefer sovereign or local models when a capable one exists (PAPILLON-style).

### 5.3 Nodes (agents & sub-apps)

- **A node is declarative, versioned config:** prompt, output schema, model policy (handed to the router), pinned tools (`mcp://…#sha256`), datasource scopes, memory policy, guardrails, budgets, exposure and eval suite. See the YAML in [04 §1](../research/04-agent-orchestration.md#1-the-node-model-a-node-is-declarative-config-not-code).
- **Graph vertex kinds:** `llm`, `tool`, `router`, `map`, `reduce`, `verify`, `human`, `subnode`, and `code` (sandboxed in WASM via wasmtime). Loops must declare a maximum iteration count.
- **Day-one templates:** ReAct (bounded), plan-and-execute (ReWOO/LLMCompiler), dual-LLM and CaMeL-secure. Plan caching is keyed by intent, tool-schema hashes and scope.
- **Execution.** An event-sourced journal per run. Idempotency keys on write tools. A hierarchical budget ledger (tenant → node → run → subnode). Human-in-the-loop interrupts map 1:1 to A2A task states and MCP Tasks.
- **Exposure.** Every node is an MCP tool (via `rmcp`) and an A2A agent. Caliban is also the MCP *client* and the single policy enforcement point toward customer tools.
- **Security.**
  - Effective permissions = node grants ∩ invoker grants ∩ tenant policy.
  - Downstream tokens are minted per call and down-scoped; client tokens are never passed through.
  - Taint labels: no writes with tainted arguments unless allowlisted.
  - Marketplace nodes run in their owner's tenant.
- **Evaluation.** pass^k, cost per success, MAST failure labels from traces, and an AgentDojo-style injection suite in CI for every node with write access.

### 5.4 Anonymization

- **Trust tiers decide the treatment:**

  | Tier | Destination | Treatment |
  |---|---|---|
  | T0 | Sovereign or local model | No redaction needed (logs are still scrubbed) |
  | T1 | Attested TEE | Same as T0 if attestation verifies |
  | T2 | Contracted provider with zero data retention | Pseudonymize direct identifiers |
  | T3 | Unknown | Strict: generalize quasi-identifiers, run the L2 verifier, fail closed |

- **Surrogates.**
  - Names, organizations and locations: realistic values from per-locale pools.
  - Structured IDs: FF1 format-preserving encryption, with checksums such as Luhn recomputed.
  - Dates: a consistent date shift.
  - Coreference resolution runs before substitution.
- **Scopes.** Surrogates are consistent within a request, session (default) or tenant. Tenant scope is required for semantic-cache hits and costs cross-session linkability, so it is opt-in.
- **Vault.** An encrypted reverse map with a per-tenant KEK (customer-held in the sovereign tier) and per-scope DEKs. Deleting the key crypto-shreds the data, which serves GDPR erasure.
- **Tool calls.** Arguments are rehydrated *inside* the trust boundary before they reach customer systems. Tool results are re-pseudonymized before they go back to an external model.
- **Marketing language.** Say "pseudonymization", not "anonymization". Under the EDPB's 2025 guidelines, pseudonymized data is still personal data.

### 5.5 Token & latency efficiency (ordered by return on effort)

1. **Provider prompt-cache steering.** Canonical order: tools → system → ontology/context modules → history → user. Automatic `cache_control` breakpoints and a tenant-HMAC `prompt_cache_key`.
2. **Routing.** Cheap models for easy intents. Published savings range from 40% to 98% at roughly equal quality, depending on the workload.
3. **T1 exact cache**, invalidated by datasource epochs.
4. **Ontology lane A.** Small models make bounded choices instead of large models writing SQL.
5. **Plan cache** for nodes (Agentic Plan Caching: −50% cost, −27% latency).
6. **T2 semantic cache** on allowlisted intents.
7. **RAG hygiene** ([06](../research/06-rag-and-token-efficiency.md)):
   - **Chunking and indexing.** Contextual chunks of 256–512 tokens, split recursively; semantic chunking isn't worth its cost. Index with tantivy BM25 plus vectors, fused with RRF, then a cross-encoder rerank via `ort`.
   - **Retrieval mode is a router head.** The router chooses none, single-shot, iterative or long-context per request (Adaptive-RAG, Self-Route).
   - **Token budget manager.** Stop adding chunks at the reranker score cliff and reorder them against "lost in the middle". More context often *lowers* quality.
   - **Prompt compression is OFF by default.** If enabled, it is extractive only, applies to long retrieved context only, and never touches the cached prefix. Abstractive compression wrecks citation fidelity.
   - **Check licences.** Provence and Jina-ColBERT-v2 weights are CC BY-NC.
8. **On-prem only:** KV/prefix reuse in SGLang or vLLM with per-tenant salts.

Every response reports `x-caliban-tokens-saved`, `x-caliban-cache` and `x-caliban-routed-model`, and the dashboard shows the dollars saved. **This proves Caliban's value to customers.**

### 5.6 Multi-tenancy and isolation (non-negotiable)

- `tenant_id` appears in every state key, cache namespace, vector collection, log partition and metric label.
- **Upstream identity per tenant**: BYOK, a provider project/workspace per tenant, or at minimum a tenant-HMAC cache key on pooled free-tier keys, with that caveat documented.
- No cross-tenant semantic or KV cache. The only exception is Caliban-authored public content.
- Compute tiers: a shared pool with VTC fair queueing, dedicated model pools, or dedicated data-plane cells. SaaS is sharded into regional cells.
- **CI isolation audit:** two synthetic tenants run timing and `cached_tokens` probes on every release and every new upstream route. The p-values are published in a trust report.

---

## 6. Deployment SKUs: one binary, three shapes

| SKU | Data plane | Control plane | Models | Connectivity |
|---|---|---|---|---|
| **SaaS** | Caliban-run regional cells (EU, US) | Caliban-hosted | Third-party APIs + Caliban-hosted open models | Internet |
| **VPC / dedicated** | Customer K8s via Helm, single tenant | Caliban-hosted (outbound-only link) or self-hosted | BYOK + customer GPUs | Private link |
| **Sovereign / air-gapped** | Customer on-prem | **Embedded** (`--mode=standalone`, local Postgres) | Open-weight only, signed | None. Signed offline bundles. |

- **Swappable traits:** `ConfigSource`, `UsageSink`, `SecretStore` and `ModelProvider` are traits selected by build features and runtime config. A FIPS build uses `aws-lc-rs`.
- **On-prem serving.** Caliban does **not** build an inference engine.
  - Engines: vLLM by default, and SGLang for agent and RAG pools with heavy prefix reuse.
  - Scheduling: llm-d or NVIDIA Dynamo.
  - Per-tenant fine-tunes are LoRA adapters loaded only from a signed registry. Runtime adapter loading stays off on shared pools.
- **Starter models (licences safe for EU use).** gpt-oss-120b (fits one 80 GB GPU), gpt-oss-20b or Qwen3 for cheap tiers, the Mistral Apache-licensed line, and DeepSeek-V4 for customers with multi-node GPU budgets. **Llama 4 is blocked by default for EU tenants** because its licence excludes EU-domiciled entities.
- **Supply chain.** Reproducible builds, `cargo vet`/`cargo deny`, vendored crates, signed SBOMs and a CycloneDX ML-BOM, and Sigstore/OMS-signed model weights.

---

## 7. Rust workspace layout (proposed)

```
caliban/
  crates/
    caliban-ir          # canonical request/response IR + OpenAI/Anthropic codecs
    caliban-router      # binary: axum + tower layers (stages 1–14)
    caliban-policy      # policy DSL compiler → decision tables
    caliban-pii         # L0/L1 detectors, surrogate engine, vault client, streaming rehydrator
    caliban-cache       # T1/T2/plan caches, epochs, ACL fingerprints
    caliban-route       # encoders, kNN, ONNX heads, scorer, bandit, decision log
    caliban-ontology    # CSM types, retrieval cards, lane-A compiler, SQL validator
    caliban-connect     # Connector trait + DataFusion table providers + federation
    caliban-rag         # chunking, hybrid index (tantivy + Qdrant), rerank, compression
    caliban-nodes       # graph IR, executor, journal, budgets, taint, WASM sandbox
    caliban-mcp         # rmcp server/client, manifest pinning, token minting; A2A
    caliban-providers   # provider adapters, prefix steering, health/EWMA
    caliban-meter       # reservations, usage events, WAL
    caliban-cp          # control plane service (embeddable for standalone mode)
  sdk/ (python, typescript)   # node authoring, evals, offline optimizers, router training
  models/                     # ONNX artifacts: embedder, intent heads, PII NER (versioned, signed)
```

Training (router heads, PII fine-tunes, DSPy-style optimizers) stays in Python. **The contract between Python and Rust is ONNX plus JSON profile tables.**

---

## 8. Phased roadmap

| Phase | Goal | Scope | Exit criteria |
|---|---|---|---|
| **P0: Gateway MVP** | "Point your OpenAI SDK at us" | Stages 1–4, 10–14. OpenAI + Anthropic APIs. Per-tenant upstream identity. Prefix steering. T1 exact cache. Metering. OTel. | Under 3 ms p50 overhead. Isolation audit passes. Usage matches provider bills within 1%. |
| **P1: Privacy + routing** | "Cheaper and safe by default" | L0+L1 PII with surrogates and streaming rehydration. Routing stages 0, 1 and 4 with intent → model tables. BYO-model probe onboarding. Savings dashboard. | Rehydration miss rate under 0.5%. Measured cost savings at a fixed quality floor on design-partner traffic. |
| **P2: Ontology** | "Talk to your data, correctly" | Connector trait for 5–8 sources. CSM, bootstrap and curation UI. Lanes A and B with the validator and policy rewrite. Epoch invalidation. Per-tenant golden sets. | Golden-set accuracy above a target agreed with each design partner. Zero policy bypasses in the red-team suite. |
| **P3: Nodes** | "Build AI sub-apps" | Node config, graph executor, journal, budgets. MCP/A2A exposure. Templates. Plan cache. Eval loop. | pass^k and cost-per-success reported per node. AgentDojo suite runs in CI. |
| **P4: Sovereign** | "Run it all on-prem" | Standalone CP. Air-gap bundle. vLLM/SGLang + llm-d integration. Signed weights. LoRA registry. ModernBERT router heads. T2 semantic cache with vCache thresholds. | One customer running fully offline with no egress. |

Rationale for this order:
- P0 and P1 are what customers pay for first: lower cost and data protection.
- P2 is the moat.
- P3 depends on P2's scopes and policies.
- P4 is mostly packaging if the trait boundaries are right from P0. **Design the traits for all three SKUs on day one.**

---

## 9. Decisions

**Made (2026-10-02):**
- **BYOK only.** Each tenant brings its own provider keys or local endpoints. Upstream credentials are never pooled, which also removes the cross-tenant prompt-cache leak by construction.
- **100% on-prem capable.** All components, including the control plane, web console and models, run with zero egress. SaaS is the same software, operated by us.
- **MongoDB stays the customer's system of record; Caliban is the analytics engine** ([08](../research/08-ontology-deep-dive-nosql.md)):
  - The LLM emits **CQIR** (typed JSON naming ontology ids only).
  - A Rust compiler lowers CQIR to either a linted, time-boxed aggregation pipeline on a read-only secondary (native lane) or SQL over a change-stream-fed Parquet replica in DataFusion (accelerated lane).
  - Free-form text-to-MQL is never executed.
  - Implemented and tested in `core/crates/caliban-ontology`.
- **Ontology store = Postgres** (append-only, versioned, gated by golden-set evals), compiled into an in-memory graph in the data-plane snapshot. Oxigraph is used only for RDF/OWL import/export.

**Made (2026-10-09):**
- **Spice.ai / WrenAI: compete.** Caliban builds its own semantic layer, CQIR compiler and change-stream replica; it does not fork or embed either project. Their public designs are reference material only.
- **Node journal: a minimal Postgres journal, built in `caliban-nodes`.** It follows the Absurd model: one SQL schema, steps checkpointed as rows, workers claiming runs with `SELECT ... FOR UPDATE SKIP LOCKED`, durable sleeps and awaited events. Reasons:
  - Postgres is already the control-plane store, so nodes add no new stateful service to an on-prem or air-gapped install.
  - The Restate server is BSL 1.1. Its additional use grant forbids a "Public Restate Platform Service" and does not clearly cover a product that customers run on their own hardware, where their services register endpoints; that is legal risk for a product whose core promise is on-prem redistribution.
  - The Rust Postgres libraries are too young to depend on: DBOS Transact for Rust is early and documents at-least-once side effects; tensorzero/durable was archived in June 2026.
  - Side effects (LLM calls, tool calls) carry an idempotency key derived from (run id, step id), so a replayed step never repeats a charged call or an external write.
- **Pricing: flat price for `caliban/auto`.** Routing quality and cost are Caliban's margin risk, so metering records the routed model's real cost next to the flat price per request, and the router enforces a quality floor per intent before choosing the cheapest model.
- **Surrogate scope default: per tenant.** The same value in the same tenant always gets the same surrogate (keyed HMAC per tenant), so pseudonymised requests can hit the cache. The cost is that sessions within a tenant become linkable through their surrogates; session scope stays available as a per-tenant opt-in. Surrogates never cross tenants.
- **Cache hits on `caliban/auto` are billed at a discounted flat price.** No model is called, so the customer is charged a configurable fraction of the flat price (in the 10 to 25% range), and metering records the hit next to the full price so the saving is visible in usage.
- **SSO and roles come before P3.** Node permissions (effective permissions = node grants ∩ invoker grants ∩ tenant policy) build on real identities and roles, so OIDC login and RBAC replace the bootstrap admin token before the node executor is built.

**Still open:**

1. ~~Spice.ai / WrenAI~~: decided, compete.
2. ~~Journal backend for nodes~~: decided, Postgres journal.
3. ~~Upstream identity model~~: decided, BYOK only.
4. ~~Pricing model~~: decided, flat price for `caliban/auto`.
5. **Canonical metric format:** MetricFlow/OSI (Apache Ossie, incubating) as the canonical format, or CSM as a superset of it.
6. ~~Surrogate scope default~~: decided, per tenant.

## 10. Known research gaps

- ~~Text-to-NoSQL~~: covered in [08](../research/08-ontology-deep-dive-nosql.md). Text-to-API is still thin.
- No survey yet of multimodal (image, audio, OCR) PII handling.
- Several 2026 regulatory facts come from law-firm blogs rather than the Official Journal: Digital Omnibus details, DeepSeek-V4 licence, TGI archive date. Have counsel verify them before using them in sales material.
- All latency targets in this doc are engineering estimates. Benchmark them in P0 and P1.
