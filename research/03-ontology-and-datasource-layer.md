# Ontology & Datasource Layer: Letting LLMs Query Any Source Correctly

*Research date: 2026-10-02. Scope: OBDA/virtual knowledge graphs, LLM-built ontologies, KG+LLM, GraphRAG/LightRAG, text-to-SQL/NoSQL/Cypher/API, semantic layers, catalogs, MCP, Arrow/DataFusion federation.*

**Summary.** Three findings from 2023–2026 should drive Caliban's design. First, the model is not the main bottleneck on enterprise data. **Context is.** On real enterprise schemas, raw text-to-SQL is still poor: about 10–11% on BEAVER, about 16% on EntSQL when the needed business rules sit in documents, and 16% zero-shot on data.world's insurance benchmark. The same model jumps to 54–72% when it queries an ontology or knowledge graph instead ([Sequeda 2023](https://arxiv.org/abs/2311.07509), [Allemang 2024](https://arxiv.org/abs/2405.11706)), and to 98–100% when the question is in scope of a governed semantic layer ([dbt 2026](https://docs.getdbt.com/blog/semantic-layer-vs-text-to-sql-2026)). Second, the most reliable design (2026 work: SPC, GROUND) has the LLM make **bounded choices over a curated semantic graph** (which metric, which dimension, which filter, which approved join path). **Deterministic code** then compiles, secures and validates the query. Permissions are enforced in the compiler, not in the prompt. Third, academic leaderboards (BIRD ~82%, Spider 2.0-Snow >90% on vendor agents) overstate production readiness: 50–60% of the samples audited in BIRD Mini-Dev and Spider 2.0-Snow have annotation errors ([Jin et al. 2026](https://arxiv.org/abs/2601.08778)). For Caliban this means: build a typed, versioned ontology (entities, relations, metrics, synonyms, policies). Bootstrap it with LLMs from schemas, profiles and query logs, then have humans curate it. Index it for hybrid retrieval and compile through a Rust/DataFusion federation engine. Treat every MCP server and every data value as untrusted input.

---

## Key papers

### A. Text-to-SQL methods (what works on benchmarks)

| Paper | Year | Core idea | Why it matters for Caliban | Verdict |
|---|---|---|---|---|
| [DIN-SQL](https://arxiv.org/abs/2304.11015) | 2023 | Splits generation into schema linking → classify/decompose → generate → self-correct; about +10 pts over plain prompting on Spider. | The basic pipeline shape. Self-correction is cheap and generic. | Adopt (pattern) |
| [DAIL-SQL](https://arxiv.org/abs/2308.15363) | 2023 | Systematic study of question representation and few-shot example selection by question+SQL skeleton similarity; 86.6% EX on Spider. | Retrieving *verified examples* by structural similarity is the cheapest accuracy gain. Caliban's "verified queries" store should work this way. | Adopt |
| [MAC-SQL](https://arxiv.org/abs/2312.11242) | 2023 | Selector (prunes schema) / Decomposer / Refiner agents; 59.6% BIRD (GPT-4). | Early proof that schema pruning matters on "huge" DBs. | Watch (superseded) |
| [CHESS](https://arxiv.org/abs/2405.16755) | 2024 | IR agent (LSH value retrieval + description retrieval), schema selector, generator, LLM unit tester; 71.1% BIRD with ~83% fewer LLM calls. | **Value retrieval** (matching literals like "Acme" to column values) is essential and maps directly to a vector/LSH index in Caliban's data plane. | Adopt (IR + value index) |
| [The Death of Schema Linking?](https://arxiv.org/abs/2408.07702) | 2024 | With strong long-context LLMs, passing the *full* schema beats aggressive pruning when it fits; 71.83% BIRD. | Prune for cost, not for accuracy. Prune with high recall, since a dropped column cannot be recovered. | Adopt (high-recall pruning) |
| [CHASE-SQL](https://arxiv.org/abs/2410.01943) | 2024 | Several diverse generators (divide-and-conquer, query-plan CoT, synthetic examples) plus a fine-tuned pairwise selector; 73.0% BIRD test. | Generating several candidates and selecting one is the SOTA recipe, but it costs N× tokens. Use it only for high-value, uncached queries. | Prototype |
| [XiYan-SQL](https://arxiv.org/abs/2507.04701) ([preview](https://arxiv.org/abs/2411.08599)) | 2024–25 | Multi-generator ensemble + selection; **M-Schema** compact schema serialization; 75.63% BIRD, 89.65% Spider, also NL2GQL. | M-Schema is a good template for how Caliban serializes ontology slices into prompts. | Adopt (serialization) |
| [ReFoRCE](https://arxiv.org/abs/2502.00675) | 2025 | Schema compression by pattern grouping, self-refinement, majority-vote consensus, execution-driven column exploration; ~36% on Spider 2.0-Snow/Lite. | Schema compression (grouping `events_2024_01..12`-style tables) and *probe queries* are needed for warehouse-scale schemas. | Adopt (compression, probes) |
| [Agentar-Scale-SQL](https://arxiv.org/pdf/2509.24403) | 2025 | Orchestrated test-time scaling; 81.67% BIRD test (#1 at release). | Shows the ceiling of "throw compute at it". Human performance is ~93%. | Watch |
| [GATE: Bootstrapping Semantic Layer from Execution](https://arxiv.org/abs/2606.05634) | 2026 | Keeps grounding hypotheses open, executes the grounded parts, and stores the hypothesis confirmed by results as reusable memory. | Directly matches Caliban's "ontology learns from usage" loop. | Prototype |

### B. Enterprise reality checks (benchmarks)

| Paper | Year | Core idea | Why it matters for Caliban | Verdict |
|---|---|---|---|---|
| [Spider 2.0](https://arxiv.org/abs/2411.07763) | 2024 | 632 real enterprise workflow tasks on BigQuery/Snowflake, often >1,000 columns, multiple dialects; o1-preview agent solved 21.3% (vs 91.2% Spider 1, 73% BIRD). | The canonical "enterprise is different" result: dialects, huge schemas, nested data. | Adopt (as eval) |
| [BEAVER](https://arxiv.org/abs/2409.02038) | 2024–26 | Real private data warehouses (avg ~101 tables / ~869 columns in latest version). Near-zero EX initially; GPT-5.2 agents ~10.8%, ReFoRCE + Claude 4.5 Sonnet 11.4%. | Private warehouses with idiosyncratic naming defeat models trained on public data. Caliban's value is the curated context. | Adopt (as eval) |
| [EntSQL](https://arxiv.org/abs/2606.03363) | 2026 | 1,066 questions whose SQL depends on long enterprise documents (metrics, reporting conventions); best system 15.9% (EN). | Dumping wikis or PDFs into context does not work. Business rules must be extracted into structured metrics and policies. | Adopt (insight) |
| [Spider 2.0-AIFunc](https://arxiv.org/abs/2607.06229) | 2026 | Tasks that need in-warehouse AI SQL functions; proprietary 67–70%, open 58.1%; elaborate agents underperform simple setups. | Caliban should expose LLM UDFs inside the federated engine (DataFusion UDFs) and keep agents simple. | Watch |
| [Pervasive Annotation Errors Break Text-to-SQL Benchmarks](https://arxiv.org/abs/2601.08778) | 2026 | Audited samples: 52.8% annotation errors in BIRD Mini-Dev, 62.8% in Spider 2.0-Snow; leaderboard ranks shift up to 9 places. | Do not choose components from leaderboards. Build a private, per-customer golden set. Vendor claims such as [96.7 on Spider 2.0-Snow](https://genloop.ai/blogs/genloop-is-1-on-spider-2.0) need skepticism. | Adopt (eval hygiene) |
| [Text-to-SQL for Enterprise Data Analytics (LinkedIn)](https://arxiv.org/abs/2507.14372) | 2025 | Production chatbot: KG of table metadata + query logs, agent with error correction; 300+ WAU, 53% correct/near-correct. | The most honest production number available. Query logs plus a metadata KG are the high-value context. | Adopt (pattern) |
| [Beyond the Harness](https://arxiv.org/abs/2608.22830) | 2026 | Optimize the *context artifacts* (SQL "reference cards" mined via query-DAG decomposition of historical queries) rather than the retrieval harness; 12–25% AST-similarity gains vs 3–12%. | Mine customer query history into reusable, verified snippets. | Prototype |

### C. Semantic layers / ontologies as grounding

| Paper | Year | Core idea | Why it matters for Caliban | Verdict |
|---|---|---|---|---|
| [Sequeda, Allemang, Jacob: KG benchmark (data.world)](https://arxiv.org/abs/2311.07509) | 2023 | Insurance schema + ontology + R2RML mappings; GPT-4 zero-shot 16% on SQL vs 54% on the KG representation. | The founding result for "ontology > raw schema". | Adopt |
| [Ontologies to the Rescue! (OBQC + LLM repair)](https://arxiv.org/abs/2405.11706) | 2024 | Validate generated SPARQL against ontology constraints (domain/range etc.), explain errors, let the LLM repair; 72% accuracy, 8% "I don't know". | **Ontology-based query checking** is a cheap, deterministic validator. Port it to Caliban's plan validator. | Adopt |
| [dbt: Semantic Layer vs Text-to-SQL 2026](https://docs.getdbt.com/blog/semantic-layer-vs-text-to-sql-2026) | 2026 | Same ACME benchmark, 11 Qs × 20 runs: Sonnet 4.6 90.0% → 98.2%, GPT-5.3-Codex 84.1% → 100% via semantic layer. The SL errors out-of-scope; text-to-SQL is *silently wrong*. | Use two lanes: governed metric compile when in scope, flagged exploratory SQL otherwise. Vendor blog, small N. | Adopt |
| [Semantic Layers for Reliable LLM Analytics (paired benchmark)](https://arxiv.org/abs/2604.25149) | 2026 | A 4 KB semantic doc adds +17–23 pts across Opus 4.7 / Sonnet 4.6 / GPT-5.4 (45–50% → 68–69%). The semantic layer explains more variance than the model choice. | Supports investing in the ontology over model routing for data questions. | Adopt |
| [Bounded Semantic Planning + Deterministic Compilation (SPC)](https://arxiv.org/abs/2608.16663) | 2026 | LLM grounds phrases and picks from controlled options; code does graph traversal, role predicates, grain, SQL build, validation. 97.4% vs 55.3% direct, 0 wrong executions in 114 runs. | Closest published architecture to what Caliban should build. Small eval (38 Qs). | Adopt (architecture) |
| [GROUND](https://arxiv.org/abs/2608.26157) | 2026 | Governed semantic definitions + SQL validated against RLS, join and grain rules. Ungoverned baselines violated RLS, and "semantics-only" still leaked data. | Permissions must be compiled in. Accurate metric definitions alone do not secure the data. Single author, synthetic benchmark. | Adopt (principle) |
| [The Virtual Knowledge Graph System Ontop](https://www.semanticscholar.org/paper/The-Virtual-Knowledge-Graph-System-Ontop-Xiao/96d1a91cb3779b457ff74f26d307d726c16986a6) | 2020 | OBDA: ontology + R2RML mappings; SPARQL rewritten to SQL over the live DB, no materialization. | The proven theory for virtual (not copied) ontologies. Caliban should reuse the mapping model, not the RDF/SPARQL stack. | Prototype (concepts) |

### D. LLM-built ontologies, schema matching, KG+LLM

| Paper | Year | Core idea | Why it matters for Caliban | Verdict |
|---|---|---|---|---|
| [Unifying LLMs and KGs: A Roadmap](https://arxiv.org/abs/2306.08302) | 2023 | Taxonomy: KG-enhanced LLMs, LLM-augmented KGs, synergized LLM+KG. | Framing: Caliban is "LLM-augmented KG construction" + "KG-enhanced inference". | Watch (framing) |
| [LLM-empowered KG construction: A survey](https://arxiv.org/abs/2510.20345) | 2025 | How LLMs reshape ontology engineering → extraction → fusion; schema-based vs schema-free. | Map of techniques for the bootstrap pipeline. | Watch |
| [AutoSchemaKG](https://arxiv.org/pdf/2505.23628) | 2025 | Autonomous KG construction with dynamic schema induction at web scale. | Useful for unstructured sources (docs, tickets) feeding the same ontology. | Prototype |
| [Matchmaker](https://arxiv.org/abs/2410.24105) | 2024 | Zero-shot self-improving LLM program for schema matching: candidate gen → refine → confidence score (NeurIPS 2024). | Template for cross-source entity/column matching ("`cust_id` in Postgres = `AccountId` in Salesforce") with calibrated confidence. | Adopt |
| [LLMs4OM](https://arxiv.org/html/2404.10317v1) / [OLaLa](https://arxiv.org/pdf/2311.03837) | 2023–24 | Retrieve-then-LLM-match for ontology alignment; matches or beats classical OM on 20 datasets. | Retrieval-first matching keeps cost linear. [OAEI-LLM](https://arxiv.org/pdf/2409.14038) documents hallucinated matches, so human review is required. | Prototype |

### E. Graph RAG, non-SQL targets, tool/API generation

| Paper | Year | Core idea | Why it matters for Caliban | Verdict |
|---|---|---|---|---|
| [GraphRAG (Microsoft)](https://arxiv.org/abs/2404.16130) | 2024 | LLM-extracted entity graph + community summaries, map-reduce answers for "global" sensemaking queries. | Good for corpus-level questions over documents. Indexing is expensive (MS warns explicitly) and the repo is now in maintenance mode. | Watch |
| [LightRAG](https://arxiv.org/abs/2410.05779) | 2024 | Graph + vector dual-level retrieval with **incremental** updates; much cheaper than GraphRAG. | Better fit for a gateway: incremental, cheap. Candidate for the unstructured half of the ontology. | Prototype |
| [RAG vs GraphRAG: Systematic Evaluation](https://arxiv.org/abs/2502.11371) | 2025 | Each wins on different tasks; hybrid selection/integration does best. | Route per query: vector RAG vs graph RAG vs structured query. This is a routing feature. | Adopt (routing) |
| [Text-to-NoSQL (TEND benchmark + SAG)](https://arxiv.org/abs/2502.11201) | 2025–26 | 1,210 MongoDB-native tasks; strong NL2SQL models "degrade substantially"; schema-as-data grounding with execution verification. | NoSQL needs inferred schemas from sampled documents plus execution checks. Do not reuse the SQL prompt. | Prototype |
| [CypherBench](https://arxiv.org/abs/2412.18702) | 2024 | 11 property graphs, 7.8M entities, 10k+ Qs; property-graph *views* over RDF to tame huge schemas. | Views are the trick: expose a narrow, typed graph view of the ontology to the LLM. | Watch |
| [Gorilla](https://arxiv.org/abs/2305.15334) | 2023 | Retrieval-aware fine-tuned API-call generation; retrieval cuts hallucinated calls and adapts to doc changes. | Text-to-API = retrieve the OpenAPI operation, then generate constrained arguments. | Adopt (pattern) |

### F. Security of the data path

| Paper | Year | Core idea | Why it matters for Caliban | Verdict |
|---|---|---|---|---|
| [From Prompt Injections to SQL Injection (P2SQL)](https://arxiv.org/abs/2308.01990) | 2023/ICSE'25 | LangChain-style NL→SQL apps are highly susceptible to prompt-to-SQL injection across 7 LLMs, both direct and via stored data. | DB privileges and plan-level policy, not prompts, must enforce security. | Adopt (threat model) |
| [MCP at First Glance](https://arxiv.org/abs/2506.13538) | 2025–26 | 1,899 open-source MCP servers: 7.2% with general vulnerabilities, 5.5% with MCP-specific tool poisoning, 66% with code smells. | Third-party MCP connectors are untrusted code. | Adopt (threat model) |
| [MCPTox](https://arxiv.org/abs/2508.14925) | 2025 | 1,348 poisoned-metadata cases on 45 live servers; o1-mini 72.8% ASR; best refusal <3%. More capable models are *more* susceptible. | Tool descriptions must be sanitized/pinned. Model alignment will not save you. | Adopt (threat model) |
| [SoK: Security & Safety in the MCP Ecosystem](https://arxiv.org/abs/2512.08290) | 2025 | Taxonomy across Resources/Prompts/Tools; defenses from provenance (ETDI) to runtime intent verification. | Checklist for Caliban's MCP gateway. | Adopt |
| [Apache Arrow DataFusion (SIGMOD '24)](https://dl.acm.org/doi/10.1145/3626246.3653368) | 2024 | Embeddable, modular Rust query engine with 16+ extension APIs. | The federation core for Caliban's Rust data plane. | Adopt |
| [ConnectorX (VLDB '22)](https://www.vldb.org/pvldb/vol15/p2994-wang.pdf) | 2022 | Client-side overhead dominates DB→dataframe loading; Rust, parallel partitioned reads; up to 21× faster, 3× less memory. | Bulk-extract path for acceleration/caching. | Adopt |

---

## Open-source implementations

| Project | Language | Use | Notes |
|---|---|---|---|
| [Apache DataFusion](https://docs.rs/datafusion/latest/datafusion/) | **Rust** | Federated SQL planner/executor over Arrow | Core engine. Custom `TableProvider`s, optimizer rules (inject RLS), UDFs (LLM functions). |
| [datafusion-federation](https://github.com/datafusion-contrib/datafusion-federation) | **Rust** | Pushes the largest possible sub-plans to remote engines | Alpha status. Supports Flight SQL/Substrait. Needed so joins/aggregates run at source. |
| [datafusion-table-providers](https://github.com/datafusion-contrib/datafusion-table-providers) | **Rust** | Postgres, MySQL, SQLite, ClickHouse, DuckDB, Oracle, MongoDB, Flight SQL, ADBC, ODBC | Community (not ASF). Apache-2.0. Starting connector set. |
| [datafusion-sqlparser-rs](https://github.com/apache/datafusion-sqlparser-rs) | **Rust** | Multi-dialect SQL parser | Syntax only, no semantics. Use it for AST-level guardrails (statement allow-list, table/column extraction). |
| [ConnectorX](https://github.com/sfu-db/connector-x) | **Rust**/Py | Fast parallel DB → Arrow loads | Use for cache warming / materialization. |
| [Arrow ADBC](https://github.com/apache/arrow-adbc) | C/Go/Java/**Rust**/… | Arrow-native DB connectivity standard (v1.0) | Preferred driver interface for warehouses (Snowflake, BigQuery, etc.) where drivers exist. |
| [Spice.ai OSS](https://github.com/spiceai/spiceai) | **Rust** | DataFusion-based federation + acceleration + hybrid search + LLM gateway + MCP + NSQL | Apache-2.0. **Closest existing analog** to Caliban's data plane: study it, possibly fork components, and treat it as a competitor. |
| [WrenAI](https://github.com/Canner/WrenAI) | Python + **Rust** | Governed text-to-SQL with MDL semantic layer on a DataFusion engine, 22+ sources, MCP | Mixed licenses (Apache/AGPL). MDL is a good reference schema for models/relationships/metrics. |
| [MetricFlow](https://www.getdbt.com/blog/open-source-metricflow-governed-metrics) | Python | Compiles metric definitions → SQL deterministically | Apache-2.0 since Oct 2025. Reference implementation for Open Semantic Interchange, now [Apache Ossie (incubating)](https://ossie.apache.org/updates/announcing-open-source-metricflow-governed-metrics-for-ai-and-agents/). **Import/export target** for Caliban's metric model. |
| [Cube Core](https://github.com/cube-js/cube) | TS + **Rust** (CubeStore) | Semantic layer: metrics, joins, access rules → SQL/REST/GraphQL | Apache-2.0 backend. Second import format. Its access-control model is worth copying. |
| [Ontop](https://github.com/ontop/ontop) | Java | Virtual KG: SPARQL → SQL via R2RML | Apache-2.0. Reference for mapping semantics. Not for the hot path. |
| [CHESS](https://github.com/ShayanTalaei/CHESS) | Python | Reference text-to-SQL agent (IR/SS/CG/UT) | Read for value-retrieval and unit-test ideas. |
| [MAC-SQL](https://github.com/wbbeyourself/MAC-SQL) | Python | Multi-agent text-to-SQL baseline | Baseline only. |
| [Spider 2.0](https://github.com/xlang-ai/Spider2) | Python | Enterprise text-to-SQL eval harness | Use (after re-auditing) for regression evals. |
| [Vanna 2.0](https://github.com/vanna-ai/vanna) | Python | Agentic text-to-SQL, user-aware RLS, audit logging | MIT. Good product reference for identity propagation. |
| [Microsoft GraphRAG](https://github.com/microsoft/graphrag) | Python | Graph-based RAG over text | MIT, maintenance mode, expensive indexing. |
| [LightRAG](https://github.com/hkuds/lightrag) | Python | Incremental graph+vector RAG | Lighter alternative. Algorithm is small enough to port to Rust. |
| [DataHub MCP server](https://github.com/acryldata/mcp-server-datahub) | Python | Catalog search, column lineage, schema, historical SQL via MCP | Shows how to pull existing catalog metadata (and query history) into ontology bootstrap. Per-user auth preserved. |

---

## Risks & failure modes

| Risk | Mechanism | Evidence | Mitigation in Caliban |
|---|---|---|---|
| **Hallucinated / wrong joins** | LLM invents FK paths or joins at the wrong grain (fan-out double counting, chasm traps). The result *looks* plausible. | dbt 2026: text-to-SQL fails *silently* while the SL errors explicitly ([link](https://docs.getdbt.com/blog/semantic-layer-vs-text-to-sql-2026)). Direct generation produced 29 wrong runs of 114 vs 0 for compiled plans ([SPC](https://arxiv.org/abs/2608.16663)). | Joins only along *approved* ontology relations with declared cardinality. The compiler handles grain. Row-count explosion check on dry run. |
| **Business-logic misses** | Metric definitions live in docs and people's heads, not schemas. | EntSQL best 15.9% even with documents ([link](https://arxiv.org/abs/2606.03363)). +17–23 pts from a 4 KB semantic doc ([link](https://arxiv.org/abs/2604.25149)). | Extract rules into structured metrics. Retrieve the definition, not the PDF. |
| **Schema drift** | Columns renamed/dropped, new partitions, semantic change of a column ("status=3" meaning changes). | BEAVER/Spider 2.0 show sensitivity to schema shape ([BEAVER](https://arxiv.org/abs/2409.02038)). | Scheduled introspection diff. Ontology elements bound to physical columns go `stale`, which invalidates caches and blocks queries touching them until re-approved (auto-approve trivially safe renames). |
| **Permission bypass** | LLM-generated SQL ignores RLS. Semantic definitions without access policies leak data. Gateway uses a superuser connection. | GROUND: ungoverned baselines violated RLS, and "semantics-only" still leaks ([link](https://arxiv.org/abs/2608.26157)). P2SQL ([link](https://arxiv.org/abs/2308.01990)). | Policies are plan rewrites in DataFusion (predicate injection, column masking) applied *after* generation and *before* execution. Per-user credentials or impersonation at source where supported. Read-only roles. |
| **Prompt injection through data** | Row values, column comments, doc chunks or API responses contain instructions that the LLM follows on the next step. | P2SQL covers indirect injection via stored data ([link](https://arxiv.org/abs/2308.01990)). SoK on MCP Resources ([link](https://arxiv.org/abs/2512.08290)). | Quote/fence all data returned to the LLM, strip instructions from metadata at ingest, never let query results authorize new tool calls without a policy check. Separate "planner" context from "result summarization" context. |
| **MCP tool poisoning / supply chain** | Malicious tool descriptions, rug-pull updates, over-broad OAuth scopes, token passthrough, confused deputy, SSRF via OAuth discovery. | 5.5% of 1,899 servers show tool poisoning ([link](https://arxiv.org/abs/2506.13538)). 72.8% ASR, <3% refusal ([MCPTox](https://arxiv.org/abs/2508.14925)). [MCP Security Best Practices](https://modelcontextprotocol.io/specification/draft/basic/security_best_practices). | Pin + hash tool manifests, diff on change, allow-list servers per tenant, no token passthrough, egress proxy, sandbox local servers, scope minimization. |
| **Benchmark-driven false confidence** | Leaderboards contain wrong gold SQL. | 52.8% / 62.8% error rates in audited samples ([link](https://arxiv.org/abs/2601.08778)). | Per-tenant golden sets curated in the ontology UI. Track "verified answer" rate, not EX. |
| **LLM ontology-matching hallucinations** | Confident wrong cross-source matches. | [OAEI-LLM](https://arxiv.org/pdf/2409.14038). | Confidence + human approval for relations/matches. Value-overlap evidence required for joins. |
| **NoSQL / API degradation** | Schema-less docs, nested arrays, pagination, rate limits. | TEND: strong NL2SQL models degrade substantially ([link](https://arxiv.org/abs/2502.11201)). | Infer schemas from samples. Prefer exposing NoSQL/APIs as typed tables in DataFusion, so the LLM writes one dialect. |
| **Cost blow-up** | Multi-candidate generation, GraphRAG indexing. | CHASE-SQL N× generators. GraphRAG indexing warning ([repo](https://github.com/microsoft/graphrag)). | Candidate ensembles only on cache miss + high-stakes intents. LightRAG-style incremental indexing. |

---

## Recommended design for Caliban

### 1. Connector abstraction (Rust)

Use a single trait. Every source becomes **Arrow `RecordBatch` streams behind a DataFusion `TableProvider`**, so the LLM-facing layer sees one SQL dialect and one ontology. Native dialects are used only through pushdown.

```rust
trait Connector: Send + Sync {
    fn capabilities(&self) -> Caps;               // pushdown: filter/proj/agg/join/limit, dialect, CDC, impersonation
    async fn introspect(&self) -> SchemaSnapshot; // tables/collections/endpoints, PK/FK, comments, stats
    async fn profile(&self, obj: &ObjRef, budget: SampleBudget) -> Profile; // distincts, nulls, top values, sample docs
    async fn query_log(&self, since: Ts) -> Vec<HistoricalQuery>;           // where available (warehouses, catalogs)
    fn table_provider(&self, obj: &ObjRef, principal: &Principal) -> Arc<dyn TableProvider>;
    async fn change_feed(&self) -> Option<ChangeStream>; // CDC/watermarks/snapshot IDs for cache invalidation
}
```

| Tier | Sources | Mechanism |
|---|---|---|
| 1 SQL / warehouses | Postgres, MySQL, SQL Server, Oracle, Snowflake, BigQuery, Databricks, ClickHouse | `datafusion-table-providers` + `datafusion-federation` for full sub-plan pushdown. ADBC/Flight SQL where available. |
| 2 NoSQL / graph | MongoDB, DynamoDB, Elastic, Neo4j | Sampled schema inference → virtual tables. Nested fields flattened/unnested. Native query generation only via typed pushdown, not free-form LLM Mongo/Cypher (TEND evidence). |
| 3 APIs / SaaS | Salesforce, HubSpot, Jira, REST/GraphQL via OpenAPI | OpenAPI ops → table functions (`sf_opportunities(filters…)`) with pagination/rate-limit handled in Rust. Gorilla-style retrieval of op docs for long-tail ops. |
| 4 Files / lakes | S3/GCS/ADLS Parquet/CSV/JSON, Iceberg/Delta | `object_store` + DataFusion native readers. |
| 5 MCP servers | Customer/third-party MCP tools | **Untrusted connector class.** Pinned manifests, sandboxed, outputs treated as data, never as instructions. Caliban also *exposes* its ontology as an MCP server to customer agents. |

Credentials live in a vault and are scoped per tenant and per connector. Connections use per-principal impersonation when the source supports it (Snowflake/BigQuery/Postgres roles), otherwise a read-only service role **plus** Caliban-enforced policies.

### 2. Ontology representation ("Caliban Semantic Model", CSM)

Store it as a typed, versioned property graph in Caliban's metadata store (Postgres), serialized as YAML for git-style review, with **import/export for MetricFlow/OSI, Cube, WrenAI MDL and dbt**. Do not adopt RDF/OWL internally: borrow Ontop's *mapping* idea (logical entity ↔ physical bindings), not its stack.

| Object | Key fields |
|---|---|
| `Entity` | name, description, synonyms[], keys, `bindings[]` (source, object, filter, column map; several sources allowed, which is how cross-source "Customer" works), owner, status |
| `Attribute` | entity, physical column(s), type, *semantic type* (currency, email, country, enum w/ value labels), unit, PII class, synonyms, sample/top values, value-index flag |
| `Relation` | from/to entity, join keys, **cardinality** (1:1/1:N/N:M), optional/required, evidence (declared FK, value-overlap %, query-log frequency, LLM), verified flag |
| `Metric` | MetricFlow-compatible: measure expr, aggregation, filters, default time dimension, grain constraints, allowed dimensions |
| `Dimension` / `TimeDimension` | attribute ref, hierarchy (day→month→quarter), fiscal calendar |
| `GlossaryTerm` | business phrase → metric/entity/filter (e.g., "active customer" → `customers.status='A' AND last_order > now()-90d`) |
| `Policy` | row predicate template (`region = {principal.region}`), column mask/deny, purpose/intent allow-list, PII handling (route to anonymizer) |
| `VerifiedQuery` | NL question, compiled plan/SQL, result fingerprint, approver. Used as few-shot examples (DAIL-SQL) and as cache seeds. |
| Cross-cutting | `provenance` (introspect / profile / log-mined / LLM / human), `confidence`, `status` (proposed → approved → deprecated/stale), `version`, `bound_schema_hash` |

### 3. Bootstrap (LLM-proposed, human-approved)

1. **Introspect** catalogs, PK/FK, comments. Pull existing dbt/Cube/LookML/DataHub metadata if present (via the DataHub MCP server or APIs).
2. **Profile** with sampling: distinct counts, top-k values, null rates, value patterns. Build value sketches (MinHash) for join discovery.
3. **Mine query logs** (warehouse history, catalog "queries" tabs): frequent join paths, common filters, metric-like aggregations → candidate `Relation`/`Metric`/`VerifiedQuery` (LinkedIn; Beyond the Harness).
4. **LLM proposal pass**, per table cluster (ReFoRCE-style compression of sharded tables): entity names, descriptions, synonyms, semantic types, PII classes, enum labels, candidate implicit relations. Cross-source matching uses Matchmaker / LLMs4OM: retrieve candidates by embedding + value overlap, then LLM refine, then calibrated confidence.
5. **Human curation UI**: a queue sorted by impact (query frequency × uncertainty). Auto-approve only low-risk items (descriptions, synonyms above threshold). **Relations, metrics and policies always need a human.**
6. **Continuous learning (GATE-style)**: when exploratory queries succeed and users confirm, propose new glossary terms or relations. Failed/negative feedback demotes them.

### 4. Indexing for retrieval

- **Cards, not DDL.** Embed one compact card per entity, attribute, metric, glossary term and verified query (name + synonyms + description + type + sample values). Use M-Schema-like serialization for prompts.
- **Hybrid retrieval.** Dense vectors (per-tenant namespace in Caliban's vector DB) + BM25 + a **value index** (CHESS-style LSH/trigram over high-value categorical columns) so literals resolve to columns.
- **Graph expansion.** Retrieved seed nodes → minimal connecting subgraph over *approved* relations (Steiner tree on the relation graph) → prune with high recall ("Death of Schema Linking": recall over precision).
- **Policy-filtered retrieval.** Filter the index by the principal's visible entities/columns *before* retrieval, so restricted schema never enters the prompt.
- Unstructured docs (wikis, runbooks) go into a LightRAG-style incremental graph+vector index whose entities are **linked to CSM entities**. This gives one ontology for structured and unstructured data.

### 5. Query generation & validation

```
NL → intent classifier → ontology retrieval → lane selection
  Lane A (in-scope metric question):  LLM emits JSON {metrics, dims, filters, time, order, limit}
                                      constrained to retrieved IDs → deterministic compiler (MetricFlow-like, in Rust) → LogicalPlan
  Lane B (exploratory):               LLM emits SQL over *logical* entity views only → parse → bind to CSM
                                      (1 candidate; N candidates + selector only for high-stakes/uncached, per CHASE/XiYan)
  Lane C (doc/global question):       vector RAG / graph RAG
→ VALIDATE → EXECUTE (DataFusion federation) → answer + lineage + "verified/unverified" badge
```

Validation pipeline (all deterministic, in Rust):

1. **Parse**: `sqlparser-rs`. Single `SELECT` only. Deny DDL/DML, multiple statements, system tables, `COPY`/external functions.
2. **Bind to ontology (OBQC-style)**: every table/column must resolve to an approved CSM element. Every join must match an approved `Relation`. Aggregations over N-side of 1:N joins flagged (fan-out). Errors are explained back to the LLM for repair (≤2 rounds), then "I don't know".
3. **Policy rewrite**: a DataFusion optimizer rule injects row predicates and masks/denies columns based on the principal. PII columns route through the anonymizer. This happens *after* generation, so the LLM cannot opt out.
4. **Cost and safety dry run**: `EXPLAIN` at source / DataFusion plan stats. Enforce byte-scan and row limits. Run with `LIMIT` first. Detect row explosion vs expected cardinality. Enforce timeouts.
5. **Result sanity**: empty-result probes (ReFoRCE column exploration), type/unit checks, compare to verified-query fingerprints when available.
6. **Return** SQL, logical plan, ontology elements used, and policy rewrites applied (an audit record).

### 6. Federation via DataFusion

- One `SessionContext` per request, built from the principal's visible `TableProvider`s.
- `datafusion-federation` pushes maximal sub-plans to each source. Cross-source joins execute in Caliban over Arrow.
- **Acceleration tier**: hot entities are materialized (ConnectorX/ADBC bulk loads into local Parquet/Arrow, refreshed by CDC/watermarks), following Spice.ai's model. The ontology marks which bindings may be accelerated (policy: some data must never leave the source; this matters for the sovereign/on-prem mode).
- LLM UDFs (classify/extract/summarize) are registered as DataFusion UDFs that call back into the Caliban router (the Spider 2.0-AIFunc direction).

### 7. How the ontology feeds routing and caching

| Signal from ontology retrieval | Routing decision |
|---|---|
| Fully covered by approved metrics | Lane A with a small/cheap model (choice task only) |
| Needs unapproved joins / many entities / multi-source | Lane B with a stronger reasoning model, optional multi-candidate |
| Touches PII-classed attributes or "restricted" sources | Force anonymizer; restrict to on-prem/sovereign models |
| Doc-only entities | RAG lane; GraphRAG only for "global" summarization intents |
| No ontology hit | Clarifying question or "I don't know". Never free-form SQL over raw schema in production. |

**Cache keys (layered):**

- *NL → plan cache* (semantic, vector-matched): key = normalized intent + resolved CSM IDs + **ontology version** + principal's **policy hash**.
- *Plan → result cache*: key = canonical plan hash + per-source **data version** (CDC LSN, snapshot ID, max(updated_at) watermark) + policy hash.
- **Invalidation**:
  - Ontology publish invalidates plan entries referencing the changed element IDs.
  - Schema drift marks elements `stale` and evicts entries that reference them.
  - CDC/watermark advance evicts result entries for the affected entity.
  - A policy change rotates the policy hash.
  - Never share result-cache entries across different policy hashes.

---

## Open questions

1. **Standard choice:** commit to OSI/Apache Ossie (MetricFlow schema) as the canonical metric format, or keep CSM as a superset with lossy export? Ossie is still incubating.
2. **Build vs fork:** Spice.ai OSS already ships Rust DataFusion federation + acceleration + MCP + text-to-SQL. Should Caliban fork, embed, or compete? WrenAI's engine is another candidate.
3. **Identity propagation:** which sources support true per-user impersonation (vs service account + Caliban policy)? This determines whether Caliban is a policy enforcement point or only a policy decision point.
4. **Curation economics:** how many human-approval minutes per 100 tables before the ontology is "good enough"? No published data. Needs pilot measurement.
5. **Eval:** how to build per-tenant golden sets cheaply ([BenchPress](https://arxiv.org/pdf/2510.13853)-style human-in-the-loop annotation) given the annotation-error findings.
6. **NoSQL/API fidelity:** how much is lost by flattening Mongo/API data into tables vs native query generation? TEND suggests native is hard, but flattening loses nested semantics.
7. **Data-borne injection:** is "planner/summarizer context separation + data fencing" sufficient, or is a dedicated injection classifier on retrieved values needed? The literature has no robust defense yet.
8. **MCP evolution:** the draft spec now describes MCP as stateless with explicit state handles. Caliban's MCP gateway must track spec churn and pin versions per tenant.
9. **Sovereign mode:** can the bootstrap LLM passes (profiling + proposal) run acceptably on on-prem open models, given that open models lag proprietary ones by ~10 pts on recent benchmarks (e.g., Spider 2.0-AIFunc)?
