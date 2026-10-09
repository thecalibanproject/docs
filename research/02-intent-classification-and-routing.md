# Intent Classification and LLM Routing for Caliban

The literature supports routing as a real cost lever. Published results include 2x to 98% cost reductions at roughly equal quality: FrugalGPT reports up to 98%, RouteLLM up to 85% on MT-Bench, MixLLM about 76%, and BEST-Route and AutoMix 50–60%. The 2025–2026 benchmark wave (LLMRouterBench, RouterXBench, "Rethinking Predictive Modeling", "Most of the Routing Gap Is Task Type") adds three findings that should shape Caliban's design:
- **Simple routers are hard to beat.** A tuned kNN over prompt embeddings matches or beats complex learned routers. Most methods, including commercial ones, land within noise of each other. Most of the gain comes from knowing the *task type*, not fine-grained per-query prediction.
- **Routers are fragile.** They collapse toward the most expensive model as budgets rise. Query-independent adversarial suffixes can game them. They degrade under distribution shift, and published evals often overstate gains.
- **New models must be cheap to add.** Model pools change monthly. Methods that represent each model as a profile over prompt clusters (UniRoute, EmbedLLM, GraphRouter) let a new model join without retraining the router.

For Caliban, the right design is a cheap, explainable, staged router: rules, then embedding kNN, then a small encoder, with an LLM only as a fallback. Explicit, policy-driven intent→route mappings (Arch-Router style) sit on top of a learned quality/cost predictor per route. Caliban should keep improving the router from production feedback with guarded contextual bandits, and treat router integrity as a security surface.

## Key papers

Verdicts: **Adopt** = build into v1. **Prototype** = spike and benchmark it. **Watch** = track it. **Skip** = not worth Caliban's effort.

### Routing and cascades (model selection)

| Paper | Year | Core idea | Why it matters for Caliban | Verdict |
|---|---|---|---|---|
| [FrugalGPT](https://arxiv.org/abs/2305.05176) (Chen, Zaharia, Zou) | 2023 | Prompt adaptation, LLM approximation and a learned LLM cascade with a scorer that decides when to stop. Matches the best LLM at up to 98% lower cost, or +4% accuracy at equal cost. | Foundational evidence for cascades. Its scorer-gated "stop or escalate" pattern fits non-streaming, batch and agent sub-tasks. | Adopt (as a pattern) |
| [Hybrid LLM (ICLR'24)](https://proceedings.iclr.cc/paper_files/paper/2024/hash/b47d93c99fa22ac0b377578af0a1f63a-Abstract-Conference.html) (Ding et al., Microsoft) | 2024 | A BERT-style router predicts the quality gap between a small and a large model and routes by a tunable threshold. Up to 40% fewer large-model calls with no quality drop. | The canonical "difficulty estimator + threshold knob" pattern. The threshold maps directly to a per-tenant cost/quality slider. | Adopt |
| [BEST-Route](https://arxiv.org/abs/2506.22716) (ICML'25) | 2025 | Picks the model *and* the number of samples (best-of-n on small models). Up to 60% cost reduction with <1% quality drop. | Gives Caliban a second lever besides model choice: n-sampling small or on-prem models, which suits sovereign deployments. | Prototype |
| [AutoMix](https://arxiv.org/abs/2310.12963) | 2023 | Small model answers, few-shot self-verification scores it, and a POMDP decides whether to escalate. >50% compute reduction at comparable quality. | Post-hoc deferral works with black-box APIs. It costs a small-model generation plus a verify call, so it is unsuitable for streaming. | Prototype (non-streaming only) |
| [Language Model Cascades: Token-level Uncertainty and Beyond](https://arxiv.org/abs/2404.10136) (Gupta et al., Google, ICLR'24) | 2024 | Sequence-level confidence has a length bias, so deferral rules should use token-level uncertainty quantiles or a learned post-hoc deferral. | Directly informs the deferral signal design when Caliban hosts the small model and can see its logprobs. | Adopt (for self-hosted models) |
| [Cascade Routing: Unified Routing + Cascading](https://arxiv.org/abs/2410.10347) (Dekoninck et al., ICML'25) | 2024 | Optimal-in-theory combination: at each step choose the next model, which allows skipping or reordering. Beats both pure routing and pure cascading. | The theoretical frame for Caliban's "route first, optionally escalate" engine. Quality estimators are the bottleneck. | Adopt (framework) |
| [RouteLLM](https://arxiv.org/abs/2406.18665) (LMSYS / Berkeley) | 2024 | Binary strong/weak routers (matrix factorization, SW-ranking, BERT, causal LLM) trained on Chatbot Arena preferences with augmentation. Over 2x savings, and the repo claims up to 85% cost reduction at 95% of GPT-4 quality on MT-Bench. | The most reusable open baseline. Its matrix-factorization router is tiny and portable to Rust. Preference data is a cheap supervision source. | Adopt (baseline) |
| [RouterBench](https://arxiv.org/abs/2403.12031) (Martian / Berkeley / UCSD) | 2024 | Over 405k precomputed inference outcomes. Costs vary 2–5x for comparable performance. Lets you evaluate routers without inference. | Offline eval harness. Caliban should build its own equivalent from production logs. | Adopt (eval) |
| [RouterEval](https://arxiv.org/abs/2503.10657) | 2025 | Over 200M records from 8,500+ LLMs. Shows "model-level scaling up": with a good router, quality rises as the candidate pool grows. | Justifies a large BYO pool, but only if the router is strong. Weak routers do not benefit. | Watch |
| [LLMRouterBench](https://arxiv.org/abs/2601.07206) | 2026 | 400k instances, 21 datasets, 33 models. Many methods, including commercial routers, fail to reliably beat a simple baseline. Embedding choice matters little, and gains diminish with pool size. | Strong evidence against over-investing in exotic routers. Spend effort on data and policy. | Adopt (eval) |
| [Rethinking Predictive Modeling for LLM Routing: When Simple kNN Beats Complex Learned Routers](https://arxiv.org/abs/2505.12601) | 2025 | Tuned kNN over prompt embeddings matches or beats learned routers, with lower sample complexity and more robustness under shift. | kNN on the vector DB Caliban already runs for caching and RAG is the default Stage-1 router. | Adopt |
| [Most of the LLM Routing Gap Is Task Type](https://arxiv.org/abs/2608.23023) | 2026 | Twenty-one routing methods sit within a fraction of a point of each other. One model per task type captures 21 of the 29 routable questions, and per-language splits add 2 more. Also measures 5.37% run-to-run label noise. | Validates intent→route tables as the main mechanism. Label noise caps how much fine-grained routing can win. | Adopt |
| [MixLLM](https://arxiv.org/abs/2502.18482) (NAACL'25) | 2025 | Contextual-bandit router with tag-enhanced embeddings, per-model quality/cost predictors, a latency penalty and continual learning. 97.25% of GPT-4 quality at 24.18% of the cost under a latency constraint. | Closest published design to a gateway: online updates, latency in the objective, a changing model set. | Prototype |
| [Universal Model Routing (UniRoute)](https://arxiv.org/abs/2502.08773) (Jitkrittum et al., Google) | 2025 | Represents each LLM by its per-cluster error on a probe set, then routes a prompt to the cluster's best cost-adjusted model. Handles 30+ *unseen* LLMs without retraining. | The BYO-model answer: profile a tenant's new model on probe clusters, and it is routable immediately. | Adopt |
| [EmbedLLM](https://proceedings.iclr.cc/paper_files/paper/2025/hash/bf5e4b85d203481d6e37bd32d9600162-Abstract-Conference.html) (ICLR'25) | 2025 | Learns compact model embeddings with an encoder-decoder from correctness data. Beats prior routers and predicts benchmark scores without inference. | Model-as-vector fits the vector store. It complements UniRoute for cold-starting new models. | Prototype |
| [GraphRouter](https://arxiv.org/abs/2410.03834) (ICLR'25) | 2024 | Heterogeneous task–query–LLM graph with edge prediction of effect and cost. +12.3% and ≥9.5% better generalization to new LLMs. | Interesting for new-model generalization, but GNN serving is heavier than kNN. | Watch |
| [RouterDC](https://arxiv.org/abs/2409.19886) (NeurIPS'24) | 2024 | Encoder plus learned LLM embeddings trained with dual contrastive losses (sample–LLM and sample–sample). +2.76% in-distribution and +1.90% out-of-distribution over the best single LLM. | A good recipe for training Caliban's Stage-2 encoder head when several models are "good enough". | Prototype |
| [Zooter](https://arxiv.org/abs/2311.08692) (NAACL'24) | 2023 | Distills reward-model scores over candidate outputs into a cheap query→model router. Ranks first on 44% of tasks. | Shows how to label routing data with a reward model or LLM judge instead of humans. Caliban's offline labeling should do this. | Adopt (labeling method) |
| [Tryage](https://arxiv.org/abs/2308.11601) | 2023 | A perceptive router predicts downstream model performance, and the objective mixes in user flags such as size and recency. | An early form of "constraints as flags". Conceptually equivalent to tenant policy constraints. | Skip (superseded) |
| [Routing Collapse / EquiRouter](https://arxiv.org/abs/2602.03478) | 2026 | As budgets rise, routers default to the most expensive model even when cheap ones suffice, because scalar score prediction mismatches discrete choice. A ranking-based router saves about 17% at GPT-4-level quality. | Train Caliban's predictors with *ranking* or decision-aware losses, and monitor the share routed to the top tier. | Adopt (loss and monitor) |
| [OrcaRouter](https://arxiv.org/abs/2605.30736) | 2026 | Production router: offline per-model ridge regression on a reward matrix, then online LinUCB on lexical and embedding features. Ranked second on RouterArena with 75.54% accuracy at $1 per 1k queries. | A near-exact template for Caliban's offline-to-online learning loop. Small linear algebra is trivial in Rust. | Adopt |
| [Correlation-Aware Contextual Bandits with Surrogate Rewards](https://arxiv.org/abs/2607.09015) | 2026 | Mixes sparse true rewards with cheap surrogate rewards such as judges and heuristics to speed up bandit learning. | Production feedback is sparse, so surrogate rewards (LLM judge, retry signals) are needed. | Prototype |
| [NeuralUCB Online Routing](https://arxiv.org/abs/2603.30035) | 2026 | NeuralUCB for cost-aware routing. Beats min-cost and random baselines, and is far cheaper than always choosing max quality. | Neural bandit option if linear bandits underfit. | Watch |
| [Drift-Aware Sparse Routing](https://arxiv.org/abs/2609.00662) | 2026 | Rolling-window sparse linear routing with an audit rate, shared compute, latency and money budgets, and regret bounds under drift. | Formalizes the "audit sampling rate" and window size Caliban needs for drift. | Watch |
| [Mixture-of-Agents](https://arxiv.org/abs/2406.04692) (Together) | 2024 | Layered multi-LLM aggregation. 65.8% LC win rate on AlpacaEval 2.0 vs 57.5% for GPT-4o. | A quality-maximizing premium mode, not a cost saver. Expose it as an opt-in route ("max quality"). | Watch |
| [Dynamic Model Routing and Cascading: A Survey](https://arxiv.org/abs/2603.04445) (TMLR'26) | 2026 | Taxonomy by *when* a decision is made, *what* information it uses, and *how* it is computed. Covers difficulty, preference, clustering, uncertainty, RL and cascades. | Reference map for the team. | Read |
| [Doing More with Less: Routing Strategies Survey](https://www.arxiv.org/abs/2502.00409v2) | 2025 | Survey of routing for resource optimization in LLM systems. | Second survey, more systems-oriented. | Read |
| [LLMRouter infrastructure + xRouteBench](https://arxiv.org/abs/2608.06867) (UIUC) | 2026 | Unifies routers into context encoder, model encoder, scorer, decision rule and learning signal. Ships 16+ routers. Learned routers beat fixed baselines by 14.6%, and lightweight routers are competitive under cost constraints. | Its five-part decomposition is a good internal API for Caliban's router crate. | Adopt (architecture) |

### Preference-aligned and intent-based routing

| Paper | Year | Core idea | Why it matters for Caliban | Verdict |
|---|---|---|---|---|
| [Arch-Router](https://arxiv.org/abs/2506.16655) (Katanemo) | 2025 | A 1.5B generative model maps a query plus conversation to user-defined (domain, action) route descriptions. New models or routes need no retraining. Reports SOTA preference matching vs proprietary LLMs. | The best fit for *tenant-defined* routing policies written in natural language. It is a good Stage-3 fallback and policy-authoring UX, but too slow for the hot path on every request. | Adopt (Stage 3) |
| [When to Reason: Semantic Router for vLLM](https://arxiv.org/abs/2510.08731) (NeurIPS'25 MLSys workshop) | 2025 | A ModernBERT-based classifier decides per query whether to enable reasoning mode. +10.2 pp on MMLU-Pro, −47.1% latency, −48.5% tokens. | "Reasoning effort" is a route dimension Caliban should control, not only model choice. Their classifier core runs in Rust on Candle. | Adopt |
| [Intent Detection in the Age of LLMs](https://arxiv.org/abs/2410.01627) (EMNLP'24 Industry) | 2024 | Compares 7 LLMs (ICL/CoT) against SetFit. A hybrid that routes uncertain SetFit cases to the LLM gets within 2% of LLM accuracy at 50% less latency. LLM out-of-scope ability degrades with label-space size. | Direct evidence for "small encoder first, LLM on uncertainty". | Adopt |
| [Efficient OOS Detection via Uncertainty-Driven LLM Routing](https://arxiv.org/abs/2507.01541) | 2025 | Uncertainty on an intent classifier, with high-uncertainty cases sent to a fine-tuned LLM. SOTA out-of-scope results, including on real deployed task-oriented dialogue traffic. | The same cascade applied to out-of-scope detection, which Caliban needs before mapping to agents or datasources. | Adopt |
| [SetFit](https://arxiv.org/abs/2209.11055) | 2022 | Contrastive fine-tuning of a sentence transformer plus a light head. Competitive with 8 labels per class. | Lets tenants define custom intents with a handful of examples. The output is an ONNX-exportable encoder. | Adopt |
| [ModernBERT](https://arxiv.org/abs/2412.13663) | 2024 | Modernized encoder (RoPE, GeGLU, alternating attention, unpadding, 8k context). Fastest and most memory-efficient encoder in its class. | Default backbone for Caliban's multi-head classifier (intent, difficulty, OOS, jailbreak, PII-presence). | Adopt |

### Routing among agents and tools

| Paper | Year | Core idea | Why it matters for Caliban | Verdict |
|---|---|---|---|---|
| [Multi-Agent Routing as Set-Valued Prediction (WildChat)](https://arxiv.org/abs/2606.28925) | 2026 | 3,000 prompts over a fixed 12-agent catalog. Routing outputs a *set* of agents. A fine-tuned encoder beats both kNN and zero-shot LLM routing, and a linear multilabel classifier is the most practical option. | Agent ("node") selection is multilabel, not single-label. Caliban should use supervised encoders once it has labels. | Adopt |
| [ACE-Router](https://arxiv.org/abs/2601.08276) | 2026 | History-aware routing trained on MCP tool data transfers to agent selection, with 91.6% accuracy. | Caliban can bootstrap agent routing from tool-selection data. | Watch |
| [AgentRouter (KG-guided)](https://arxiv.org/abs/2510.05445) | 2025 | Knowledge-graph-guided router for collaborative multi-agent QA. | Relevant to routing over ontology-linked datasources. | Watch |
| [Agent-as-a-Graph](https://arxiv.org/abs/2511.18194) | 2025 | Knowledge-graph retrieval of tools and agents for multi-agent systems. | Maps well onto Caliban's ontology layer: retrieve agents and datasources as graph nodes. | Watch |

### Routing under latency SLOs

| Paper | Year | Core idea | Why it matters for Caliban | Verdict |
|---|---|---|---|---|
| [HW-Router](https://arxiv.org/abs/2608.14575) | 2026 | Uses live queue, memory and GPU signals to predict actual latency per model. 3.4–3.9x lower end-to-end latency and +46–48 pp SLO attainment, with about 200 µs routing overhead. | Caliban's sovereign mode controls the GPUs, so its route score must include *live* load, not static latency. | Adopt (on-prem) |
| [RouterWise](https://arxiv.org/abs/2604.10907) | 2026 | Joint resource allocation and routing under latency SLOs. | Capacity planning for on-prem pools. | Watch |
| [Online LP for Multi-Objective Routing in LLM Serving](https://arxiv.org/abs/2607.03948) | 2026 | Online linear programming with explicit SLO terms. | A principled way to enforce per-tenant budget pacing. | Watch |

## Open-source implementations

| Project | Language | Use | Notes |
|---|---|---|---|
| [lm-sys/RouteLLM](https://github.com/lm-sys/RouteLLM) | Python | Train and serve strong/weak routers. OpenAI-compatible server. Apache-2.0. | The matrix-factorization router is an embedding dot-product plus a linear layer, so its weights are trivial to port to Rust. The BERT router exports to ONNX. Use it as the offline baseline. |
| [vllm-project/semantic-router](https://github.com/vllm-project/semantic-router) | Go + Rust (`candle-binding`) | Signal-driven Mixture-of-Models router: intent/category, reasoning on/off, semantic cache, LoRA heads. Apache-2.0. | **The closest reference architecture.** Its ModernBERT classifiers run in Rust on Candle. Read before writing Caliban's classifier crate. |
| [katanemo/archgw → Plano](https://github.com/katanemo/archgw) | Rust + Envoy | AI-native proxy: model aliases, preference routing, agent orchestration via a 4B orchestrator model. Apache-2.0. | The most direct competitor and design reference (Rust data plane, policy-described routes). |
| [katanemo/Arch-Router-1.5B](https://huggingface.co/katanemo/Arch-Router-1.5B) ([GGUF](https://huggingface.co/katanemo/Arch-Router-1.5B.gguf)) | Model | Preference / (domain, action) routing LLM. | GGUF runs via llama.cpp, and the Qwen-family architecture is loadable in Candle. Check the license terms (paper says CC-BY-4.0) before shipping it in a sovereign appliance. |
| [aurelio-labs/semantic-router](https://github.com/aurelio-labs/semantic-router) | Python | Utterance-exemplar kNN routes over a vector index. MIT. | The concept is about 200 lines of Rust on top of Caliban's vector DB. Copy the idea, not the code. |
| [Not-Diamond/RoRF](https://github.com/Not-Diamond/RoRF) | Python | Random-forest pairwise router over prompt embeddings, with 12 pretrained routers. | Tree models export to ONNX (sklearn→ONNX), or can be reimplemented. Not Diamond's commercial router is a closed successor. Also see [awesome-ai-model-routing](https://github.com/Not-Diamond/awesome-ai-model-routing). |
| [withmartian/routerbench](https://github.com/withmartian/routerbench) | Python / data | Offline routing eval dataset. | Seed Caliban's eval harness with it. Martian's commercial router is closed. |
| [microsoft/best-route-llm](https://github.com/microsoft/best-route-llm) | Python | BEST-Route / Hybrid-LLM code. | Reference for difficulty-gap labels. |
| [automix-llm/automix](https://github.com/automix-llm/automix) | Python | Self-verification cascades. | Reference only. |
| [shuhao02/RouterDC](https://github.com/shuhao02/RouterDC) | Python | Contrastive router training. | The training recipe matters, not the runtime. |
| [ulab-uiuc/GraphRouter](https://github.com/ulab-uiuc/GraphRouter) | Python | GNN router. | Watch only. |
| [richardzhuang0412/EmbedLLM](https://github.com/richardzhuang0412/EmbedLLM) | Python | Model embeddings for routing. | Its model vectors can live in Caliban's vector DB. |
| RouterXBench ([paper](https://arxiv.org/abs/2602.11877); code at GitHub `zhuchichi56/RouterXBench` per the paper) | Python | Router eval across scenarios and out-of-distribution data. Includes a hidden-state "ProbeDirichlet" router. | Probes need hidden states, so they apply only to self-hosted models. |
| [togethercomputer/MoA](https://github.com/togethercomputer/moa) | Python | Mixture-of-Agents. | A pattern to implement natively as a "max-quality" route. |
| [huggingface/setfit](https://github.com/huggingface/setfit) | Python | Few-shot intent training. | Export the body and head to ONNX, then serve in Rust via `ort`. |
| [AnswerDotAI/ModernBERT](https://github.com/answerdotai/modernbert) | Python / weights | Encoder backbone. | Export to ONNX for `ort`. The vLLM SR Candle binding shows a native Rust path. |
| [pykeio/ort](https://github.com/pykeio/ort) | Rust | ONNX Runtime bindings (CPU, CUDA, TensorRT, CoreML and others). Apache-2.0/MIT. | **Primary runtime for in-process classifiers and embedders.** |
| [huggingface/candle](https://github.com/huggingface/candle) | Rust | Pure-Rust ML (CPU/MKL/Accelerate, CUDA, WASM). Apache-2.0/MIT. | Use when avoiding the ONNX Runtime C++ dependency matters, for example in air-gapped builds. Includes BERT-family models and small LLMs (Qwen, Phi) for a Stage-3 router. |
| [Anush008/fastembed-rs](https://github.com/Anush008/fastembed-rs) | Rust | Local embeddings and rerankers via `ort` and tokenizers. Apache-2.0. | Drop-in embedder for Stage-1 kNN and the semantic cache. |
| [Shyamcharan06/semantic-router-rs](https://github.com/Shyamcharan06/semantic-router-rs) | Rust | Hobby-scale Rust port in the spirit of vLLM SR: embedding plus classifier routing, PII/jailbreak guards, cache. | Not production-grade. Skim for crate choices only. |
| [MilkThink-Lab/Awesome-Routing-LLMs](https://github.com/MilkThink-Lab/Awesome-Routing-LLMs) | List | Paper index. | Keep it bookmarked for tracking new work. |

## Risks & failure modes

| Risk | Evidence | Mitigation in Caliban |
|---|---|---|
| **Adversarial rerouting (cost inflation)**: query-independent "confounder gadgets" push weak queries to the strong model with over 90% success on open routers. The attack transfers, and works against commercial routers too. | [Rerouting LLM Routers (COLM'25)](https://arxiv.org/abs/2501.01818) | Per-key spend caps and escalation-rate monitors. Price strong-tier usage to the tenant, not absorbed by Caliban. Use rule and kNN agreement checks. Flag requests whose route flips under prompt-suffix ablation. |
| **Cascade attacks**: one adversarial suffix lowers accuracy, raises cost and enables jailbreaks across a cascade. Images can force deferral in multimodal cascades. | [When Efficiency Backfires](https://arxiv.org/abs/2605.17288), [Forced Deferral](https://arxiv.org/abs/2606.15308) | Run safety classification *before* the cascade. Cap escalations per request. Do not let model self-confidence be the only gate. |
| **Routing collapse**: routers drift to the most expensive model as budgets loosen. | [When Routing Collapses](https://arxiv.org/abs/2602.03478) | Ranking or decision-aware losses. A dashboard of tier share per intent. Alert when the top-tier share exceeds the policy. |
| **Distribution shift and drift**: prompt mix changes by hour, cohort and release, and provider model updates silently change quality. | [Drift-Aware Sparse Routing](https://arxiv.org/abs/2609.00662), [kNN robustness](https://arxiv.org/abs/2505.12601), [RouterXBench](https://arxiv.org/abs/2602.11877) | Audit-sample a fixed percentage to multiple models. Use rolling windows. Re-profile models on probe clusters when a provider version changes. Prefer kNN, which degrades more gracefully. |
| **Eval inflation and selection bias**: safety-routing gains vanish under held-out categories (7–9x larger bias on AgentDojo). Routers cluster within noise on standard benchmarks. | [False Floors](https://arxiv.org/abs/2610.01535), [LLMRouterBench](https://arxiv.org/abs/2601.07206), [Routing Gap Is Task Type](https://arxiv.org/abs/2608.23023) | Evaluate on held-out tenants, intents and time windows. Always report "one model per intent" and kNN baselines. Measure label noise by re-running. |
| **Label noise caps gains**: 5.37% of model–question outcomes flip between runs. | [Routing Gap Is Task Type](https://arxiv.org/abs/2608.23023) | Don't chase sub-point routing wins. Use repeated or majority labels for the offline matrix. |
| **LLM-classifier OOS weakness**: LLM out-of-scope detection degrades as the label space grows. | [Intent Detection in the Age of LLMs](https://arxiv.org/abs/2410.01627) | Use an explicit OOS head and thresholds in the encoder. Show the LLM fallback only the top-k candidate routes, not all of them. |
| **Self-verification and confidence miscalibration**: sequence-level confidence is length-biased. | [LM Cascades: Token-level Uncertainty](https://arxiv.org/abs/2404.10136) | Use token-quantile features or learned deferral. Calibrate per model version. |
| **Latency blowup from LLM-based routers**: a 1.5B generative router on every call adds tens to hundreds of ms. | Inferred from Arch-Router model size. Contrast [HW-Router](https://arxiv.org/abs/2608.14575) at about 200 µs overhead. | Use the LLM router only on low-confidence traffic, with a hard timeout that falls back to the tenant's default route. |
| **Cold start for BYO models and new tenants** | [UniRoute](https://arxiv.org/abs/2502.08773), [EmbedLLM](https://proceedings.iclr.cc/paper_files/paper/2025/hash/bf5e4b85d203481d6e37bd32d9600162-Abstract-Conference.html) | Profile on a probe set. Use priors from global (cross-tenant, privacy-safe) model profiles. |

## Recommended design for Caliban

### Principles
1. **Route = (intent, agent/node set, datasources, model, reasoning effort, sampling n).** The router does more than pick a model. Intent is the primary key, and most routing value comes from it ([Task Type paper](https://arxiv.org/abs/2608.23023)).
2. **Explicit policy on top, learned prediction underneath.** Tenants declare routes and constraints, and Caliban learns which concrete model best serves each route under those constraints.
3. **One embedding per request, reused everywhere.** The same vector serves the semantic cache, RAG retrieval, Stage-1 routing and drift monitoring.
4. **Fail toward the tenant default.** Every stage has a timeout, and a router failure must never block a request.

### Staged pipeline (hot path)

The latency figures below are design targets to validate in benchmarks, not measured numbers.

| Stage | What runs | Decides | Exit condition | Target p50 / p99 |
|---|---|---|---|---|
| 0. Rules / policy | Compiled tenant policy: pinned model (`model=` not `auto`), header overrides, modality, tool-call presence, context length vs model window, data-classification tags (PII class → on-prem only), allow/deny lists, budget state | Hard constraints and forced routes | A pinned or forced route exits. Otherwise the stage emits a *feasible model set* | <0.1 ms |
| 1. Embedding kNN (semantic router) | `fastembed-rs`/`ort` small embedder (shared with cache), then HNSW over per-tenant labeled exemplars (intent utterances + historical routed prompts with outcomes) | Intent (+ OOS distance), and a kNN estimate of per-model quality | Top-1 vote margin ≥ τ₁ and similarity ≥ σ_oos → exit | 2–8 ms CPU / <15 ms |
| 2. Small classifier | ModernBERT-base (or distilled MiniLM for edge) multi-head via `ort`. Heads: intent (tenant LoRA/head), difficulty / quality-gap (Hybrid-LLM style), reasoning-needed, OOS, jailbreak, PII-presence. Agent selection uses a multilabel head. | Intent, agent set, difficulty bucket, reasoning effort | Calibrated max-prob ≥ τ₂ → exit | 5–20 ms CPU, <5 ms GPU |
| 3. LLM fallback | Arch-Router-1.5B-class model (Candle or llama.cpp, on-prem friendly) given the *top-k* candidate routes as natural-language descriptions, with constrained JSON output | Route on ambiguous or novel queries | Hard timeout (e.g., 150 ms), else tenant default | 50–200 ms. Target ≤5–10% of traffic |
| 4. Model selection within route | Score = predicted quality (kNN/UniRoute cluster profile, ranking loss) − λ_cost·$ − λ_lat·live-latency (HW-Router-style queue signals for on-prem). Contextual bandit (LinUCB / Thompson) adds exploration within a budget. | Concrete model, n-samples, provider | Always | <0.5 ms |
| 5. Optional post-hoc deferral | Non-streaming, batch and agent sub-tasks only: token-uncertainty or verifier gate (Gupta et al. / AutoMix), at most one escalation | Escalate or accept | Tenant opt-in | +1 small-model call |

Total router overhead target: **p50 under 10 ms and p99 under 30 ms** without Stage 3, which is small next to LLM time-to-first-token.

### How intents map to models, agents and datasources
- **Route registry** (per tenant, versioned) with these fields: `route_id`, natural-language description (used by Stage 3 and for explainability), exemplar utterances (Stage 1), and optional classifier label (Stage 2). It also holds `agents[]` (Caliban nodes/sub-apps), `datasources[]` (ontology entity types, which scope RAG retrieval), a `model_pool` (ordered or quality-tiered candidates, which can include BYO endpoints), `constraints` (residency, PII class, max $ per call, latency SLO, reasoning allowed) and an `objective` (λ_cost, λ_lat or a "quality floor").
- **Hierarchical intents**: domain → action (Arch-Router's taxonomy). A global Caliban taxonomy (code, SQL/analytics, extraction, summarization, chit-chat, math/reasoning, RAG-QA and others) gives cold-start defaults. Tenants extend it with custom leaves.
- **Agent routing is set-valued.** Use a multilabel head or kNN, then cost-aware set pruning ([WildChat set-valued routing](https://arxiv.org/abs/2606.28925)). Datasource selection reuses the ontology graph: retrieve entity nodes via the same embedding ([Agent-as-a-Graph](https://arxiv.org/abs/2511.18194) pattern).
- **OOS** → tenant-defined fallback route (a general model, a clarifying question, or a refusal). OOS is an explicit decision, never an argmax over known intents.

### Per-tenant policies and BYO models
- Policy DSL (YAML/JSON, compiled to a Rust decision table) evaluated in Stage 0. Hierarchy: Caliban defaults → org → project → API key → request header. Every decision logs the policy version and rule ID.
- **BYO model onboarding**: register the endpoint and price, then Caliban runs the **probe set** (about 1–2k prompts stratified by global intent clusters, judged by a reference LLM judge, plus the tenant's own probe set if supplied). The result is a per-cluster quality vector ([UniRoute](https://arxiv.org/abs/2502.08773)) and optionally an EmbedLLM-style model embedding. The model is routable immediately, with no router retraining. Cost and latency come from the registry and live telemetry.
- **Sovereign mode**: all of Stages 0–3 run in-process or on-node (ONNX/Candle weights shipped in the appliance). There are no external calls, and the learning loop runs locally with optional federated profile sharing.

### Learning the router from production feedback
1. **Log** every decision: embedding ID, stage that decided, scores, feasible set, chosen model, **propensity** (needed for off-policy evaluation), cost, latency, policy version.
2. **Rewards**, in order of trust: explicit feedback (thumbs, accepted edits) > task success for agents (tool calls succeeded, SQL ran, no retry) > implicit (regenerate or retry within N seconds, user switches model) > sampled LLM-judge scores (surrogate rewards, mixed as in [correlation-aware bandits](https://arxiv.org/abs/2607.09015)).
3. **Offline loop (nightly)**: rebuild the kNN exemplar store with outcome-labeled prompts. Refit per-model ridge or ranking predictors ([OrcaRouter](https://arxiv.org/abs/2605.30736) offline phase, ranking loss per [EquiRouter](https://arxiv.org/abs/2602.03478)). Retrain encoder heads (SetFit/RouterDC recipe) when the intent label count grows. Label new data via a Zooter-style judge or reward-model distillation.
4. **Online loop**: LinUCB per route over a small feature vector (projected embedding + difficulty + length) with an exploration budget (e.g., ≤2–5% of traffic, only among models above the route's quality floor and within the tenant's cost ceiling).
5. **Audit traffic**: a fixed small percentage (configurable, off by default for strict tenants) is sent to a second model for comparison. This gives unbiased drift detection ([DRS](https://arxiv.org/abs/2609.00662)).
6. **Gates**: shadow-deploy new router versions. Promote only if off-policy evaluation (IPS/DR on logged propensities) and held-out-window evals beat both the current router *and* the "one model per intent" baseline. Roll back automatically on SLO, cost or top-tier-share alarms.

### What to build in Rust (data plane)
- `caliban-router` crate split into the [LLMRouter](https://arxiv.org/abs/2608.06867) components: `ContextEncoder` (ort/fastembed), `ModelEncoder` (profile vectors), `Scorer` (kNN, linear, ONNX head), `DecisionRule` (policy + constrained argmax + bandit), `Signal` (feedback ingestion).
- In-process inference: `ort` as the default, with `candle` as the pure-Rust or air-gapped option. Models are versioned artifacts, hot-swapped via `ArcSwap`, one tokenizer per model via `tokenizers`.
- HNSW exemplar index shared with the semantic cache (same embedding space, per-tenant namespaces).
- Live latency/load table fed by the provider clients (EWMA TTFT, tokens/s, queue depth, error rate) for Stage-4 scoring and SLO-aware fallbacks.
- Policy compiler (YAML → decision table) and decision logger (async, to the event bus).
- **Training stays in Python** (SetFit, ModernBERT fine-tunes, RouteLLM/RouterDC baselines, judges). The artifact contract is ONNX plus JSON profile tables.

### Build order
1. Stages 0, 1 and 4 with static per-intent model tables. This already captures most of the "task-type" gain.
2. Add the ModernBERT multi-head (intent, difficulty, reasoning, OOS, jailbreak), probe-set BYO onboarding, and the offline learning loop.
3. Add the LinUCB online loop, the Arch-Router Stage 3, and optional cascades for batch and agent workloads.

## Open questions
- **Quality signal at scale:** which judge is trustworthy per intent, how to calibrate it per tenant, and how much judge cost the savings can absorb.
- **Cross-tenant learning:** can global model-quality profiles be shared (privacy, competitive concerns) to cold-start new tenants? Is federated profile aggregation needed for sovereign deployments?
- **Streaming vs cascades:** post-hoc deferral conflicts with streaming. Is "speculative" streaming from the small model with a mid-stream switch acceptable UX?
- **Multi-turn routing:** route per turn or per session? Switching models mid-conversation can hurt coherence and prompt-cache hits. Provider prompt caching changes the cost math.
- **Agent-level routing:** how to attribute end-task success back to individual routing decisions in multi-step agent graphs (credit assignment).
- **Pricing incentives:** if Caliban sells "auto" at a flat price, router manipulation and routing collapse become Caliban's margin risk. If usage is passed through, the incentives differ.
- **Licensing:** Arch-Router weights, the Plano orchestrator and judge models need a license review for on-prem redistribution.
- **Benchmarking:** Caliban needs its own RouterBench-style matrix from real traffic, plus an agreed metric (e.g., cost at a fixed quality floor per intent) to stop chasing within-noise gains.
