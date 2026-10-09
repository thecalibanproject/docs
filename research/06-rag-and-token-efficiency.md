# RAG Done Right & Token/Latency Reduction for Caliban

*Research date: 2026-10-02. Scope: how Caliban's Rust data plane should retrieve context for customer requests, and how it should cut tokens and latency per request in a way it can prove.*

**Summary.** The 2020–2026 literature agrees on six points.

1. **Retrieval quality matters more than generator effort.** Hybrid lexical + dense retrieval, fused with RRF and then reranked, is still the best value. Anthropic's contextual retrieval reported −35% failed retrievals from contextual embeddings alone, −49% with contextual BM25 added, and −67% with a reranker on top.
2. **More context is not better.** Accuracy follows an inverted U as you add chunks. Related-but-wrong "hard negatives" hurt, and evidence in the middle of the prompt gets lost. A token budget improves quality as well as cost.
3. **Retrieval should be adaptive.** Adaptive-RAG, Self-Route, CRAG and DRAG all show that routing by query complexity (no retrieval / single-shot / iterative / full long-context) keeps accuracy while cutting compute. This is the same decision as Caliban's intent classifier and model router.
4. **Prompt compression is real but risky.** LLMLingua-family and RECOMP reach 2–20× compression with small accuracy loss on benchmarks. Later studies show lost details, worse citation grounding (abstractive) and gains that depend on the task.
5. **Caching wins first.** Provider prefix caching charges 0.1× (or less) for cache reads, and KV reuse for RAG chunks (RAGCache, CacheBlend, TurboRAG) cuts TTFT 2–9×. Both depend on how the prompt is laid out, which Caliban controls.
6. **On-prem latency.** For sovereign deployments, speculative decoding (EAGLE-3: 3–6.5×) and distilled task-specific small models give the largest latency gains.

The main risks are corpus poisoning (PoisonedRAG reaches 97% attack success with 5 injected texts), stale documents overriding correct answers (17–91% flips), and semantic-cache poisoning.

---

## Key papers

### RAG

| Paper | Year | Core idea | Reported gain | Why it matters for Caliban | Verdict |
|---|---|---|---|---|---|
| [RAG (Lewis et al.)](https://arxiv.org/abs/2005.11401) | 2020 | Seq2seq generator + dense Wikipedia index via neural retriever (parametric + non-parametric memory). | SOTA on 3 open-domain QA tasks; more specific/factual generations. | Foundational framing: the "ontology layer" is Caliban's non-parametric memory. | **Adopt** (concept) |
| [DPR](https://arxiv.org/pdf/2004.04906) | 2020 | Dual-encoder dense retrieval trained on Q/passage pairs. | +9–19 pts absolute top-20 accuracy vs Lucene BM25. | Baseline dense path; shows that dense retrieval needs domain-matched training, so keep BM25 alongside. | **Adopt** (as one leg of hybrid) |
| [ColBERTv2](https://arxiv.org/pdf/2112.01488) (orig. [ColBERT](https://arxiv.org/abs/2004.12832)) | 2021/22 | Late interaction: per-token vectors, MaxSim scoring; residual compression. | 6–10× smaller index than ColBERTv1; best on 22/28 out-of-domain tests (up to +8% relative). | Strong zero-shot quality and a cheaper reranker than a cross-encoder. Storage cost is the catch. | **Prototype** (as reranker / prefetch rescoring) |
| [Jina-ColBERT-v2](https://arxiv.org/abs/2408.16672) | 2024 | Multilingual ColBERT with efficiency tweaks. | Strong EN + multilingual retrieval. | Candidate multilingual late-interaction model; **weights are CC BY-NC** (see Risks). | **Watch** (license) |
| [Reciprocal Rank Fusion](https://research.google/pubs/reciprocal-rank-fusion-outperforms-condorcet-and-individual-rank-learning-methods/) | 2009 | Fuse rankings by Σ 1/(k+rank). | Beats any individual system and Condorcet fuse. | Parameter-light fusion of BM25 + dense; no score calibration needed. Easy to implement in Rust. | **Adopt** |
| [BGE M3-Embedding](https://arxiv.org/abs/2402.03216) | 2024 | One model emits dense, sparse and multi-vector representations; 100+ languages, 8k input. | SOTA multilingual / cross-lingual / long-doc retrieval (at release). | One model can feed all three index types; available in fastembed-rs. Good on-prem default. | **Adopt** (default open embedder candidate) |
| [Matryoshka Representation Learning](https://arxiv.org/abs/2205.13147) | 2022 | Nested embeddings: prefixes of the vector are usable embeddings. | Up to 14× smaller embeddings at equal accuracy; up to 14× retrieval speed-ups. | Two-stage ANN: search on truncated (e.g. 256-d) vectors, rescore with full vectors. Cuts RAM for multi-tenant indexes. | **Adopt** |
| [HyDE](https://arxiv.org/abs/2212.10496) | 2022 | LLM writes a hypothetical answer; embed it for retrieval. | Beats unsupervised Contriever; comparable to fine-tuned retrievers. | Adds an LLM call before retrieval, so the latency cost is high. Use it only as a fallback for zero-hit / low-confidence queries. | **Prototype** (fallback only) |
| [Rewrite-Retrieve-Read](https://arxiv.org/abs/2305.14283) | 2023 | Small trainable query rewriter (RL from reader feedback). | Consistent QA gains over retrieve-then-read. | Conversational queries need rewriting into standalone queries. Do it with the small model already used for intent classification. | **Adopt** (small-model rewrite) |
| [Self-RAG](https://arxiv.org/pdf/2310.11511) | 2023 | LM trained to emit reflection tokens: when to retrieve, and whether output is supported. | 7B/13B beat larger LLMs on QA, fact verification, citation. | Needs a fine-tuned generator, so it doesn't fit routing to closed APIs. Borrow the idea (retrieve on demand, critique support), not the model. | **Watch** |
| [CRAG](https://ar5iv.labs.arxiv.org/html/2401.15884) | 2024 | Lightweight evaluator grades retrieval as Correct / Ambiguous / Incorrect and triggers different actions (refine, web search). | Significant gains over RAG and Self-RAG on 4 datasets; plug-and-play. | Maps onto a cheap reranker-score threshold plus a fallback policy (escalate / abstain / web). Implementable without an LLM. | **Adopt** (pattern) |
| [Adaptive-RAG](https://arxiv.org/pdf/2403.14403) | 2024 | Small classifier predicts query complexity and picks no-retrieval / single-step / multi-step. | Better accuracy-efficiency trade-off than fixed strategies. | **Same mechanism as Caliban's intent router.** Add a retrieval-strategy head to it. Online variant: [MBA-RAG](https://arxiv.org/abs/2412.01572) (bandit that penalizes extra retrieval steps). | **Adopt** |
| [DRAGIN](https://arxiv.org/abs/2403.10081) / [FLARE](https://arxiv.org/abs/2305.06983) | 2023/24 | Retrieve mid-generation when the model's information need or uncertainty spikes. | Better long-form QA than static RAG. | Needs token-level logits/attention during decoding. Possible only on self-hosted models. | **Watch** |
| [DRAG: Dynamic Retriever & Generator Selection](https://arxiv.org/abs/2609.17709) | 2026 | Route jointly over retriever strength and generator size using query-performance prediction. | Matches strong static RAG at much lower inference cost. Stronger retrieval beats more generation effort. | Direct evidence for joint retrieval + model routing in one decision. Very new preprint. | **Prototype** |
| [RAPTOR](https://arxiv.org/pdf/2401.18059) | 2024 | Recursively cluster and summarize chunks into a tree; retrieve across levels. | Large gains on long-document QA (ICLR'24). | Useful for "summarize this corpus" questions. Costly offline LLM summarization; index rebuilds on change. | **Prototype** (opt-in index type) |
| [GraphRAG](https://arxiv.org/abs/2404.16130) | 2024 | LLM-built entity graph + community summaries for global sensemaking queries. | Better comprehensiveness/diversity on ~1M-token corpora. | Overlaps with the ontology layer. Expensive to build and keep fresh. | **Watch** |
| [Contextual Retrieval (Anthropic)](https://www.anthropic.com/engineering/contextual-retrieval) | 2024 | Prepend a 50–100-token LLM-written context to each chunk before embedding and BM25. | Failed top-20 retrievals: −35% (embeddings), −49% (+BM25), −67% (+rerank). ~$1.02/M doc tokens with prompt caching (2024 pricing). If KB < 200k tokens, skip RAG and put it all in the prompt. | Cheapest large quality gain available at ingest. Batch it with prompt caching on the doc prefix. | **Adopt** |
| [Late Chunking](https://arxiv.org/pdf/2409.04701) | 2024 | Embed the whole doc with a long-context embedder, then pool per chunk. | Better retrieval than naive chunking, no extra training. | Gets context into chunk vectors without an LLM call. Needs token-level outputs from a self-hosted long-context embedder (feasible with ort/candle). | **Prototype** (on-prem alternative to contextual retrieval) |
| [Chroma: Evaluating Chunking](https://research.trychroma.com/evaluating-chunking) | 2024 | Token-level recall/precision eval of chunkers. | Up to 9% recall difference between strategies. Recursive splitter at ~200 tokens, no overlap, is consistently strong. | Default chunker should be cheap and recursive. Make chunking a measured, tenant-tunable parameter. | **Adopt** |
| [Is Semantic Chunking Worth the Cost?](https://arxiv.org/abs/2410.13070) | 2024 | Systematic comparison of semantic vs fixed-size chunking. | Semantic chunking's cost "not justified by consistent gains." | Don't run embedding-based semantic chunking by default. | **Skip** (as default) |
| [Lost in the Middle](https://arxiv.org/abs/2307.03172) | 2023 | Accuracy is U-shaped in evidence position. | Big drops when evidence sits mid-context. | Put the best chunks at the start and end. Keep the context short. | **Adopt** |
| [The Power of Noise](https://arxiv.org/abs/2401.14887) | 2024 | Related-but-non-answering docs hurt; random docs can help. | Random docs improved accuracy up to 35% in their setup. | Hard negatives in the top-k are harmful, which argues for rerank + cutoff over a large fixed k. | **Watch** (counter-intuitive; validate) |
| [Long-Context LLMs Meet RAG](https://arxiv.org/abs/2410.05983) | 2024 | More retrieved passages: quality rises, then falls (hard negatives). | Retrieval reordering is a strong training-free fix. | Supports a budgeted top-k with reordering. | **Adopt** |
| [OP-RAG (In Defense of RAG)](https://arxiv.org/abs/2409.01666) | 2024 | Keep retrieved chunks in original document order. | Inverted-U curve; sweet spot beats full long-context with far fewer tokens. | Cheap ordering rule for same-doc chunks; tune k per tenant. | **Adopt** |
| [RAG vs Long-Context + Self-Route](https://arxiv.org/pdf/2407.16833) | 2024 | LC beats RAG when resourced; route per query via model self-reflection. | Comparable to LC at much lower cost. | "Try RAG first, escalate to full context if the model says unanswerable" is a cheap router policy. | **Adopt** |
| [Long Context vs RAG: Evaluation & Revisits](https://arxiv.org/abs/2501.01880) | 2025 | Re-eval filtering questions answerable without context. | LC wins on Wikipedia QA; summarization-based retrieval ≈ LC; chunk RAG lags; RAG wins on dialogue/general queries. | Evidence for RAPTOR-style summary nodes plus routing by query type. | **Adopt** (as routing evidence) |
| [LaRA](https://arxiv.org/abs/2502.09977) | 2025 | 2,326-case benchmark comparing RAG vs LC across 11 LLMs. | "No silver bullet": the best choice depends on model, context length, task and chunk traits. | The RAG-vs-LC decision should be learned per (model, task), not hard-coded. | **Adopt** (as routing evidence) |
| [Sufficient Context](https://arxiv.org/abs/2411.06037) | 2024 | Classify whether retrieved context suffices; use it for selective abstention. | +2–10% accuracy among answered queries. Big models answer wrongly instead of abstaining when context is insufficient. | A sufficiency signal is the trigger for escalate / abstain decisions in the CRAG-style policy. | **Prototype** |
| [Speculative RAG](https://arxiv.org/abs/2407.08223) | 2024 | Small specialist drafts answers from document subsets in parallel; big model verifies. | Up to +12.97% accuracy and −50.83% latency (PubHealth). | Fits a router that owns both small and large models. | **Prototype** |
| [Agentic RAG survey](https://arxiv.org/pdf/2501.09136) | 2025 | Taxonomy of agent-driven retrieval (reflection, planning, tool use, multi-agent). | Survey. | Vocabulary for "agentic retrieval" node types; see doc 04 for orchestration. | **Watch** |
| [Search-R1](https://arxiv.org/abs/2503.09516) | 2025 | RL-train an LLM to interleave reasoning and search calls. | +41% (Qwen2.5-7B) / +20% (3B) over RAG baselines. | Shows trained search agents beat prompted ones. Relevant if Caliban ships a fine-tuned retrieval agent for on-prem. | **Watch** |
| [PACE: Evidence Frontloading & Pressure-Adaptive Budgeting](https://arxiv.org/abs/2608.25115) | 2026 | Under load the bottleneck moves to the reranker; adapt rerank budget to load; submodular evidence coverage. | Better evidence recall, lower p95 under load. | The reranker budget must adapt to load in the Rust scheduler. | **Prototype** |
| [RAGAS](https://arxiv.org/abs/2309.15217) | 2023 | Reference-free metrics: context relevance, faithfulness, answer relevance. | No ground-truth labels needed. | Online quality guardrail for "tokens saved without quality loss." | **Adopt** (offline/shadow eval) |
| [ARES](https://aclanthology.org/2024.naacl-long.20/) | 2024 | Fine-tuned lightweight LM judges + prediction-powered inference with a few hundred human labels. | Accurate across 8 KILT/SuperGLUE/AIS tasks. | Cheaper, calibrated judges for per-tenant evals. | **Prototype** |
| [RAGChecker](https://arxiv.org/abs/2408.08067) | 2024 | Fine-grained claim-level diagnostics for retriever vs generator. | Better human correlation than prior metrics. | Tells customers whether a failure came from retrieval or generation. | **Prototype** |

### Token & latency reduction

| Paper | Year | Core idea | Reported gain | Why it matters for Caliban | Verdict |
|---|---|---|---|---|---|
| [Selective Context](https://arxiv.org/abs/2310.06201) | 2023 | Drop low self-information lexical units using a small causal LM. | −50% context → −36% memory, −32% inference time, minor metric drops. | Simple baseline; superseded by LLMLingua-2. | **Skip** |
| [LLMLingua](https://arxiv.org/pdf/2310.05736) | 2023 | Coarse-to-fine budget controller + iterative token-level pruning with small LM. | Up to 20× compression with little loss (GSM8K, BBH, etc.). | Shows how far compression can go; the small-LM perplexity pass costs latency. | **Watch** |
| [LongLLMLingua](https://arxiv.org/abs/2310.06839) | 2023/ACL'24 | Question-aware compression + document reordering + dynamic ratios. | +17.1% on NQ with ~4× fewer tokens; 1.4–2.6× end-to-end speed-up at 2–6× on ~10k prompts. | Most relevant for RAG contexts; query-aware. | **Prototype** |
| [LLMLingua-2](https://arxiv.org/abs/2403.12968) | 2024 | Distilled token-classification compressor (XLM-RoBERTa / mBERT encoder). | 3–6× faster than prior compressors; 1.6–2.9× E2E latency at 2–5× compression. | Encoder-only and MIT-licensed weights, so it can be exported to ONNX and run in Rust via `ort`. Best fit for an in-gateway compressor. | **Prototype → Adopt** behind a flag |
| [RECOMP](https://arxiv.org/html/2310.04408) | 2023/ICLR'24 | Extractive + abstractive compressors trained on end-task; can return empty (selective augmentation). | Down to 6% of tokens with minimal loss. | "Return empty when docs are useless" is free adaptive retrieval. Abstractive mode hurts attribution (see Risks). | **Prototype** (extractive only) |
| [xRAG](https://arxiv.org/abs/2405.13792) | 2024/NeurIPS | Feed the doc embedding as one soft token via a trained modality bridge. | >10% avg improvement over no-retrieval baselines on 6 tasks at extreme compression. | Needs a model-side adapter, so it doesn't apply to closed APIs. On-prem only. | **Watch** |
| [Provence](https://arxiv.org/abs/2501.16214) | 2025 | Sentence-level context pruner unified with reranking (DeBERTa-v3, 430M). | Negligible-to-no accuracy drop across domains at almost no added cost. | Rerank and prune in one encoder pass is ideal for the gateway, but the [weights are CC BY-NC 4.0](https://huggingface.co/naver/provence-reranker-debertav3-v1). Train our own on the same recipe. | **Prototype** (own weights) |
| [Empirical Study on Prompt Compression](https://arxiv.org/abs/2505.00019) | 2025 | 6 methods × 13 datasets incl. hallucination analysis. | Compression hurts more on short contexts. Moderate compression can help on LongBench. | Enable compression only above a length threshold. | **Adopt** (as policy evidence) |
| [Information Preservation in Prompt Compression](https://arxiv.org/abs/2503.19114) | 2025 | Evaluate compressors on grounding and entity preservation, not just task score. | Some methods lose key details. Fix gives +23% downstream, 2.7× more entities preserved. | Measure entity/number preservation as a guardrail metric. | **Adopt** (metric) |
| [Prompt Cache](https://arxiv.org/abs/2311.04934) / [SGLang RadixAttention](https://arxiv.org/abs/2312.07104) / [vLLM PagedAttention](https://arxiv.org/abs/2309.06180) | 2023 | Reuse attention/KV states for shared prefixes (radix tree); paged KV memory. | SGLang up to 6.4× throughput; vLLM 2–4× throughput. | On-prem "sovereign router" should run on vLLM/SGLang with prefix caching. Caliban's job is to make prefixes shareable. | **Adopt** (serving layer) |
| Provider prompt caching ([Anthropic](https://platform.claude.com/docs/en/build-with-claude/prompt-caching), [OpenAI](https://developers.openai.com/api/docs/guides/prompt-caching)) | 2024–26 | Pay less for repeated exact prefixes. | Anthropic: writes 1.25× (5 min) / 2× (1 h), reads 0.1× (0.05× or 0.025× on some current models), 4 breakpoints, min 512–4,096 tokens by model; cache hits don't count against rate limits. OpenAI: cached reads up to 95% off; ≥1,024-token prefix; `prompt_cache_key`. | **Largest, risk-free saving.** The gateway should own prompt layout (stable → volatile), insert breakpoints, and report `cache_read_input_tokens` / `cached_tokens`. | **Adopt** |
| [RAGCache](https://arxiv.org/abs/2404.12457) | 2024 | Multilevel GPU/host cache of retrieved-doc KV states in a knowledge tree; overlap retrieval and inference. | TTFT up to 4×, throughput up to 2.1× vs vLLM+Faiss. | Design reference for on-prem chunk-KV caching. | **Prototype** (on-prem) |
| [CacheBlend](https://arxiv.org/abs/2405.16444) | 2024 | Reuse per-chunk KV caches at non-prefix positions; selectively recompute a few tokens. | TTFT −2.2–3.3×, throughput 2.8–5×, no quality loss. Code in [LMCache](https://github.com/LMCache/LMCache). | Makes RAG chunk caches reusable in any order. Pair with vLLM on-prem. | **Prototype** (on-prem) |
| [TurboRAG](https://arxiv.org/abs/2410.07590) | 2024 | Precompute and store doc KV caches offline; fine-tune for mask/position fixes. | TTFT up to 9.4× (avg 8.6×). | Needs a fine-tuned model. Interesting for high-QPS sovereign tenants. | **Watch** |
| [Speculative Decoding](https://arxiv.org/abs/2211.17192) | 2022/ICML'23 | Draft tokens with a small model, verify in parallel; output distribution unchanged. | 2–3× (paper's T5 results). | Lossless decode speed-up for self-hosted models. | **Adopt** (on-prem) |
| [Medusa](https://arxiv.org/pdf/2401.10774) | 2024 | Extra decoding heads + tree attention, no separate draft model. | 2.3–2.8×. | Superseded by EAGLE-3 for most models. | **Skip** |
| [EAGLE-3](https://arxiv.org/abs/2503.01840) (and [EAGLE-2](https://arxiv.org/abs/2406.16858)) | 2025 | Token-level drafting with multi-layer feature fusion; dynamic draft trees. | Up to 6.47× (HumanEval), ~5.5× mean; EAGLE-2 3.05–4.26×. Lossless. | Default speculative method for vLLM/SGLang deployments. Train drafters for the on-prem models customers use most. | **Adopt** (on-prem) |
| [TALE: Token-Budget-Aware Reasoning](https://arxiv.org/pdf/2412.18547) | 2024/25 | Estimate per-problem token budget and put it in the prompt (or post-train). | −67% tokens with <3% accuracy drop (EP); ~−50% (PT). | Per-intent output budgets in the system prompt are a cheap, measurable win. | **Adopt** |
| [Chain of Draft](https://arxiv.org/abs/2502.18600) | 2025 | Minimal intermediate reasoning drafts instead of verbose CoT. | Matches CoT using as little as 7.6% of tokens. | A gateway-level "concise reasoning" prompt policy for non-reasoning models. Related: [Concise Thoughts / CCoT](https://arxiv.org/abs/2407.19825), [overthinking in o1-like models](https://arxiv.org/abs/2412.21187). | **Prototype** (per-tenant A/B) |
| [Distilling Step-by-Step](https://arxiv.org/abs/2305.02301) | 2023 | Distill LLM rationales into small task models. | 770M T5 beats few-shot 540B PaLM. >50% less training data. | Repeated tenant tasks (classification, extraction, routing) should move to distilled small models trained on gateway logs, with consent. | **Adopt** (pipeline) |
| [FinCacheServe](https://arxiv.org/abs/2607.26076) | 2026 | Answer cache keyed by intent and guarded by doc versions, evidence/tool fingerprints, model and decode config. | Skips ~53% of LLM calls with zero observed stale outputs, vs 39% for versioned semantic caching. | The right design for Caliban's answer cache over mutable enterprise data. | **Adopt** (design) |

---

## Open-source implementations & Rust crates

Versions from crates.io / GitHub as of 2026-10-02.

| Component | Project | Lang / License | Use in Caliban | Notes |
|---|---|---|---|---|
| BM25 / full-text | [tantivy](https://github.com/quickwit-oss/tantivy) (crate 0.26) | Rust / MIT | Contextual-BM25 leg of hybrid search; per-tenant indexes | Lucene-like, embeddable, mature (16k★). Index the *contextualized* chunk text. |
| Local embeddings, sparse, rerank | [fastembed-rs](https://github.com/Anush008/fastembed-rs) (crate 7.1) | Rust / Apache-2.0 | On-prem embeddings (BGE, BGE-M3, Nomic, E5, EmbeddingGemma, Qwen3), SPLADE/BGE-M3 sparse, rerankers (bge-reranker-base/v2-m3, Jina) | Uses `ort`; candle backend for some models; quantized variants. Check each model's license. |
| ONNX inference | [ort](https://github.com/pykeio/ort) (2.0.0-rc.13) | Rust / Apache-2.0 | Cross-encoder rerankers, LLMLingua-2 compressor, intent classifier | Production-grade, CPU/GPU EPs. |
| Native ML | [candle](https://github.com/huggingface/candle) (0.11) | Rust / Apache-2.0 | Late-chunking token outputs; models without clean ONNX export | Pure Rust, no Python. Fewer optimized kernels than ORT. |
| Tokenizers | [tokenizers](https://github.com/huggingface/tokenizers), [tiktoken-rs](https://github.com/zurawiki/tiktoken-rs) | Rust / Apache-2.0, MIT | Exact per-model token counting for the budget manager | Precompute chunk token counts per tokenizer family at ingest. |
| Chunking | [text-splitter](https://github.com/benbrandt/text-splitter) (0.33) | Rust / MIT | Default chunker: text, Markdown (heading-aware), code (tree-sitter) | Sizes by tokens through tiktoken-rs or HF tokenizers. |
| Vector DB | [Qdrant](https://github.com/qdrant/qdrant) + [rust-client](https://github.com/qdrant/rust-client) | Rust / Apache-2.0 | Primary vector store; [Query API](https://qdrant.tech/documentation/concepts/hybrid-queries/) does dense+sparse fusion (RRF, DBSF) and multi-stage prefetch with multivector (ColBERT) rescoring | Matryoshka-style "small vector prefetch → full rescore" is native. |
| Embedded vector store | [LanceDB](https://github.com/lancedb/lancedb) (0.39) | Rust / Apache-2.0 | Single-binary sovereign deployments; edge | Embedded, columnar, versioned (helps index generations). |
| ANN library | [USearch](https://github.com/unum-cloud/USearch), [arroy](https://github.com/meilisearch/arroy) | C++ with Rust bindings / Apache-2.0; Rust / MIT | Semantic-cache index, small in-process indexes | For the L1 cache, avoiding a network hop. |
| Embedding server | [text-embeddings-inference](https://github.com/huggingface/text-embeddings-inference) | Rust / Apache-2.0 | GPU embedding + rerank sidecar for high QPS | Rust server; good for GPU on-prem. |
| Rust RAG frameworks | [swiftide](https://github.com/bosun-ai/swiftide), [rig](https://github.com/0xPlaygrounds/rig) | Rust / MIT | Reference designs for streaming ingest pipelines | Learn from them; keep Caliban's pipeline in-house for control over budgets and metrics. |
| Structured outputs | [llguidance](https://github.com/guidance-ai/llguidance) (crate 1.9) | Rust / MIT | Grammar-constrained decoding on self-hosted models; JSON-schema outputs cut retries and verbose prose | Used inside vLLM/SGLang. |
| Prompt compression | [LLMLingua](https://github.com/microsoft/LLMLingua) | Python / MIT (LLMLingua-2 weights MIT) | Export LLMLingua-2 encoder to ONNX and run with `ort` | Don't call Python in the hot path. |
| Late interaction | [ColBERT](https://github.com/stanford-futuredata/ColBERT), [PyLate](https://github.com/lightonai/pylate) | Python / MIT | Train or evaluate ColBERT models offline | Serve via Qdrant multivector. |
| KV-cache reuse | [LMCache](https://github.com/LMCache/LMCache) (CacheBlend) | Python / Apache-2.0 | On-prem chunk-KV reuse with vLLM | Sidecar to serving, not the gateway. |
| Serving | [vLLM](https://github.com/vllm-project/vllm), [SGLang](https://github.com/sgl-project/sglang), [EAGLE](https://github.com/SafeAILab/EAGLE) | Python / Apache-2.0 | Sovereign router backends with prefix caching + EAGLE-3 speculative decoding | |
| RAG eval | [Ragas](https://github.com/vibrantlabsai/ragas), [ARES](https://github.com/stanford-futuredata/ARES), [RAGChecker](https://github.com/amazon-science/RAGChecker), [FlashRAG](https://arxiv.org/abs/2405.13576) | Python | Offline/shadow eval harness; FlashRAG has 16 RAG methods for bake-offs | Run out of band. Results feed the savings ledger. |
| Reference impls | [RAPTOR](https://github.com/parthsarthi03/raptor), [CRAG](https://github.com/HuskyInSalt/CRAG), [Adaptive-RAG](https://github.com/starsuzi/Adaptive-RAG), [late-chunking](https://github.com/jina-ai/late-chunking), [HyDE](https://github.com/texttron/hyde) | Python | Port the logic, not the code | |

---

## Risks & failure modes

| Risk | Evidence | Mitigation in Caliban |
|---|---|---|
| **Compression drops facts and breaks citations** | Some compressors fail to keep key details and entities ([Info Preservation](https://arxiv.org/abs/2503.19114)). Compression hurts more on short contexts ([Empirical Study](https://arxiv.org/abs/2505.00019)). RECOMP-style abstractive compression: citation precision 0.86 against its summaries but **0.12 against source spans**; 0.88 unsupported claims after source recovery ([Attribution-Compression Frontier](https://arxiv.org/abs/2609.14245), 2026 preprint). | Compression off by default. Turn it on only above a length threshold, extractive only. Never compress code, JSON, tables, numbers or quoted citations. Guardrail metric: entity/number preservation rate. Keep chunk IDs so citations map to the source. |
| **Compression vs caching conflict** (our inference) | Provider caches need byte-identical prefixes ([Anthropic docs](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)). Query-aware compression produces different bytes on every request. | Compress only the volatile suffix (retrieved chunks). Never compress the cached prefix. Make compression deterministic per (chunk, query-class). |
| **Too much or wrong context lowers quality** | U-shaped position effect ([Lost in the Middle](https://arxiv.org/abs/2307.03172)). Hard negatives cause an inverted U ([LC meets RAG](https://arxiv.org/abs/2410.05983), [OP-RAG](https://arxiv.org/abs/2409.01666)). Related-but-wrong docs hurt ([Power of Noise](https://arxiv.org/abs/2401.14887)). | Rerank, then cut by score and budget rather than a fixed k. Reorder: best first, second-best last. Same-doc chunks stay in document order. |
| **Overconfident answers on insufficient context** | Frontier models answer instead of abstaining when context is insufficient ([Sufficient Context](https://arxiv.org/abs/2411.06037)). | CRAG-style confidence bands from reranker scores, plus an optional sufficiency check, lead to abstain / escalate / ask a clarifying question. |
| **Corpus poisoning** | 5 injected texts → 97% attack success in a 2.6M-doc corpus; tested defenses insufficient ([PoisonedRAG](https://arxiv.org/pdf/2402.07867)). Retrieved text carries indirect prompt injection ([Greshake et al.](https://arxiv.org/abs/2302.12173)). | Provenance and trust tiers per source. Per-tenant isolation of indexes. Retrieved text is always quoted as data, never as instructions. Anomaly checks on near-duplicate high-similarity inserts. Pair with the injection defenses in doc 04. |
| **Semantic-cache poisoning** | An attacker caches a malicious answer under a query that is highly similar to benign queries ([Similarity Is Not Validity](https://arxiv.org/abs/2609.35908), 2026). | Scope the answer cache per tenant and per ACL. Verify hits beyond cosine similarity (text check, dependency fingerprint). Never share answers across tenants. |
| **Stale indexes / stale documents** | Outdated retrieved docs flip correct answers in 30–37% of cases (17–91% across models/domains) ([Stale-Document Poisoning](https://arxiv.org/abs/2609.31342), 2026). LLMs struggle on fast-changing facts ([FreshLLMs](https://arxiv.org/abs/2310.03214)). Context-memory conflicts ([Knowledge Conflicts survey](https://arxiv.org/abs/2403.08319)). | CDC-driven re-ingest with an index generation number. `valid_from`/`valid_to` and `superseded_by` metadata, injected into the prompt as explicit validity statements (the paper shows telling the model when evidence stops applying fixes most errors). Freshness SLOs per connector. |
| **Stale caches after edits** | Answer caches over mutable docs need dependency consistency ([FinCacheServe](https://arxiv.org/abs/2607.26076)). KV caches go stale downstream of edits; edit-local repair is 13–21× faster than re-prefill ([Stale KV repair](https://arxiv.org/abs/2609.17983), 2026). | Every cached artifact (answer, retrieval result, chunk KV) is keyed by doc-version and index-generation fingerprints. Invalidate on CDC events. |
| **Reranker becomes the bottleneck under load** | Bottleneck moves from LLM to reranker at high QPS ([PACE](https://arxiv.org/abs/2608.25115)). | Load-adaptive rerank depth (e.g. 50 → 20 candidates). Separate CPU pool for rerank. |
| **Model licenses** | Provence and Jina-ColBERT-v2 weights are CC BY-NC 4.0 (checked on Hugging Face). | Maintain an allow-list of commercially licensed models (e.g. bge-reranker-v2-m3 Apache-2.0, LLMLingua-2 MIT). |
| **Eval judges are noisy** | RAGAS is reference-free (LLM judge). ARES still needs a few hundred human labels for calibration. | Report savings together with a quality delta and confidence interval. Calibrate judges per tenant with a small labeled set. |
| **LC-vs-RAG findings conflict** | LC beats RAG when resourced ([Self-Route](https://arxiv.org/pdf/2407.16833)); "no silver bullet" ([LaRA](https://arxiv.org/abs/2502.09977)). | Make it a routing decision learned per (model, task, corpus size). Don't hard-code it. |

---

## Recommended design for Caliban

### 1. Default pipeline (all hot-path stages in Rust)

```
INGEST (async, Rust workers)
 connector → parse/normalize → PII tagging (doc 05) → chunk → contextualize → embed → index
QUERY (hot path, Rust)
 intent+strategy classify → [skip?] → rewrite → hybrid retrieve → RRF → rerank → budget-fit
 → (prune/compress) → order → assemble (cache-aware) → route+generate → stream → log/measure
```

| Stage | Default | Rust? | Notes |
|---|---|---|---|
| Parse / normalize | Format-specific parsers; keep structure (headings, tables, code). | Yes (with sidecars for PDF/OCR) | Store `doc_id`, `version`, `valid_from/to`, ACLs, source trust tier. |
| Chunk | `text-splitter`: Markdown/code-aware, **~256–512 tokens, no overlap**, sized by the target embedder's tokenizer. | Yes | Per Chroma study and semantic-chunking critique. Tenant-tunable; measured by recall@k. |
| Contextualize | Anthropic-style 50–100-token chunk context from a cheap model, batching all chunks of a doc behind a cached doc prefix. | Orchestrated in Rust; LLM via Caliban's own router | On-prem / no-LLM option: **late chunking** with a self-hosted long-context embedder (candle/ort). Re-run only for changed docs. |
| Embed | Hosted embedder via router, or on-prem **BGE-M3 via fastembed-rs/ort** (dense + sparse). Matryoshka-capable models preferred. | Yes | Store full vector + truncated (e.g. 256-d) vector; int8/binary quantization for prefetch. |
| Index | **tantivy BM25 over contextualized text** + **Qdrant** (or LanceDB embedded for single-node sovereign). Per-tenant collections; index generation counter. | Yes | Qdrant prefetch: truncated vector → full-vector rescore → optional ColBERT multivector. |
| Retrieve + fuse | BM25 top-50 ‖ dense top-50 (parallel, tokio) → **RRF (k≈60)** → top-50. | Yes | Run BM25 and ANN concurrently; fusion is microseconds. |
| Rerank | Cross-encoder **bge-reranker-v2-m3 (Apache-2.0) via ort**, load-adaptive depth 20–50. | Yes | Scores feed CRAG confidence bands and the budget manager. |
| Prune / compress | **Off by default.** If retrieved context > budget and > ~2k tokens: extractive sentence pruning (own Provence-style model or LLMLingua-2 via ort). Never touch code, tables, numbers. | Yes | Abstractive summaries (RECOMP/RAPTOR) only as index-time nodes, never per request. |
| Order | Highest-scoring chunk first, second-highest last. Chunks from the same doc in document order (OP-RAG). | Yes | |
| Assemble | **Stable → volatile layout**: [system + tool schemas + tenant static instructions] ⟶ cache breakpoint ⟶ [long-lived tenant context] ⟶ breakpoint ⟶ [retrieved chunks with IDs + validity dates] ⟶ [history summary + last turns] ⟶ [user query]. | Yes | Byte-stable serialization (sorted keys, fixed whitespace) so prefixes cache. Set `prompt_cache_key` per tenant. |
| Generate | Router picks model. Per-intent `max_tokens` and conciseness instruction (TALE/CoD style). JSON-schema structured output where the caller wants data. | Router in Rust | On-prem: vLLM/SGLang + prefix caching + EAGLE-3 + LMCache/CacheBlend. |

### 2. Adaptive retrieval (the "skip retrieval" decision)

Extend the intent classifier (a small encoder via ort, run once per request) with a **retrieval-strategy head**:

| Strategy | When | Cost |
|---|---|---|
| `none` | Chit-chat, transformations of user-supplied text, code edits on given code, general-knowledge queries the routed model handles well. | 0 retrieval tokens |
| `full_context` | Tenant corpus (or the selected doc set) < ~100–200k tokens and the model has a long context. Put it in the cached prefix (Anthropic's <200k guidance + prompt caching). | High first call, then 0.1× reads |
| `single_shot` | Default factual / lookup queries. | 1 retrieval |
| `iterative` / agentic | Multi-hop, comparison, "across all X" queries → planner node (doc 04), RAPTOR/summary index. | N retrievals, budgeted |

Runtime corrections (CRAG + Self-Route):
- top rerank score ≥ τ_high → answer from the chunks;
- τ_low < score < τ_high → add the query-rewrite / HyDE fallback, then retry once;
- score ≤ τ_low → escalate (`full_context` if it fits, otherwise web or abstain).

If the model returns an "unanswerable" signal on `single_shot`, escalate once (Self-Route). Learn τ and strategy priors per tenant online with a bandit that pays for accuracy and penalizes tokens and latency (MBA-RAG / DRAG idea). Train labels come from shadow evals.

### 3. Per-request token budget manager

- **Budget** = min(tenant policy, model context − output reserve, cost ceiling ÷ price). The router knows the price table.
- **Partitions** (defaults, tenant-overridable): cached prefix (not counted against the cost ceiling at 0.1×) | retrieved context ≤ 40% | history ≤ 20% (older turns summarized and cached) | output reserve by intent class (e.g. classification 16, extraction 512, chat 1k).
- **Fill policy**: greedy by reranker score per token, with a marginal-coverage penalty for near-duplicates (PACE-style submodular selection). Stop at a score cliff even if budget remains, because more context can hurt. Chunk token counts are precomputed at ingest for each tokenizer family, so the hot path does no tokenization.
- **Overflow**: (1) drop the lowest-utility chunks; (2) extractive prune within the remaining chunks; (3) re-route to a longer-context model; (4) never truncate silently mid-chunk.
- **Output control**: `max_tokens` per intent, a concise-answer instruction, structured outputs for machine consumers, and stop sequences. On tenants that opt in, log output length against the per-intent median to find verbosity regressions.

### 4. Caching hierarchy (cheapest first)

| Layer | Key | Invalidation | Notes |
|---|---|---|---|
| L0 exact response | hash(normalized request, model, decode params, doc-version set) | doc-version change | Only for deterministic settings (temp 0) |
| L1 answer cache | intent + query embedding + **dependency fingerprints** (doc versions, tool outputs, model, ACL scope) | CDC events | FinCacheServe design. Poisoning-checked hits. Tenant-scoped only. |
| L2 retrieval cache | rewritten-query embedding → candidate IDs + scores | index generation bump | Saves embed + ANN + rerank |
| L3 provider prefix cache | prompt layout discipline + breakpoints + `prompt_cache_key` | provider TTL (5 min / 1 h; 30 min on newer OpenAI) | Choose 1 h TTL only when the expected reads before expiry justify the 2× write (Anthropic doc's break-even math) |
| L4 self-hosted KV | vLLM/SGLang radix prefix cache + LMCache/CacheBlend chunk KV | doc version; edit-local repair | Sovereign deployments |

### 5. Proving value: measuring tokens saved and latency per stage

**Per-request trace** (OpenTelemetry spans emitted from Rust; one row per request in a columnar store):
- Latency per stage: `classify`, `rewrite`, `embed_q`, `bm25`, `ann`, `fuse`, `rerank`, `compress`, `assemble`, `upstream_ttft`, `upstream_total`, `stream_total`, plus p50/p95/p99. Track **gateway overhead = total − upstream** as a first-class SLO.
- Tokens: `input_uncached`, `cache_write`, `cache_read` (taken from provider usage fields: Anthropic `cache_creation_input_tokens` / `cache_read_input_tokens`, OpenAI `cached_tokens`), `retrieved_raw`, `retrieved_after_budget`, `retrieved_after_compress`, `output`, `output_reserve`.
- Decisions: retrieval strategy, escalations, cache layer hit, model chosen, compression ratio.

**Counterfactual baseline** (stated explicitly in the contract and dashboard): "naive RAG" = same model, fixed top-k=20 uncompressed chunks, full history, no caching, no output cap. Optionally "full-context" for small corpora. Metrics:
- `tokens_saved = baseline_input_tokens − actual_billable_equivalent`, where cache reads count at their price multiplier.
- `$ saved` from the live price table, split into savings from **routing, caching, retrieval budgeting, compression, skip-retrieval, and answer-cache hits**, so each feature's value is visible on its own.
- `latency_saved` for TTFT and total, against a sampled replay of the baseline.

**Quality must be shown next to savings:** shadow-run the baseline on a 1–5% sample (offline, async, within the tenant's privacy policy). Score both with RAGAS-style faithfulness/relevance plus ARES-calibrated judges, and with entity/number preservation for compressed requests. Report "savings at Δquality = x ± CI". Automatically roll back any optimization whose quality delta crosses a tenant threshold.

**Retrieval health dashboard:** recall@k on tenant golden sets (seeded from synthetic Q/A over their docs), share of no-retrieval / escalation decisions, index freshness lag per connector, stale-hit rate.

### 6. Build order

1. Prompt-layout discipline + provider caching + usage accounting + per-stage tracing (largest saving, no quality risk).
2. Hybrid tantivy + Qdrant + RRF + ort cross-encoder; text-splitter chunking; contextual retrieval at ingest.
3. Budget manager + per-intent output caps + strategy head on the intent classifier (including `none` and `full_context`).
4. Dependency-fingerprinted retrieval and answer caches.
5. Shadow-eval harness and savings ledger.
6. Extractive compression behind a flag. On-prem: vLLM/SGLang with EAGLE-3 + LMCache.
7. Distilled per-tenant small models for repeated tasks.

---

## Open questions

- **Small-corpus threshold:** what corpus size makes `full_context` + caching cheaper than RAG for each provider/model, given cache TTLs and the tenant's request rate? This needs a cost simulator built from real traffic.
- **Contextualization cost on churny corpora:** how often do docs change, and does late chunking (no LLM) match contextual retrieval quality on our tenants' data?
- **Own pruner/reranker:** train a commercially licensed Provence-style rerank+prune model (DeBERTa/ModernBERT class) on synthetic data? What is the CPU latency budget for 50 pairs?
- **Compression and attribution:** can extractive pruning keep citation precision at the source-span level? Measure this before we market compression.
- **Cross-tenant learning:** can strategy priors and thresholds learned on one tenant transfer to others without leaking data? Should the bandit be per tenant or federated?
- **Answer-cache legality:** does serving cached answers count as "processing" under tenants' data-residency and retention terms? This ties to the sovereign-deployment requirements.
- **Counterfactual honesty:** which baseline will customers accept as fair (naive RAG vs their pre-Caliban setup)? Should we let them define it?
- **KV-cache portability:** CacheBlend/TurboRAG gains are model-specific. Is chunk-KV reuse worth operating for on-prem tenants with fewer than N QPS?
- **Stale-evidence prompting:** how much do explicit validity statements ("superseded on DATE by DOC") reduce stale-document errors on closed frontier models, not just the open models studied?
