<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/thecalibanproject/website/main/public/brand/logo-white.svg">
  <img src="https://raw.githubusercontent.com/thecalibanproject/website/main/public/brand/logo.svg" alt="Caliban" width="200">
</picture>

# Caliban Docs

Architecture, conventions and cited research notes behind Caliban, the sovereign AI gateway.

[core](https://github.com/thecalibanproject/core) · [web](https://github.com/thecalibanproject/web) · [ml](https://github.com/thecalibanproject/ml) · [deploy](https://github.com/thecalibanproject/deploy) · [sdk-typescript](https://github.com/thecalibanproject/sdk-typescript) · [sdk-python](https://github.com/thecalibanproject/sdk-python) · [website](https://github.com/thecalibanproject/website)

Caliban is one OpenAI- and Anthropic-compatible endpoint for a company. It classifies and routes each request, screens and pseudonymises personal data before anything reaches an outside provider, grounds answers in an ontology over the company's own datasources, caches exact and semantic matches, runs agents ("nodes"), and can serve open-weight models on-prem. It is BYOK only (upstream keys are never pooled), can run 100% on-prem with zero egress, and its core is a single Rust binary. Caliban is in active development with design partners.

## What this repo holds

| Path | Contents |
|---|---|
| [`architecture/`](architecture/) | The reference architecture and the cross-repo conventions. These describe what Caliban builds, how the repos fit together, and the product decisions taken so far. |
| [`research/`](research/) | Nine literature reviews, each with a table of key papers (with verdicts: adopt, prototype, watch, skip), open-source implementations, risks, a recommended design for Caliban, and open questions. |
| [`assets/`](assets/) | Brand logos (`logo.svg`, `logo-white.svg`, `logo-mark.svg`), identical to the ones in the [website](https://github.com/thecalibanproject/website) repo's `public/brand/`. |

The notes are plain Markdown with no frontmatter. They are research and design documents, not product documentation: the architecture is a draft, the "Rust workspace layout" and roadmap are proposals, and latency figures marked "target" are estimates, not measurements. The research snapshot date is 2026-10-02.

## Suggested reading order

1. [Reference architecture](architecture/caliban-reference-architecture.md): what Caliban is, how a request flows through it, and in what order it is built. Each decision links to the research note behind it.
2. [Repos and conventions](architecture/repos-and-conventions.md): how the code is split across repos, the product decisions, and the runtime conventions (ports, environment variables, API extensions).
3. The research notes, in numbered order, or jump to the one behind the subsystem you care about. Note 08 is a second pass on note 03, so read 03 first.

## Architecture

| File | Summary |
|---|---|
| [caliban-reference-architecture.md](architecture/caliban-reference-architecture.md) | Draft v0.1 of the whole system: the six capabilities and where Caliban differs from existing gateways, the evidence behind the design, system overview, request lifecycle, subsystem designs, the three deployment shapes of one binary, a proposed Rust workspace layout, a phased roadmap, decisions and known research gaps. |
| [repos-and-conventions.md](architecture/repos-and-conventions.md) | What each repo contains, the product decisions (BYOK only, 100% on-prem capable, licensing, dependency rules), and the runtime conventions: ports, config and environment variables, the `caliban` request extension and response headers, the Anthropic Messages API, rate limits and token budgets, virtual models, and known contract gaps. |

## Research notes

| # | File | Summary |
|---|---|---|
| 01 | [Semantic caching, prefix/KV reuse and the Rust vector stack](research/01-semantic-caching-and-rust-vector-stack.md) | Static-threshold semantic caches give false hits and shared caches leak across tenants, so semantic caching should be per-tenant, intent-gated and verified; steering provider prompt caching plus exact-match caching is the biggest cheap win. Also covers the Rust crates for embeddings, ANN, BM25 and cache tiers. |
| 02 | [Intent classification and LLM routing](research/02-intent-classification-and-routing.md) | Simple routers (tuned kNN over embeddings) match complex ones, most of the gain comes from knowing the task type, and routers are fragile; recommends a staged router (rules, kNN, small encoder, LLM fallback) with per-model quality profiles so new models can join without retraining. |
| 03 | [Ontology and datasource layer](research/03-ontology-and-datasource-layer.md) | Raw text-to-SQL on enterprise schemas performs poorly while querying a governed ontology or semantic layer does far better; recommends LLM bounded choices over a typed, versioned ontology, compiled and secured by deterministic code, with every MCP server and data value treated as untrusted. |
| 04 | [Agent orchestration ("nodes")](research/04-agent-orchestration.md) | Free-form multi-agent systems fail often, while plan-then-execute graphs are cheaper, faster and safer; recommends a typed, durable, budgeted graph runtime with every node callable over MCP and A2A, and security enforced in the Rust core. |
| 05 | [Anonymization and privacy](research/05-anonymization-and-privacy.md) | No single PII detector is enough, consistent surrogates beat placeholders, removing identifiers is not anonymization, and caches and embeddings leak; recommends a tiered Rust detector, reversible vault-backed pseudonymisation, privacy-aware routing and per-tenant cache isolation. |
| 06 | [RAG and token efficiency](research/06-rag-and-token-efficiency.md) | Hybrid retrieval with fusion and reranking, token budgets, adaptive retrieval, careful prompt compression, cache-friendly prompt layout and, on-prem, speculative decoding; also covers corpus poisoning and stale-document risks. |
| 07 | [Multi-tenant gateway, model serving and sovereign deployment](research/07-gateway-serving-and-sovereign-deployment.md) | Competitive landscape of AI gateways and the gap Caliban targets, the serving engines to integrate with rather than build (vLLM, SGLang and Kubernetes schedulers), Rust building blocks, on-prem requirements and EU AI Act timelines. |
| 08 | [Ontology deep dive: MongoDB as warehouse and the ontology store](research/08-ontology-deep-dive-nosql.md) | Keep MongoDB as the system of record but never let the LLM write MQL: the LLM emits a typed query IR (CQIR) that a Rust compiler lowers to a validated aggregation pipeline or to SQL over a CDC-fed columnar replica; the ontology itself lives in Postgres as versioned rows. |
| 09 | [Open-weight models on-prem](research/09-open-models-on-prem.md) | Which open-weight chat, embedding and reranker models an on-prem site should run, under which licence, on which engine and hardware tier; Qwen is the best-covered family, with licence flags for the models that cannot go in a default bundle. |

Each research note ends with a recommended design for Caliban and its open questions. Claims are linked to their sources inline.

## Where else these notes appear

The [website](https://github.com/thecalibanproject/website) repo renders the research notes and both architecture documents as pages. It reads them at build time from a `docs` checkout placed next to it (`../docs/research` and `../docs/architecture`), so this repo is the single source of truth; edit the notes here, not in the website.

## Licence

Copyright 2026 Elie Sfeir. All rights reserved.

This repository is **proprietary and source-available**, not open source. It is public for reference and evaluation only. No right to use, copy, modify or distribute it is granted without a separate written agreement with the copyright holder. See [LICENSE](LICENSE). For licensing, contact [elie@internalizable.dev](mailto:elie@internalizable.dev).

Papers, products and models cited in the notes belong to their respective owners.
