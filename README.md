<p align="center"><img src="assets/logo.svg" alt="Caliban" height="56"></p>

# Caliban docs

Caliban is a single AI endpoint for a company: an OpenRouter-style gateway with an ontology layer, intent routing, agent "nodes", PII pseudonymization, token/latency savings, and a sovereign on-prem mode. The data plane is written in Rust.

## Start here
- [**Reference architecture**](architecture/caliban-reference-architecture.md): what we build, why, and in what order.
- [**Repos & conventions**](architecture/repos-and-conventions.md): the 7 repos under `~/caliban/`, ports, env vars, product decisions (BYOK only, 100% on-prem).

## Research (2026-10-02; cited literature reviews)
| # | Topic |
|---|---|
| 01 | [Semantic caching, prefix/KV reuse, Rust vector stack](research/01-semantic-caching-and-rust-vector-stack.md) |
| 02 | [Intent classification & LLM routing](research/02-intent-classification-and-routing.md) |
| 03 | [Ontology & datasource layer (text-to-SQL, semantic layers, MCP, DataFusion)](research/03-ontology-and-datasource-layer.md) |
| 04 | [Agent orchestration ("nodes"), MCP/A2A, agent security](research/04-agent-orchestration.md) |
| 05 | [Anonymization & privacy (PII, surrogates, embedding inversion, GDPR/AI Act)](research/05-anonymization-and-privacy.md) |
| 06 | [RAG & token efficiency](research/06-rag-and-token-efficiency.md) |
| 07 | [Gateway landscape, model serving, sovereign deployment](research/07-gateway-serving-and-sovereign-deployment.md) |
| 08 | [Ontology deep dive: MongoDB-as-warehouse, CQIR, ontology store](research/08-ontology-deep-dive-nosql.md) |
| 09 | [Open-weight models on-prem: Qwen lineup, licences, parsers, hardware tiers](research/09-open-models-on-prem.md) |

Every research note ends with a design recommendation and its open questions. Latency figures marked "target" are estimates, not measurements.
