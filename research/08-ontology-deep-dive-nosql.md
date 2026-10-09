# Ontology Deep Dive: MongoDB-as-Warehouse and the Ontology Store

*Research date: 2026-10-02. Second pass on [03](03-ontology-and-datasource-layer.md), closing the text-to-NoSQL gap listed in [reference architecture §10](../architecture/caliban-reference-architecture.md#10-known-research-gaps). Trigger: target customers run their "warehouse" on MongoDB.*

**Summary.**

**(A) MongoDB as the source.** Keep MongoDB as the customer's system of record, but do not use it as the engine for heavy analytics, and never let the LLM write MQL directly. The evidence is consistent:

- **Text-to-MQL accuracy.** It is the weakest of SQL, Cypher and MQL in a controlled multi-model benchmark (SM3: MQL 30–40% vs SQL 49–58% with the same models).
- **Schema-less collections.** On realistic MongoDB-native data, the best published solver reaches 35.8%. A schema-only prompt reaches 0.2% (TEND v3).
- **Speed.** On ClickBench, MongoDB took about 19,100 s across 43 analytic queries. DataFusion-on-Parquet took 46 s and ClickHouse 18 s on the same instance type.
- **MongoDB's own SQL and BI paths.** They are gone for on-prem customers: the BI Connector reached end of life in September 2026, the SQL Interface needs Enterprise Advanced or Atlas, and Data Federation is Atlas-only. Cube's MongoDB support depends on the BI Connector.

So Caliban should do the following:

1. The LLM emits a **typed Caliban Query IR (CQIR)** that references ontology IDs only.
2. A deterministic Rust compiler lowers the IR to **either** a validated MongoDB aggregation pipeline (fresh, selective queries on a read-only, JS-disabled, time-boxed secondary) **or** SQL over a **CDC-fed columnar replica**. The replica is fed by change streams, stored as Arrow and Parquet, and queried by DataFusion inside the same binary.
3. A planner picks the target per query from freshness, `explain()` cost and policy.

The ontology layer is what makes this work: it states which embedded arrays are child entities, which ObjectIds are references, and which documents are subtypes. Flattening cannot recover that information, and an LLM cannot reliably infer it.

**(B) Where the ontology lives.** Keep the ontology in **Postgres**, the control-plane store that already exists. Store it as typed, append-only, versioned rows with JSONB bodies. Compile it into an **in-memory typed graph** that ships inside the signed data-plane snapshot. Use **Oxigraph** (Rust, Apache-2.0/MIT) only as an optional RDF/OWL import/export library.

Reject the alternatives:

- **Graph database servers.** Neo4j is GPLv3, Memgraph is BSL, and Kuzu was archived in October 2025.
- **MongoDB as the ontology store.** It is licensed under SSPL and offers no transactional versioning advantage.
- **An RDF store as the primary store.** Its SPARQL evaluation is "not optimized yet", and LLMs are worst at SPARQL.

An ontology is a small metadata graph (10³–10⁵ nodes). It needs versioning and joins, not a graph engine.

---

## Key papers & benchmarks

| Source | Year | Core idea | Numbers | Relevance | Verdict |
|---|---|---|---|---|---|
| [TEND + SAG (Text-to-NoSQL)](https://arxiv.org/abs/2502.11201) ([v3 HTML](https://arxiv.org/html/2502.11201v3)) | 2025–26 | MongoDB-native benchmark: 1,210 tasks, 11 DBs, nested arrays, sparse paths, polymorphism, dynamic keys. SAG = schema-as-data grounding from stored docs → bounded MQL → execution repair → consistency selection. | DeepSeek-V4-Flash backbone: schema-only direct **0.2%**, sampled-doc direct 28.6%, ReAct 30.8%, **SAG 35.8%** EXC. 61.5% of failures are "wrong rows" (the pipeline runs but returns wrong data). Dynamic-key maps appear in 83.7% of tasks. | Shows the problem with free-form MQL. **Schema descriptions alone are useless; you need document evidence plus execution checks.** Failures are silent. | Adopt (as eval + grounding ideas) |
| [EvoMQL (Draft-Refine-Optimize)](https://arxiv.org/abs/2604.13045) | 2026 | Self-evolving NL2MQL: query-aware evidence retrieval + RL with execution rewards; 3B active params. | 76.6% on MongoDB "EAI" set, **83.1% on TEND** (not comparable with TEND v3's 35.8%; different version and metric). | A small on-prem model can be specialized for MQL, which matters for the sovereign SKU. Only useful if raw MQL is ever needed. | Watch |
| [MongoDB NL→mongosh benchmark](https://huggingface.co/datasets/mongodb-eai/natural-language-to-mongosh) | 2025 | MongoDB's own eval: 766 cases, 8 Atlas sample DBs, 9 models, prompt-strategy ablation. | Claude 3.7 Sonnet 86.7% avg, GPT-4o 82.5%, Mistral Large 2 71.3%. Annotated schemas and sample docs help. Agentic mode costs 3.5× time and 8.94× tokens. | The optimistic upper bound: Atlas sample DBs are public (contamination risk) and simple. Annotated schema = what the ontology provides. | Adopt (prompt findings), distrust headline |
| [SM3-Text-to-Query](https://arxiv.org/abs/2411.05521) ([HTML](https://arxiv.org/html/2411.05521v2)) | 2024 | Same medical data in Postgres, MongoDB, Neo4j and GraphDB; 10K Q/query pairs per language. | 5-shot with schema, Llama3-70b: SQL 57.5, Cypher 57.1, **MQL 40.4**, SPARQL 30.5. GPT-3.5: SQL 56.3, **MQL 35.1**. | The only controlled comparison across query languages on identical data. **LLMs are 15–20 pts worse at MQL than SQL.** Argues for compiling to MQL, not generating it. | Adopt (core evidence) |
| [Conversational Text-to-NoSQL (Stage-MCTS, CoNoSQL)](https://arxiv.org/abs/2602.12574) | 2026 | Multi-turn NoSQL; MCTS-generated reasoning data to fine-tune small models; 2,000+ dialogues, 150 DBs. | +7.93% EVM over large reasoning models. | Follow-up questions ("now by region") are the norm in chat. In Caliban, an IR diff handles this better than regenerating a query. | Watch |
| [VKG for IoT telemetry chatbots](https://arxiv.org/abs/2506.22267) | 2025 | Builds a per-query virtual KG over a NoSQL telemetry lake (Cassandra/KairosDB); LLM writes SPARQL over the KG, not NoSQL. | **92.5% vs 25%** for direct LLM-to-NoSQL; latency 20.4 s → 3.0 s; Llama-3.1-8B on-prem on one H100; 100 questions from 12 templates. | Direct evidence that **an ontology-mediated target beats native NoSQL generation**, including with a small on-prem model. Templated questions, so optimistic. | Adopt (principle) |
| [EvoOntology](https://arxiv.org/abs/2609.15779) | 2026 | Ontology served as an MCP server (schema/content/tool layers); builder agent; typed edits accepted only if a paired held-out eval improves. | BIRD 58.1 → 67.9 avg over 6 backbones; DDR-Bench 69.5 → 89.5; a static semantic layer *underperformed* baseline on DDR (~64%). | (1) Gate ontology edits on eval deltas (extends GATE in 03). (2) A stale, static semantic layer can hurt, so drift handling is essential. | Adopt (edit-gating) |
| [Sequeda et al. KG benchmark](https://arxiv.org/abs/2311.07509) / [Ontologies to the Rescue](https://arxiv.org/abs/2405.11706) | 2023–24 | Ontology/KG representation vs raw SQL schema. | 16% → 54%; 72% with ontology-based query checking. | Already in 03. The mechanism carries over to documents: the ontology fixes the *meaning*, the compiler fixes the *syntax*. | Adopt (in 03) |
| [ClickBench: MongoDB entry](https://github.com/ClickHouse/ClickBench/tree/main/mongodb) | 2022–26 | 43 analytic queries on 100M-row web-analytics table, same `c6a.4xlarge`. | Sum of best-of-3 runs: **MongoDB 19,144 s** (2022 run, queries hand-adapted to indexes), PostgreSQL 11,875 s, **DataFusion-Parquet 45.6 s, DuckDB 26.3 s, ClickHouse 17.8 s**. Storage: Mongo 86.4 GB vs Parquet 14.8 GB. Load: Mongo 44,824 s vs ClickHouse 178 s. | Row stores (Mongo and Postgres alike) are 2–3 orders of magnitude slower on scan-heavy aggregates. This is the case for a columnar replica. Caveat: the Mongo result is old and the README admits manual index tuning. | Adopt (sizing evidence) |
| [Taxonomy of Schema Changes for NoSQL (U-Schema/Orion)](https://arxiv.org/abs/2205.11660) | 2022 | Unified model with aggregation (embedding) and reference relations, structural variations, and a change taxonomy. | n/a | U-Schema's split of **aggregation vs reference relationships and structural variations** is the right shape for Caliban `Relation.kind` and drift classification. | Adopt (concepts) |
| [Text2Cypher](https://arxiv.org/abs/2412.10064) / [Grounded Text2Cypher data gen](https://arxiv.org/abs/2606.14325) | 2024–26 | Fine-tuning data for NL→Cypher; synthetic grounded data lets small models match large ones. | 44,387 instances; "significant" gains (abstract). | Supports **not** exposing Cypher to the LLM: it is another dialect to train. Caliban's ontology graph is for the compiler, not a query target. | Watch |
| [Portable Ontological Expressions in NoSQL Queries](https://arxiv.org/abs/1610.06084) | 2016 | Embeds ontology "address expressions" in MongoDB queries to decouple them from physical paths. | Preliminary perf only. | Early OBDA-over-Mongo idea, i.e., logical paths resolved to physical paths. CQIR does the same with typed IDs. Little OBDA-over-Mongo research has followed. | Watch (prior art) |

**Text-to-SQL vs text-to-MQL, the numbers side by side.**
- On controlled, identical data, MQL trails SQL by 15–20 pts (SM3).
- MQL is solved at 83–87% only on public or simple sample DBs (MongoDB eval, EvoMQL).
- MQL drops to 0.2–36% on MongoDB-native complex documents (TEND v3).
- The 03 enterprise text-to-SQL numbers (10–16% raw → 54–72% with an ontology → 98–100% in a governed semantic layer) suggest the same lever applies. The ontology plus a compiler gives most of the gain whatever the target language.

---

## Tools, engines & crates

| Tool | Licence | Rust usability | On-prem fit | What it does / key finding | Verdict |
|---|---|---|---|---|---|
| [mongo-rust-driver](https://github.com/mongodb/mongo-rust-driver) | Apache-2.0 | Native | ✅ | Official async driver: aggregate, explain, change streams, read prefs. | **Use** (foundation) |
| [datafusion-table-providers-mongodb](https://github.com/datafusion-contrib/datafusion-table-providers) ([crate](https://crates.io/crates/datafusion-table-providers-mongodb)) | Apache-2.0 | Native | ✅ | Source read 2026-10-02: **only `find()` filter/projection/sort/limit pushdown, no aggregate pushdown**. Schema inferred from `find({}).limit(N)` (first N in natural order, **not `$sample`**). **Nested docs become JSON strings**, arrays become `List<Utf8>`, conflicting types widen to `Utf8`, `Decimal128` is fixed at `(18,6)`, and midnight datetimes are inferred as `Date32`. | **Fork/extend**: fine for small lookups, wrong for analytics and lossy for semantics. |
| [datafusion-federation](https://github.com/datafusion-contrib/datafusion-federation) | Apache-2.0 | Native | ✅ | Sub-plan pushdown framework (SQL-oriented). | Use for SQL sources. Mongo needs a custom IR→pipeline "unparser". |
| [duckdb-mongo](https://github.com/stephaniewang526/duckdb-mongo) ([community ext](https://github.com/duckdb/community-extensions/tree/main/extensions/mongo)) | MIT | C++ (via `duckdb` crate) | ✅ | Pushes filters, projections, TopN and **COUNT/SUM/MIN/MAX/AVG**; reads Atlas SQL `__schema` docs; v0.3.0 **adds writes** (INSERT/CTAS/DROP). | Reference design for aggregate pushdown. **Do not ship** (write paths, C++ dependency). |
| [Spice.ai MongoDB connector](https://spiceai.org/docs/components/data-connectors/mongodb) | Apache-2.0 | Native | ✅ | Schema inference from 400 docs by default, `mongodb_unnest_depth`, arrays → `List<Utf8>`; **change-stream CDC into DuckDB/SQLite/Postgres accelerators** (`refresh_mode: changes`). | Closest prior art for "CDC → acceleration". Study it, and decide fork vs build (03 open question). |
| [Trino MongoDB connector](https://trino.io/docs/current/connector/mongodb.html) | Apache-2.0 | JVM | ✅ (heavy) | Schema stored in a `_schema` collection, guessed from samples ("can be incorrect… modify manually"); objects → `ROW`, arrays → `ARRAY`. | Reject as runtime (JVM). Copy the persisted-schema idea. |
| [ClickHouse MongoDB engine](https://clickhouse.com/docs/engines/table-engines/integrations/mongodb) | Apache-2.0 | HTTP/native client | ✅ | Read-only; only "simple expressions" (`WHERE field = const ORDER BY … LIMIT`) are pushed down; nested docs → JSON strings. | Not a pushdown story. ClickHouse is useful only as a **CDC sink** for very large tenants. |
| [ClickPipes for MongoDB](https://clickhouse.com/docs/integrations/clickpipes/mongodb) | Proprietary (Cloud) | — | ❌ | Change streams → ClickHouse JSON type; public beta; needs Mongo 5.1+. | Reject (cloud-only). The PeerDB core is [AGPL-3.0](https://github.com/PeerDB-io/peerdb). |
| [Debezium MongoDB](https://debezium.io/documentation/reference/stable/connectors/mongodb.html) | Apache-2.0 | JVM + Kafka | ✅ | Change streams with resume tokens; initial snapshot; pre-images on 6.0+; **requires a replica set** (no standalone); 16 MB event limit; oplog-window loss forces re-snapshot. | Support as an *input* for customers who already run Kafka. Not required. |
| [mongodb-schema](https://github.com/mongodb-js/mongodb-schema) | Apache-2.0 | JS (port the algorithm) | ✅ | Probabilistic schema: per-path probability, type distribution, `hasDuplicates`, semantic-type detectors (GeoJSON, custom e.g. email); default 100 random docs. | **Port the algorithm to Rust.** |
| [Variety](https://github.com/variety/variety) | MIT | JS | ✅ | Key occurrence % per type; `limit` analyses the **newest** docs (`_id: -1`); `maxDepth`. | Copy the "newest-N" idea for drift sampling. |
| [Compass sampling](https://www.mongodb.com/docs/compass/current/sampling/) | — | — | — | 1,000 docs via `$sample`. | Baseline sample size. |
| [PyMongoArrow / mongo-arrow](https://github.com/mongodb-labs/mongo-arrow) ([types](https://www.mongodb.com/docs/languages/python/pymongo-arrow-driver/current/data-types/)) | Apache-2.0 | Python/C | ✅ (offline tooling) | Embedded doc → `struct`, array → `list_`; ObjectId/Decimal128 as **Arrow extension types**. | Copy the type mapping (struct/list, not JSON strings). |
| [polarsmongo2](https://crates.io/crates/polarsmongo2), [mongodb-arrow-connector](https://crates.io/crates/mongodb-arrow-connector) | various | Native | ✅ | Small BSON→Arrow readers; the latter last updated 2022. | Reference only. |
| [MongoSQL / SQL Interface](https://www.mongodb.com/docs/atlas/data-federation/query/sql/language-reference/) | Proprietary | — | ⚠️ EA/Atlas only | SQL-92 dialect with `FLATTEN`/`UNWIND`, read-only. [Schema Builder CLI](https://www.mongodb.com/docs/sql-interface/schema/schema-builder-cli/) scans **every document**, flags "unstable" (too polymorphic) namespaces, and **does not support Community**. | Not available to most on-prem customers. Copy the "unstable schema" concept. |
| [MongoDB BI Connector](https://www.mongodb.com/docs/bi-connector/current/) | Proprietary | — | ❌ | **EOL September 2026** ([transition guide](https://www.mongodb.com/docs/sql-interface/transition-bic-to-atlas-sql/)). | Dead path. |
| [Atlas Data Federation](https://www.mongodb.com/docs/atlas/data-federation/) | Proprietary | — | ❌ Atlas-only | Federated MQL over clusters/S3; scheduled `$out` to Parquet/CSV. | Optional snapshot path for Atlas customers only. |
| [Apache DataFusion](https://github.com/apache/datafusion) + [iceberg-rust](https://github.com/apache/iceberg-rust) | Apache-2.0 | Native | ✅ | Columnar execution over Parquet; Iceberg tables for customers who want a lakehouse. | **Use** (acceleration engine) |
| [DuckDB](https://github.com/duckdb/duckdb) | MIT | `duckdb` crate (C++) | ✅ | Fastest single-node option in ClickBench after ClickHouse (26 s). | Optional accelerator. Default to DataFusion to keep one Rust engine. |
| [ClickHouse](https://github.com/ClickHouse/ClickHouse) | Apache-2.0 | Client | ✅ (separate service) | 17.8 s ClickBench. | Optional sink for >TB or high-concurrency tenants. |
| **Ontology-store candidates** | | | | | |
| Postgres (JSONB + recursive CTE) | PostgreSQL | `sqlx`/`tokio-postgres` | ✅ already in CP | Transactions, versioning, RLS, familiar to DBAs. | **Chosen** |
| [Apache AGE](https://github.com/apache/age) | Apache-2.0 | SQL ext | ⚠️ needs extension install | openCypher inside Postgres. | Optional later; not needed at ontology scale. |
| [Oxigraph](https://github.com/oxigraph/oxigraph) | Apache-2.0 / MIT | **Native Rust** | ✅ | RocksDB-backed SPARQL 1.1, preliminary RDF 1.2; README: "SPARQL query evaluation has not been optimized yet". | **Embed for RDF/OWL import/export only.** |
| [Neo4j](https://github.com/neo4j/neo4j) | GPL-3.0 (Community) | Bolt client | ⚠️ copyleft, extra server | Mature Cypher DB. | Reject (licence + another server). |
| [Memgraph](https://github.com/memgraph/memgraph) | BSL 1.1 / enterprise | Bolt client | ⚠️ BSL | In-memory graph. | Reject (BSL). |
| [Kuzu](https://github.com/kuzudb/kuzu) / [LadybugDB fork](https://github.com/LadybugDB/ladybug) | MIT | C++ bindings | ⚠️ | **Kuzu repo archived (last push 2025-10-10)**; LadybugDB is a young fork. | Reject (project risk). |
| [SurrealDB](https://github.com/surrealdb/surrealdb) | BSL 1.1 | Rust | ⚠️ BSL | Multi-model. | Reject (BSL). |
| [TypeDB](https://github.com/typedb/typedb) | MPL-2.0 | Rust | ✅ | Typed knowledge graph with type inference. | Interesting, but another server for a small graph. Reject for now. |
| MongoDB as ontology store | [SSPL](https://github.com/mongodb/mongo) | driver | ⚠️ SSPL | — | Reject. |

---

## MongoDB as the source: what works, what breaks

### Correctness traps (why flattening or free-form MQL fails)

| Trap | Mechanism | Evidence | Caliban handling |
|---|---|---|---|
| **Array fan-out** | `$unwind` (or a flattened child table join) multiplies parent rows, so `SUM(order.total)` after unwinding `lines` double counts. | Same chasm-trap class as SQL fan-out (03). TEND: 96.7% of tasks use array operators. | `Relation{kind: embedded, card: 1:N}`. The compiler unwinds only when a metric's grain is the child entity, and aggregates parent metrics *before* unwinding. |
| **Same-element semantics** | `{"lines.sku":"A","lines.qty":{$gt:5}}` matches when *different* elements satisfy each condition; `$elemMatch` is needed for the *same* element. | [MongoDB docs](https://www.mongodb.com/docs/manual/tutorial/query-array-of-documents/) | IR filters on a child entity always compile to `$elemMatch` (native) or a child-table `EXISTS` (SQL). An "any element" vs "same element" choice is explicit in the IR. |
| **Missing vs null vs empty array** | Three distinct states; flattening collapses them to NULL. `$unwind` drops missing/empty unless `preserveNullAndEmptyArrays`. | TEND: optional/sparse paths in 96.2% of tasks. | Attribute carries `presence` stats. IR has `is_missing` / `is_null` / `is_empty` ops. Unwind mode is derived from inner vs outer relation semantics. |
| **Polymorphism** | One path holds int, string and doc across documents; subtypes share a collection ([polymorphic / inheritance patterns](https://www.mongodb.com/docs/manual/data-modeling/design-patterns/polymorphic-data/)). | DF provider widens conflicts to `Utf8`. The Schema Builder flags "unstable" namespaces. | Subtype entities bound by a discriminator filter. Per-path type histogram. Coercion rules need human approval. |
| **Dynamic keys** | Maps keyed by date, SKU or locale become thousands of "columns". | 83.7% of TEND tasks. | `MapAttribute` (key dimension + value type). Compiles to `$objectToArray` (native) or a key/value child table (replica). |
| **References without FKs** | ObjectId fields reference other collections with no declared constraint; `$lookup` is a left outer join to the *same database* and is slow if the foreign field is unindexed ([`$lookup` docs](https://www.mongodb.com/docs/manual/reference/operator/aggregation/lookup/)). | — | `Relation{kind: reference}` is only approved after a value-overlap check. The native lane allows `$lookup` only if `foreignField` is indexed. Otherwise the query goes to the replica. |
| **Denormalized copies** (extended references) | `order.customer.name` copied at write time drifts from `customers.name`. | — | Attribute `canonical_of: Customer.name`, `staleness: write-time copy`. Answers say which one was used. |
| **Unbounded arrays / 16 MB docs** | [Anti-pattern](https://www.mongodb.com/docs/manual/data-modeling/design-antipatterns/unbounded-arrays/); also the 16 MB change-event limit (Debezium). | — | Profile array-length p99. Large arrays become their own child tables in the replica. |

### Performance reality

- **Limits.** Pipeline stages are capped at **100 MB RAM** unless they spill (`allowDiskUse`). There is a **1,000-stage** cap. MongoDB 9.0 adds a **per-operation memory cap** (default max(1 GB, 20% of RAM)) ([limits](https://www.mongodb.com/docs/manual/core/aggregation-pipeline-limits/)). Large `$group`/`$sort` on big collections either spill to disk or fail.
- **Row store on scans.** ClickBench (above) shows roughly 400–1,000× slower total time than DataFusion, DuckDB or ClickHouse for scan-and-aggregate workloads, and about 6× the storage. Indexes help selective queries, not "revenue by category over 2 years".
- **Indexes.** Wildcard indexes exist for unknown fields but "don't perform as well as targeted indexes" ([docs](https://www.mongodb.com/docs/manual/core/indexes/index-types/index-wildcard/)). The current docs have no columnar index to rely on.
- **Workload isolation.** Atlas **analytics nodes** need M10+ ([docs](https://www.mongodb.com/docs/atlas/cluster-config/multi-cloud-distribution/)). Self-managed customers should use a **tagged (or hidden) secondary** with read-preference tag sets and `maxStalenessSeconds` ([read preference](https://www.mongodb.com/docs/manual/core/read-preference/)). Native analytic queries must never hit the primary.
- **On-demand materialized views** (`$merge`/`$out`) ([docs](https://www.mongodb.com/docs/manual/core/materialized-views/)) can pre-aggregate inside Mongo. They require **write** access to the customer cluster, which Caliban should not hold. They are a customer-side option only.
- **CDC is available on any replica set.** [Change streams](https://www.mongodb.com/docs/manual/changeStreams/) need a replica set or sharded cluster (WiredTiger, pv1). Resume tokens make the CDC process restartable. The oplog window bounds downtime before a re-snapshot is needed (Debezium docs).

### Safety (native lane)

| Control | Where | Source |
|---|---|---|
| Dedicated user with a **custom role**: `find`, `listCollections`, `listIndexes`, `collStats`, `dbStats` on approved DBs only. A **separate** CDC user with `changeStream` + `find`. Never `readWrite`. | Customer DB | [built-in roles](https://www.mongodb.com/docs/manual/reference/built-in-roles/) |
| Recommend `security.javascriptEnabled: false` / `--noscripting`. **Independently**, the Caliban validator denies `$where`, `$function`, `$accumulator`, `mapReduce`. JS in `$where` can run arbitrary code and peg CPU (OWASP example: a 10 s busy loop). | Customer config + Caliban | [server-side JS](https://www.mongodb.com/docs/manual/core/server-side-javascript/), [`$where`](https://www.mongodb.com/docs/manual/reference/operator/query/where/), [OWASP WSTG NoSQL](https://github.com/OWASP/wstg/blob/master/document/4-Web_Application_Security_Testing/07-Injection/05.6-NoSQL_Injection.md) |
| **Operator injection.** Literals come from typed IR values and are serialized as BSON scalars. Any `$`-prefixed key or Extended-JSON wrapper inside a literal is rejected. Field paths come only from ontology bindings, never from the LLM. | Caliban compiler | OWASP WSTG |
| Stage allow-list: `$match`, `$project`/`$set`, `$unwind`, `$group`, `$sort`, `$limit`, `$count`, `$bucket`, `$lookup` (approved relations only), `$facet` (capped), `$setWindowFields` (capped). **Deny** `$out`, `$merge`, `$unionWith` (unless an approved relation), `$currentOp`, `$listSessions`, `$collStats`, `$documents`, unbounded `$graphLookup`. | Caliban validator | [limits](https://www.mongodb.com/docs/manual/core/aggregation-pipeline-limits/) |
| `maxTimeMS` on every op. Recommend the cluster `defaultMaxTimeMS` (8.0+) as a backstop. | Both | [maxTimeMS](https://www.mongodb.com/docs/manual/reference/method/cursor.maxTimeMS/) |
| `allowDiskUse: false` in the native lane. If a query would spill, it belongs on the replica. | Caliban | [limits](https://www.mongodb.com/docs/manual/core/aggregation-pipeline-limits/) |
| **Dry run with `explain` (`queryPlanner` verbosity**, which does not execute). Reject or re-route on `COLLSCAN` over large collections, unindexed `$lookup`, or a blocking `$sort` without an index. | Caliban planner | [explain](https://www.mongodb.com/docs/manual/reference/command/explain/) |
| `comment: "caliban:<audit_id>"` on every command, so the customer's profiler and logs tie back to Caliban's audit record. | Caliban | — |

### How semantic layers handle MongoDB today

| Product | MongoDB path | Status 2026-10 |
|---|---|---|
| Cube | "accessed using SQL via the MongoDB Connector for BI"; community-supported driver ([docs](https://docs.cube.dev/admin/connect-to-data/data-sources/mongodb)) | **Broken by BI Connector EOL (Sept 2026)** |
| dbt / MetricFlow | Compiles metrics to **SQL** ([repo](https://github.com/dbt-labs/metricflow)); there is no MQL target | Needs a SQL engine in front of Mongo, i.e. a replica |
| MongoDB SQL Interface | MongoSQL over Atlas or **Enterprise Advanced** only ([schema mgmt](https://www.mongodb.com/docs/sql-interface/schema-management/)) | Unavailable on Community |
| Trino / ClickHouse engines | Sampled schemas, shallow pushdown (above) | Works, but is slow and lossy |
| Knowi | Native MQL with no flattening, plus cross-source joins and NL-to-MongoDB ([page](https://www.knowi.com/mongodb-analytics)) | Shows the market need for native document semantics. Vendor claim, no published accuracy numbers. |

**Implication:** in 2026 there is no open, on-prem, MongoDB-aware semantic layer. That is Caliban's opening, provided the ontology models document structure natively rather than as flattened tables.

---

## Recommendation: ontology store

**Chosen: Postgres (control plane) as the system of record, plus a compiled in-memory graph in the data plane.**

- **Storage model.**
  - `ontology_element(id, tenant, kind, version, status, body JSONB, bound_schema_hash, provenance, confidence)`.
  - `ontology_edge(src, dst, kind, card, body JSONB)`.
  - `ontology_commit(id, parent, author, message, eval_delta)`.
  - Tables are append-only, so publish is a pointer move. RLS by tenant.
  - Recursive CTEs cover the rare server-side traversal (impact analysis: "which metrics depend on `orders.lines.unitPrice`?").
- **Runtime.**
  - At publish, the CP compiles a tenant's approved elements into a typed graph (`petgraph`, interned IDs) plus retrieval cards. Both go inside the **signed config snapshot**.
  - The data plane does join-path search (Steiner tree), grain checks and policy lookups **in memory, in microseconds**, with no runtime graph DB.
  - This follows the reference architecture's rule that the data plane holds no authoritative state.
- **Edit gating (new, from EvoOntology).** Every ontology commit runs the tenant's golden set. A commit that lowers accuracy is blocked or flagged. The `eval_delta` is stored on the commit.
- **Interop.** Embed Oxigraph as a library for **import** (customer OWL/RDF ontologies such as industry models) and **export** (RDF/JSON-LD for catalogs). It is not a query path.

**Rejected:**

| Option | Why not |
|---|---|
| Neo4j Community | GPL-3.0. Another server to run on-prem. The graph is tiny. Cypher as an LLM target adds a dialect (Text2Cypher needs fine-tuning data). |
| Memgraph, SurrealDB, FalkorDB | BSL or other non-OSI licences. Redistribution in an on-prem product needs legal review. |
| Kuzu (embedded) | Archived October 2025. The LadybugDB fork is too young to depend on. |
| Apache AGE | Licence is fine, but it needs an extension in the customer's or Caliban's Postgres. On-prem DBAs often block extensions. No capability gap justifies it at 10³–10⁵ nodes. Revisit if impact-analysis queries get heavy. |
| Oxigraph / RDF as primary | Authors say evaluation is not optimized. OWL open-world semantics do not fit grain, cardinality and metric typing. LLMs are worst at SPARQL (SM3: 23–31%). The 03 decision stands: borrow OBDA *mapping* ideas, not the stack. |
| MongoDB | SSPL. Weaker multi-document transactional versioning ergonomics. Mixing the customer's DB tech with Caliban's metadata confuses support boundaries. |

---

## Recommendation: execution architecture

**Chosen: an IR-first compiler with two backends (native MongoDB pipeline + CDC-fed DataFusion/Parquet replica), chosen per query by a cost and freshness planner.**

Rejected alternatives:
- **Mongo-only native execution.** Too slow for analytics and unsafe under spill.
- **Flatten-only federation** (the DataFusion Mongo provider as-is). It is lossy and has no aggregate pushdown.
- **Free-form text-to-MQL.** Accuracy is 0.2–36% on realistic documents, and its failures are silent.

```
 NL question
   │  intent + ontology retrieval (policy-filtered)            [03 §4]
   ▼
 LLM ──emits──▶ CQIR (JSON, ontology IDs only; schema-constrained decoding)
   │            lane B: SQL over *logical entity views* → parsed → bound → CQIR
   ▼
 ┌──────────── caliban-ontology: deterministic compiler (Rust) ─────────────┐
 │ 1 validate IR (types, IDs, grain, relation paths, op/type compat)        │
 │ 2 inject policy (row predicates, column masks) into IR                   │
 │ 3 PLAN: choose backend ─ freshness SLO vs replica lag                    │
 │                         ─ explain() cost on native candidate             │
 │                         ─ replication allowed by policy?                 │
 │                         ─ cross-source? → DataFusion                     │
 └───────┬─────────────────────────────────────────┬────────────────────────┘
         │ native                                  │ accelerated / federated
         ▼                                         ▼
 compile → aggregation pipeline            compile → DataFusion LogicalPlan (SQL)
 lint (stage/op allow-list, no $-literals) over replica tables + other sources
 explain(queryPlanner) gate                (datafusion-federation pushdown)
 run: tagged secondary, maxTimeMS,                 │
      allowDiskUse:false, comment=audit_id          │
         │                                         │
         ▼                                         ▼
 Arrow RecordBatches ──────────▶ result sanity (row-count vs cardinality, null-rate,
                                 verified-query fingerprint) → answer + lineage + as_of
                                               ▲
 CDC: change streams (resume token) ─▶ BSON→Arrow (ontology-driven projection)
      ─▶ Parquet delta files + compaction (local disk / S3 / MinIO; optional Iceberg)
      watermark = last applied clusterTime → `as_of` on every accelerated answer
```

### The typed query IR (CQIR)

Example question: *"Top 5 product categories by gross revenue for EU customers in Q3 2026."* Ontology facts used:
- `Order` binds to `orders`.
- `OrderLine` is an **embedded** 1:N entity at `orders.lines[]`.
- `Customer` binds to `customers`, joined by **reference** `orders.customerId → customers._id` (verified, N:1).
- `metric.gross_revenue = SUM(OrderLine.qty * OrderLine.unit_price) WHERE Order.status IN ('paid','shipped')`, at grain `OrderLine`.
- Policy: the principal sees only `sales_org ∈ {EU-1, EU-2}`.

```json
{
  "ir_version": "1",
  "ontology_version": "acme@42",
  "metrics":    [{ "id": "metric.gross_revenue" }],
  "dimensions": [{ "id": "OrderLine.category" }],
  "filters": [
    { "attr": "Customer.region",  "op": "eq",      "value": { "type": "string", "v": "EU" } },
    { "attr": "Order.created_at", "op": "between", "half_open": true,
      "value": { "type": "timestamp", "v": ["2026-07-01T00:00:00Z", "2026-10-01T00:00:00Z"] } }
  ],
  "order_by": [{ "metric": "metric.gross_revenue", "dir": "desc" }],
  "limit": 5,
  "freshness": "PT15M"
}
```

The LLM never names a collection, a path or an operator string. The compiler adds `Order.sales_org IN ('EU-1','EU-2')` (policy) and `Order.status IN (...)` (metric definition).

**Native backend → aggregation pipeline** (options: `maxTimeMS: 15000`, `allowDiskUse: false`, `readPreference: secondary` with tag `{workload: "analytics"}`, `comment: "caliban:q_8f3a"`):

```json
[
  { "$match": { "status": { "$in": ["paid", "shipped"] },
                "createdAt": { "$gte": { "$date": "2026-07-01T00:00:00Z" }, "$lt": { "$date": "2026-10-01T00:00:00Z" } },
                "salesOrg": { "$in": ["EU-1", "EU-2"] } } },
  { "$lookup": { "from": "customers", "localField": "customerId", "foreignField": "_id", "as": "c",
                 "pipeline": [ { "$project": { "_id": 1, "region": 1 } } ] } },
  { "$unwind": "$c" },
  { "$match": { "c.region": "EU" } },
  { "$unwind": "$lines" },
  { "$group": { "_id": "$lines.category",
                "gross_revenue": { "$sum": { "$multiply": ["$lines.qty", "$lines.unitPrice"] } } } },
  { "$sort": { "gross_revenue": -1 } },
  { "$limit": 5 },
  { "$project": { "_id": 0, "category": "$_id", "gross_revenue": 1 } }
]
```

Compiler decisions visible above:
- The selective `$match` comes first so it can use an index.
- `$lookup` only on the verified relation, projecting only the fields it needs.
- `$unwind "$c"` without preserve means an inner join (N:1 required).
- `$unwind "$lines"` happens *only because* the metric grain is `OrderLine`.

**Accelerated backend → SQL over the replica's relational projection.** Embedded arrays become child tables keyed `(_parent_id, _idx)`.

```sql
SELECT l.category, SUM(l.qty * l.unit_price) AS gross_revenue
FROM   orders o
JOIN   orders__lines l ON l._parent_id = o._id
JOIN   customers     c ON c._id = o.customer_id
WHERE  o.status IN ('paid','shipped')
  AND  o.created_at >= TIMESTAMP '2026-07-01 00:00:00+00' AND o.created_at < TIMESTAMP '2026-10-01 00:00:00+00'
  AND  o.sales_org IN ('EU-1','EU-2')
  AND  c.region = 'EU'
GROUP  BY l.category
ORDER  BY gross_revenue DESC
LIMIT  5;
```

Both backends are generated from the same IR, so the two can be **shadow-compared**. A nightly job replays verified queries on both backends and alerts on a mismatch. This catches CDC bugs and drift.

### Validation steps (all deterministic, Rust)

1. **IR schema + typing.** Check:
   - every ID exists in the published ontology version;
   - operator/type compatibility (no `between` on a string enum);
   - timestamp normalization to UTC with half-open intervals.
2. **Grain and relation checks.** Join paths only along approved relations. Fan-out check: a metric at parent grain combined with a child dimension forces pre-aggregation or an error. The IR states same-element vs any-element for child filters.
3. **Policy injection.** Row predicates and column masks are added to the IR *before* backend choice, so both backends enforce the same policy. PII columns are routed to the anonymizer (05).
4. **Backend planning.**
   - `freshness < replica_lag` → native.
   - Replication forbidden → native.
   - Cross-source → DataFusion.
   - Otherwise native if `explain` shows IXSCAN and estimated examined documents ≤ `native_budget` (default 1M); else replica.
5. **Lowering + lint.**
   - Native: stage/operator allow-list, no `$`-keys or Extended JSON in literals, `$lookup` only with an indexed `foreignField`, stage count ≤ 50.
   - SQL: `sqlparser` single-`SELECT` check (03).
6. **Dry run.** Native: `explain` with `queryPlanner`. SQL: DataFusion plan stats with byte-scan limits.
7. **Execute** with timeouts, a row cap and streaming Arrow.
8. **Result sanity.** Row count vs expected cardinality, null-rate, unit checks, verified-query fingerprint. The answer carries `as_of` (replica watermark or "live").
9. **On failure,** explain the error back to the LLM in ontology terms (≤2 repair rounds), then answer "I don't know". Never fall back to raw MQL.

### CDC acceleration policy

| Rule | Default |
|---|---|
| What gets replicated | Only collections bound by **approved** entities that appear in metrics or verified queries, and only the **bound paths**. Unbound fields are not copied. This minimizes data and PII. |
| Opt-in per binding | `replicate: allowed \| forbidden \| required` on each entity binding. `forbidden` forces the native lane. The sovereign SKU defaults to `allowed` because the replica stays on the customer's disk. |
| Trigger to accelerate | Estimated documents examined above `native_budget`, **or** ≥N queries/day on the entity, **or** the metric's grain needs an unwind of an array with p99 length > 50. |
| Snapshot | Parallel by `_id` ranges from a tagged secondary, then switch to change streams from the snapshot's cluster time (Debezium pattern). |
| Updates | Upsert by `_id` into delta Parquet. A child table for a parent is rewritten on each parent update (delete-by-parent + insert). Deletes are tombstones. Compaction every 15 min or 128 MB. `fullDocument: updateLookup`, or post-images on 6.0+. |
| Storage | Default: Parquet on local disk / customer S3 / MinIO, read by DataFusion in-process. Optional: Iceberg via `iceberg-rust` (lakehouse customers) or a ClickHouse sink (>1–5 TB or high concurrency). |
| Freshness contract | `as_of` = last applied clusterTime. Metrics declare `freshness_slo`. The planner routes to native when lag > SLO. Alert when lag > 5 min. |
| Failure | If the oplog window is lost, mark the replica `stale`, route everything native (with tighter budgets), and re-snapshot in the background. |
| Requirements | A replica set or sharded cluster (not standalone). Check at connect time, otherwise the tenant gets the native lane only. |

---

## Schema inference & ontology bootstrap for document stores

**Algorithm (`Connector::introspect` + `profile` for MongoDB):**

1. **Enumerate.** Run `listCollections` (type: collection, view or timeseries, plus `options.validator`), `listIndexes` and `collStats`.
   - A `$jsonSchema` validator is treated as **declared** schema (high trust) ([docs](https://www.mongodb.com/docs/manual/core/schema-validation/)).
   - Unique indexes become candidate keys. Indexed paths become likely filter and join paths.
   - Views are resolved through their pipeline, as the Schema Builder does.
2. **Sample (stratified, on a secondary, `maxTimeMS`).**
   - (a) `$sample` with N = 1,000–10,000. Note that below 5% of the collection it uses a pseudo-random B-tree walk that "can skew" ([docs](https://www.mongodb.com/docs/manual/reference/operator/aggregation/sample/)). Mitigate by also running (b) and (c).
   - (b) The **newest** K = 500 by `_id` desc or a detected timestamp (Variety-style). This catches recent shape changes.
   - (c) The oldest K = 200.
   - (d) Per shard, if sharded.
   - **Never** take the first-N natural-order sample (the DF provider's current behaviour).
3. **Build a path tree** (port of `mongodb-schema`). For every path, including `lines[]` element paths, record:
   - presence %, null %, missing %;
   - BSON type histogram;
   - array length p50/p99, and the share of empty arrays;
   - HLL distinct count, top-k values (low cardinality), min/max;
   - semantic-type detectors: ObjectId-hex strings, ISO dates stored as strings, email, currency, GeoJSON.
4. **Classify structures.**
   - Always-present object → nested attributes.
   - Array of documents → **child entity** candidate (embedded 1:N).
   - Array of scalars → multi-valued attribute.
   - Object with many keys, each present in a small share of documents, and keys matching a pattern (dates, IDs) → **map attribute**.
   - A low-cardinality string field whose values predict which other paths are present (mutual information above a threshold) → **discriminator**, giving subtype entities.
   - Numeric widening (int32/int64/double) is auto-resolved. String vs number on one path is marked **polymorphic** and needs a human decision.
   - More than `T` polymorphic paths or more than `W` distinct top-level keys → collection flagged **unstable** (the MongoSQL concept) and kept out of lane A until curated.
5. **Discover references.**
   - Candidates: ObjectId (or hex24 string) fields, plus `*Id`/`*_id` names.
   - For each candidate, take 1,000 sampled values and run `$in` against the target `_id` (indexed, cheap) to get a containment ratio. Accept as a candidate relation at ≥0.95. Cardinality comes from distinct ratios.
   - Detect **extended references** (a subdocument holding `_id` plus copied fields) and link the copies with `canonical_of`.
6. **Mine usage.** Where the profiler (`system.profile`) or app query logs are available, extract `$group` keys and accumulators as metric candidates, and `$lookup` pairs as relation evidence (LinkedIn pattern from 03).
7. **LLM proposal pass** (per collection cluster, compact M-Schema-style cards with 3 *redacted* sample documents). The LLM proposes names, descriptions, synonyms, PII classes, subtype names and metric candidates. Annotated schemas and sample documents are the strongest prompt components in MongoDB's own eval.
8. **Human curation and eval gate.** Relations, subtypes, coercions and metrics need approval. Each commit runs the golden set (EvoOntology-style acceptance).
9. **Give back.** Optionally generate a `$jsonSchema` for approved entities so the customer can install it with warn-only validation, which stops drift at the source.
10. **Drift loop.**
    - Every 24 h, or continuously from the CDC stream at about 1% of events, re-profile the newest-K sample.
    - Diff against `bound_schema_hash`: new paths, presence shift > 10 pts, type-histogram PSI > 0.2, new discriminator values.
    - Classify each change with the U-Schema change taxonomy (add/rename/retype/embed↔reference).
    - Mark affected elements `stale` (03 cache invalidation) and queue them for curation. Trivial additive changes are auto-approved.

---

## Risks

| Risk | Mitigation |
|---|---|
| **Replica divergence** (CDC bugs, missed events, compaction errors) gives wrong answers with confidence. | Shadow-compare verified queries across backends. Track `as_of` and lag. Re-snapshot on any resume-token gap. |
| **Customers on standalone `mongod`** (no oplog), so no CDC. | Detect at connect. Native lane only with strict budgets. Recommend converting to a single-node replica set. |
| **Native queries hurting production** | Tagged secondary only, `maxTimeMS`, `explain` gate, `allowDiskUse: false`, per-tenant concurrency limit. Recommend `defaultMaxTimeMS`. |
| **Data duplication / sovereignty objections** to the replica | Bound-path-only replication, per-binding `replicate` policy, encryption at rest with tenant keys (05), crypto-shred on disconnect. |
| **Ontology misses document semantics** (array grain, same-element filters) | Compiler-enforced grain and explicit IR semantics. Golden-set cases for each trap in the table above. |
| **Benchmark optimism** | MongoDB's own 83–87% is on public sample DBs. EvoMQL and TEND numbers vary by version. Use per-tenant golden sets only (03). |
| **Spice.ai overlap** (already ships MongoDB CDC → accelerators in Rust) | Decide fork/embed/compete (architecture §9.1). Caliban's moat is the document-aware ontology + IR + policy compiler, not CDC plumbing. |
| **MongoDB 9.0+ behaviour changes** (per-operation memory cap) break native budgets | Version-aware planner. Read `serverStatus` memory metrics. Integration tests across 7.0/8.0/9.0. |
| **Licence drift of dependencies** | Pin to Apache/MIT (DataFusion, Oxigraph, mongo driver, iceberg-rust). Avoid SSPL/BSL/AGPL components in the shipped binary. |

## Open questions

1. **Native-vs-replica threshold:** `native_budget` (1M examined docs) is a guess. Benchmark on a design partner's real collections with Mongo 8.x on a tagged secondary.
2. **Replica format:** stay on plain Parquet plus Caliban's own manifest, or adopt Iceberg from day one (iceberg-rust write maturity, compaction)?
3. **Embedded DuckDB vs DataFusion** for the accelerator: DuckDB is faster in ClickBench (26 s vs 46 s) but adds C++ and a second SQL dialect. Is the speed worth it?
4. **Polymorphic coercion UX:** how much curation does an "unstable" collection need before it is useful? No published data.
5. **Do customers really have no SQL path?** Survey design partners for Enterprise Advanced licences (SQL Interface) or Atlas (analytics nodes, Data Federation `$out` → Parquet as a snapshot shortcut).
6. **Raw-MQL expert lane:** should it exist at all? If yes, gate it behind SAG-style grounding and an EvoMQL-class fine-tuned local model, and keep it off by default.
7. **Write-time copies:** when denormalized and canonical values disagree, which does the answer use by default, and do we tell the user?
8. **Contributing upstream:** fix the DataFusion Mongo provider (`$sample`, struct/list types, Decimal128, aggregate pushdown) upstream, or keep a private fork?
