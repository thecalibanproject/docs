# Multi-Tenant AI Gateway, Model Serving & Sovereign Deployment

*Research snapshot: 2026-10-02. Scope: the Caliban data plane (gateway), on-prem model serving, and "sovereign router" deployment.*

**Summary.** The gateway layer is crowded but split. Hosted routers like OpenRouter and Cloudflare are easy to adopt but cannot run on-prem. Infra-grade proxies (Agent Router, formerly Envoy AI Gateway; Kong; Plano) are built for platform teams, not for application builders. Developer gateways (LiteLLM, Portkey) are written in Python or TypeScript and keep their governance features behind enterprise paywalls. Two Rust gateways already exist. Helicone and TensorZero both show that a Rust data plane can add less than 5 ms (P95) and less than 1 ms (P99) of overhead respectively. Low latency therefore cannot be Caliban's differentiator. It is only the minimum needed to compete. None of these products combines all of: (1) reversible PII anonymization that works on streamed output, (2) intent routing that uses the tenant's own data and ontology, (3) a real multi-tenant control plane, and (4) the **same binary** running in SaaS, a customer VPC and an air-gapped site, with signed models and offline licensing. That combination is Caliban's opening. On the serving side, the field has settled. TGI is archived. vLLM is the default and SGLang suits workloads that reuse long prompt prefixes. Kubernetes-native schedulers (llm-d, NVIDIA Dynamo) now handle prefix-aware routing and split prefill from decode. Caliban should **integrate with these engines, not build its own**. On regulation, the EU AI Act's GPAI duties have applied since 2025-08-02. The Digital Omnibus (Reg. (EU) 2026/1744) moved high-risk deadlines to Dec 2027 (Annex III systems) and Aug 2028 (AI inside Annex I products), but the Art. 50 transparency duties still apply from 2026-08-02.

---

## Competitive landscape

| Product | Language | OSS? | Strengths | Gaps Caliban can exploit |
|---|---|---|---|---|
| [OpenRouter](https://openrouter.ai/docs/guides/routing/provider-selection) | n/a (closed SaaS) | No | Largest model catalogue. Default routing is price-weighted with uptime checks (inverse-square weighting by cost, avoids providers with outages in the last 30 s). Per-request options: `allow_fallbacks`, `sort` by latency or throughput, `quantizations` filter, `max_price`, `data_collection: deny`, `zdr: true` ([routing docs](https://openrouter.ai/docs/guides/routing/provider-selection)). BYOK keys are tried first, then shared capacity ([BYOK](https://openrouter.ai/docs/guides/overview/auth/byok)). | SaaS only, so no VPC or air-gap. No PII handling, no datasource or RAG layer, no agents. All prompts pass through a third party. No per-tenant self-hosted models. |
| [LiteLLM](https://docs.litellm.ai/docs/proxy/architecture) | Python | Yes (MIT core + paid enterprise) | Huge provider coverage. Virtual keys with USD budgets, RPM limits and team/user hierarchies backed by Postgres ([virtual keys](https://docs.litellm.ai/docs/proxy/virtual_keys), [budgets](https://docs.litellm.ai/docs/proxy/users)). Retries inside a model group, then fallback across groups ([reliability](https://docs.litellm.ai/docs/proxy/reliability)). Anthropic-format `/v1/messages` that routes to any provider ([docs](https://docs.litellm.ai/docs/anthropic_unified)). | Python runtime limits throughput and adds operational load. **Supply-chain compromise:** malicious PyPI releases 1.82.7 and 1.82.8 on 2026-03-24 stole credentials. The attackers got in through a compromised Trivy step in CI ([LiteLLM post-mortem](https://docs.litellm.ai/blog/security-update-march-2026), [Datadog](https://securitylabs.datadoghq.com/articles/litellm-compromised-pypi-teampcp-supply-chain-campaign/), [Trend Micro](https://www.trendmicro.com/en_us/research/26/c/inside-litellm-supply-chain-compromise.html)). Enterprises now ask hard questions about gateway provenance. |
| [Portkey Gateway](https://github.com/portkey-ai/gateway) | TypeScript | Yes (MIT) | Claims 1,600+ LLMs and 50+ guardrails. Production gateway merged into OSS on 2026-03-24 ([discussion](https://github.com/Portkey-AI/gateway/discussions/1576)). Reports about 2T tokens a day of production traffic. | Node runtime. Its governance features (RBAC, analytics) depend on the hosted control plane. No sovereign model serving, no ontology or RAG layer. |
| [Helicone AI Gateway](https://github.com/Helicone/ai-gateway) | **Rust** | Yes (Apache-2.0) | OpenAI-compatible. Routing options include P2C + PeakEWMA, latency, weighted and cost-based. Rate limits by requests, tokens and dollars. Redis or S3 cache, OTel export. Claims <5 ms P95 and ~64 MB RAM. | Small community (~640 stars). Built around observability. No PII handling, no agents, no real multi-tenant control plane. Shows that tower-style P2C/EWMA balancing works for LLM traffic. |
| [TensorZero](https://github.com/tensorzero/tensorzero) | **Rust** | Yes (Apache-2.0) | Claims <1 ms P99 at 10k+ QPS. Retries, fallbacks, A/B tests ([bandits](https://www.tensorzero.com/blog/bandits-in-your-llm-gateway/)). Inference-time optimizations: DICL and best/mixture-of-N. Feedback loop over stored inferences ([gateway docs](https://tensorzero.mintlify.dev/docs/gateway/index)). Frames LLM apps as [POMDPs](https://www.tensorzero.com/blog/think-of-llm-applications-as-pomdps-not-agents/). | Built for one team optimizing its own app, with schema-first "functions" config. Not a multi-tenant SaaS gateway. No PII, no data connectors. **Steal the idea:** tie inferences to feedback and use that to improve routing. |
| [Agent Router](https://theagentrouter.ai/blog/envoy-ai-gateway-is-now-agent-router/) (formerly Envoy AI Gateway) | Go control plane + Envoy (C++) | Yes (Apache-2.0) | Renamed and moved to the Agentic AI Foundation on 2026-09-10. Token-based rate limits use Envoy's global rate limiter with Redis, CEL cost expressions and per-model limits ([usage-based rate limiting](https://aigateway.envoyproxy.io/docs/0.1/capabilities/usage-based-ratelimiting/)). | Kubernetes and CRDs only, aimed at platform teams. No PII rehydration, semantic cache, ontology or billing. A likely **partner, not rival**: Caliban can sit behind it or alongside it. |
| [Kong AI Gateway](https://konghq.com/products/kong-ai-gateway) | Lua / Nginx | Partly (AI Proxy is OSS; enterprise plugins are paid) | AI Proxy, Semantic Cache (Redis, pgvector, Valkey), AI Rate Limiting Advanced, and an [AI PII Sanitizer](https://developer.konghq.com/plugins/ai-sanitizer.md) that can use placeholders or synthetic replacements. | A general API gateway with AI features added on. PII, semantic guard and advanced rate limiting require the enterprise licence. No model serving and no intent routing that knows about the tenant's data. |
| [Cloudflare AI Gateway](https://developers.cloudflare.com/ai-gateway/) | closed (Workers) | No | Free analytics, caching and rate limits. Unified Billing adds 5% on credits and passes inference prices through ([pricing](https://developers.cloudflare.com/ai-gateway/reference/pricing/)). Cloudflare also built its own Rust inference engine, [Infire](https://blog.cloudflare.com/cloudflares-most-efficient-ai-inference-engine/). | Cloudflare-hosted only, so no air-gap. Exact-match caching only. Not sovereign for regulated EU or public-sector buyers. |
| [Plano](https://github.com/katanemo/plano) (formerly Katanemo Arch) | Rust + Envoy | Yes (Apache-2.0) | Out-of-process data plane for agents. Small routing models (e.g. a 4B orchestrator) route by stated preference. Guardrail filter chains. OTel traces without code changes. ~7k stars. | Single-tenant design with Envoy complexity. No billing, BYOK vault or PII rehydration. Its routing-model approach is close to Caliban's intent classifier, so benchmark against it. |
| [vLLM Semantic Router](https://vllm-project.github.io/2025/09/11/semantic-router.html) | Go + Rust (Envoy ext_proc) | Yes | ModernBERT classifiers for intent, PII and jailbreaks, with a Rust classification core ([paper](https://arxiv.org/abs/2510.08731)). Decides whether a request needs a reasoning model. | Research-grade and tied to the vLLM/Envoy stack. Shows how to build Caliban's classifier: a small encoder running in-process. |

**Takeaway:** OpenAI Chat Completions is the common interface. Every product above accepts it, and the serving engines expose it too: vLLM, SGLang, llama.cpp `llama-server`, `trtllm-serve`, mistral.rs. Anthropic Messages compatibility is now expected (LiteLLM `/v1/messages`, mistral.rs). Neither one alone is enough.

---

## Key papers (serving & scheduling)

| Paper | Year | Core idea | Reported gain | Relevance to Caliban | Verdict |
|---|---|---|---|---|---|
| [PagedAttention / vLLM](https://arxiv.org/abs/2309.06180v1) (SOSP'23) | 2023 | Stores the KV cache in fixed-size blocks, like OS virtual-memory paging, so less memory is wasted and blocks can be shared. | 2–4× throughput vs. prior systems. | The basis of the default on-prem engine. | **Adopt** (via vLLM) |
| [SGLang / RadixAttention](https://arxiv.org/abs/2312.07104) | 2023–24 | Radix tree over token prefixes so KV cache is reused across requests. Compressed FSM for structured decoding. | Up to 6.4× throughput. | Agents and RAG reuse long system prompts and documents, which is Caliban's workload. | **Adopt** for prefix-heavy pools |
| [Sarathi-Serve](https://arxiv.org/abs/2403.02310) (OSDI'24) | 2024 | Splits prefill into chunks so new requests join a batch without stalling decodes. | 2.6× (Mistral-7B, 1×A100) and 6.9× (Falcon-180B) capacity within SLO. | Now built into engines (Infire uses chunked prefill too). Set the flag; nothing to build. | **Configure** |
| [DistServe](https://arxiv.org/abs/2401.09670) (OSDI'24) | 2024 | Runs prefill and decode on separate GPUs, with resources and parallelism tuned per phase for TTFT and TPOT. | 4.48× more requests or 10.2× tighter SLO (>90% SLO attainment). Up to 7.4× lower cost per query. | Worth it only for large on-prem pools with long prompts. | **Adopt via llm-d/Dynamo at scale** |
| [Splitwise](https://arxiv.org/abs/2311.18677) | 2023 | Splits phases across machines with different hardware for prompt and token generation. | 1.4× throughput at 20% lower cost, or 2.35× at equal cost. | Sizing argument for mixed GPU fleets on-prem (e.g. older GPUs for decode). | **Reference** |
| [Mooncake](https://arxiv.org/abs/2407.00079) (FAST'25) | 2024 | KV cache as the central resource: a disaggregated KV pool across CPU, DRAM and SSD, plus a cache-aware scheduler. | Up to +525% throughput (simulated); +75% requests in production (Kimi). | Supports the idea that the gateway should send **prefix and cache-affinity hints** to the serving tier. | **Reference / adopt pattern** |
| [S-LoRA](https://arxiv.org/abs/2311.03285) | 2023 | Adapters live in host memory. One paged pool holds both adapters and KV cache. Custom kernels batch requests that use different adapters. | Up to 4× throughput vs. PEFT and vLLM; thousands of adapters. | Per-tenant fine-tunes on one shared base model. | **Adopt (via vLLM multi-LoRA)** |
| [Punica](https://arxiv.org/abs/2310.18547) | 2023 | SGMV kernel batches different LoRAs over one base model. | 12× throughput, +2 ms/token. | Its kernels are what vLLM's LoRA path uses ([vLLM punica wrapper](https://docs.vllm.ai/en/stable/api/vllm/lora/punica_wrapper/punica_gpu/)). | **Adopt (indirect)** |
| [dLoRA](https://www.usenix.org/conference/osdi24/presentation/wu-bingyang) (OSDI'24) | 2024 | Merges or unmerges adapters on the fly and migrates requests and adapters between replicas. | Up to 57.9× vs. vLLM; 1.8× lower latency vs. S-LoRA. | Placement policy for tenant adapters across replicas. | **Reference** for the adapter-placement scheduler |
| [Fairness in Serving LLMs (VTC)](https://arxiv.org/abs/2401.00588) (OSDI'24) | 2024 | Virtual Token Counter: weighted fair queuing on tokens served, with a counter "lift" for clients that were idle. | Provable 2× bound on service difference between clients. | **Directly usable in the Rust gateway** for fair queuing of tenants on shared self-hosted pools, better than request-count limits. | **Build (in gateway)** |
| [AWQ](https://arxiv.org/abs/2306.00978) (MLSys'24 best paper) | 2023 | Weight-only 4-bit quantization that uses activation statistics to protect the ~1% most important weight channels. | Low accuracy loss at INT4. | Fits larger models into on-prem GPU budgets. | **Adopt** for memory-bound sites |
| [GPTQ](https://arxiv.org/abs/2210.17323) (ICLR'23) | 2022 | One-shot 3–4-bit weight quantization using second-order information. | ~3.25× (A100) and 4.5× (A6000) speedup vs. FP16. | Widely supported format. | **Adopt (format support)** |
| [SmoothQuant](https://arxiv.org/abs/2211.10438) (ICML'23) | 2022 | Moves activation outliers into the weights so both weights and activations run at INT8 (W8A8). | 1.56× speedup, 2× less memory. | Good fit for Ampere-era on-prem GPUs that lack FP8. | **Adopt where FP8 is unavailable** |
| [When to Reason: Semantic Router for vLLM](https://arxiv.org/abs/2510.08731) | 2025 | Classifier decides whether a request needs reasoning mode. | Paper reports accuracy and latency gains (see paper). | Directly matches Caliban's intent classification and model routing. | **Benchmark against** |

FP8 and FP4: supported natively in TensorRT-LLM ([repo](https://github.com/NVIDIA/TensorRT-LLM)) and mistral.rs. gpt-oss ships MXFP4 MoE weights ([repo](https://github.com/openai/gpt-oss)). On Hopper or Blackwell, prefer FP8 over INT8 or INT4 when quality matters.

---

## Rust building blocks

| Crate / project | Use in Caliban | Notes |
|---|---|---|
| [axum](https://github.com/tokio-rs/axum) | Public API (OpenAI and Anthropic routes, admin) | Built on hyper and tower. Middleware is `tower::Service`. SSE through `axum::response::sse`. |
| [tower](https://github.com/tower-rs/tower) | Middleware: timeout, retry, rate limit, **p2c load balancing**, buffer | Helicone uses P2C + PeakEWMA for provider balancing. |
| hyper / reqwest (crates.io) | Upstream HTTP/2 client pools to providers | One pool per provider endpoint. Stream bodies; never buffer whole responses. |
| [Pingora](https://github.com/cloudflare/pingora) ([blog](https://blog.cloudflare.com/pingora-open-source/)) | Optional L7 edge: TLS termination, graceful reload, connection reuse | Apache-2.0, serves 40M+ req/s at Cloudflare. Best for pure proxying. Caliban rewrites request bodies (PII, format translation), so put axum app logic behind it, or skip it at first. |
| [governor](https://github.com/boinkor-net/governor) | Local GCRA rate limiting with keyed limiters | Enforces locally. Needs a Redis or Valkey token bucket for global quotas across replicas. |
| [opentelemetry-rust](https://github.com/open-telemetry/opentelemetry-rust) + `opentelemetry-otlp` | Traces, metrics, logs | Metrics and logs are stable, traces are beta. Emit [GenAI semconv](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/gen-ai-spans.md). |
| [candle](https://github.com/huggingface/candle) | In-process intent classifier, PII NER and embeddings (CPU/CUDA/Metal) | No Python, small binaries. Good for ModernBERT-class encoders. |
| [mistral.rs](https://github.com/EricLBuehler/mistral.rs) | Embedded small-LLM runtime (edge or air-gapped dev SKU) | MIT. OpenAI- and Anthropic-compatible server. GGUF, GPTQ, AWQ, FP8. Per-request LoRA. |
| [Qdrant](https://github.com/qdrant/qdrant) | Semantic cache and RAG vector store | Rust, Apache-2.0. Payload-based multitenancy. Self-hostable for air-gap. |
| [model-transparency / model-signing](https://github.com/sigstore/model-transparency) | Verify model-weight signatures before loading | OpenSSF Model Signing (OMS) v1.0 ([OpenSSF](https://openssf.org/blog/2025/04/04/launch-of-model-signing-v1-0-openssf-ai-ml-working-group-secures-the-machine-learning-supply-chain/), [spec](https://github.com/ossf/model-signing-spec)). Python CLI; verify in an init step or reimplement the check in Rust. |
| Standard crates (crates.io) | `tokio`, `rustls`, `serde_json` / `simd-json`, `tokenizers` (HF), `sqlx` (Postgres), `redis`/`fred`, `rdkafka` or NATS for metering events, `jsonwebtoken`, `aws-lc-rs` (FIPS) | Not separately cited here. Check maintenance status at build time. |

---

## Sovereign / on-prem requirements

### Regulatory

- [ ] **Know Caliban's EU AI Act role.** Caliban serves models it does not train, so it is usually a deployer, distributor or AI-system provider, **not** a GPAI model provider. A modifier only becomes a GPAI provider if the fine-tune uses **more than one-third of the original training compute**. Per-tenant LoRAs are far below that ([EC GPAI guidelines FAQ](https://digital-strategy.ec.europa.eu/en/faqs/guidelines-obligations-general-purpose-ai-providers)). GPAI threshold: >10^23 FLOP. Systemic-risk presumption: >10^25 FLOP.
- [ ] **Dates.** GPAI duties apply from 2025-08-02, with Commission enforcement powers from 2026-08-02 and a 2027-08-02 deadline for models already on the market ([EC FAQ](https://digital-strategy.ec.europa.eu/en/faqs/guidelines-obligations-general-purpose-ai-providers), [Baker McKenzie](https://www.bakermckenzie.com/en/insight/publications/2025/08/general-purpose-ai-obligations)). The GPAI Code of Practice is voluntary but gives a presumption of compliance ([WSGR](https://www.wsgr.com/en/insights/eu-releases-final-code-of-practice-for-general-purpose-ai-models.html)).
- [ ] **Digital Omnibus, Reg. (EU) 2026/1744, in force 2026-07-27.** High-risk deadlines: Annex III systems by **2027-12-02**, Annex I products by **2028-08-02**. **Art. 50 transparency still applies from 2026-08-02** ([Gibson Dunn](https://www.gibsondunn.com/eu-ai-act-omnibus-agreement-postponed-high-risk-deadlines-and-other-key-changes/), [Jones Walker](https://www.joneswalker.com/en/insights/blogs/ai-law-blog/yes-august-2-still-matters-the-eu-approved-a-high-risk-ai-delay-but-most-trans.html?id=102nbon)). Caliban needs: AI-interaction disclosure flags, response metadata to support content marking, and immutable request logs that high-risk deployers can export.
- [ ] **GPAI provenance per model.** Keep a model registry entry with the provider's technical documentation and training-data summary links, so tenants can do their downstream due diligence.
- [ ] **Model licences.** Prefer Apache-2.0 or MIT: Qwen3, Mistral open line, gpt-oss, DeepSeek-V4 (MIT) ([HF overview](https://huggingface.co/blog/daya-shankar/open-source-llm-models-to-run-locally), [gpt-oss](https://github.com/openai/gpt-oss)). **Llama 4's licence excludes EU-domiciled entities** and adds a 700M-MAU clause and "Built with Llama" branding ([licence](https://www.llama.com/llama4/license/), [The Decoder](https://the-decoder.com/meta-releases-first-multimodal-llama-4-models-leaves-eu-out-in-the-cold/)). Block it by default for EU tenants. Gemma uses its own terms of use. The policy engine must enforce licence rules per tenant, not only route by model.
- [ ] **Data residency and retention.** Pin each tenant to a region at both control-plane and data-plane level. Offer a ZDR mode where prompts and completions are never written to disk, only metadata (same idea as OpenRouter's `zdr`). Restrict upstream providers per tenant by region and data-collection policy. Default to no retention of prompt content, as OTel's opt-in content capture also implies.

### Technical

- [ ] **Air-gap bundle.** OCI images plus Helm chart, mirrored to the customer's registry. Model weights as signed artifacts. Engines run with `HF_HUB_OFFLINE=1` and a prefilled `HF_HUB_CACHE` ([HF env vars](https://huggingface.co/docs/huggingface_hub/package_reference/environment_variables)). Disable telemetry (`HF_HUB_DISABLE_TELEMETRY`). No outbound DNS at all.
- [ ] **Weight integrity.** Sign with OMS / Sigstore, or with customer PKI when offline. Use the [Sigstore model-validation operator](https://blog.sigstore.dev/model-validation-operator-v1.0.1/) (init container blocks the pod if verification fails) or an equivalent Caliban check. Signed SBOMs for the Caliban binaries.
- [ ] **AI-BOM.** CycloneDX ML-BOM for CI, SPDX 3.0 AI Profile for regulator-facing documents ([CycloneDX vs SPDX](https://wavect.io/blog/ai-bill-of-materials-cyclonedx-spdx-2026/), [AIBoMGen paper](https://arxiv.org/pdf/2601.05703)).
- [ ] **Build supply chain.** Learn from LiteLLM: pinned dependencies (its Docker image was safe because it pinned), reproducible builds, hermetic CI, no publishing from CI steps that run third-party scanners with publish credentials ([LiteLLM post-mortem](https://docs.litellm.ai/blog/security-update-march-2026)). For the Rust code: `cargo vet` or `cargo deny`, vendored crates in the air-gap build.
- [ ] **Kubernetes and Helm serving stack.** vLLM or SGLang pods behind [llm-d](https://github.com/llm-d/llm-d?tab=readme-ov-file) (CNCF Sandbox; prefix-cache-aware scheduling and prefill/decode split) or [NVIDIA Dynamo](https://github.com/ai-dynamo/dynamo) (KV-aware router, tiered KV cache). Do not use TGI: maintenance mode since Dec 2025, archived 2026-03-21 ([migration guide](https://www.spheron.network/blog/migrate-tgi-to-vllm-sglang-2026/)).
- [ ] **GPU sizing rules of thumb** (estimates; benchmark per site). Weights ≈ params × bytes/param: FP16 = 2, FP8 = 1, INT4 ≈ 0.5–0.6. KV cache per token ≈ 2 × layers × kv_heads × head_dim × bytes. Reserve 20–40% of HBM for KV at the target concurrency. Reference points: gpt-oss-120b fits **1×80 GB** and gpt-oss-20b fits **16 GB** ([gpt-oss](https://github.com/openai/gpt-oss)). Frontier open MoE models (DeepSeek-V4-Pro at ~1.7T params ([HF](https://huggingface.co/deepseek-ai))) need multiple nodes with disaggregation.
- [ ] **Multi-LoRA for tenant fine-tunes:** vLLM `--enable-lora`, `--max-loras`, `--max-lora-rank`. The adapter is selected through the `model` field. **Do not enable runtime adapter loading on shared pools**: vLLM flags it as a security risk outside fully trusted environments ([vLLM LoRA](https://docs.vllm.ai/en/latest/features/lora.html)). Load adapters from the control plane through a signed, allow-listed path only.
- [ ] **Observability without content leakage.** OTel GenAI spans named `{operation} {model}`. Required: `gen_ai.operation.name` and `gen_ai.provider.name`. Usage tokens include cache-read and cache-creation token counts. `gen_ai.response.time_to_first_chunk`. Message content is **opt-in**. The conventions are still in Development status ([spans](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/gen-ai-spans.md), [registry](https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/)). Pin a semconv version.

---

## Risks & failure modes

| Risk | Failure mode | Mitigation |
|---|---|---|
| **PII rehydration over SSE** | A placeholder like `⟦PERSON_3⟧` is split across two stream chunks, or the model changes it (`PERSON 3`, translated). The client then receives raw placeholders or wrongly restored text. | Streaming rehydrator with a hold-back buffer as long as the longest placeholder. Use placeholders the tokenizer keeps intact. Fuzzy-match variants. Add a "fail closed" mode that withholds the chunk. Fuzz-test with real tokenizers. |
| **Cross-tenant cache leakage** | A semantic cache hit returns tenant A's answer to tenant B. Prompt-cache timing side channels. | Tenant ID is part of every cache key and vector-store filter. Per-tenant collections for regulated tenants. Semantic cache off by default for personalised or RAG-grounded answers. Never share prefix cache across tenants on self-hosted engines unless the tenant opts in. |
| **Metering drift** | Client disconnects mid-stream, provider omits usage, retries are double-counted, or reasoning and cached tokens are not priced. | Request `stream_options.include_usage`. On abort, count tokens with a local tokenizer and mark the record "estimated". Use idempotent usage events keyed by `request_id + attempt`. Reconcile with provider invoices daily. Separate pricing fields for cache-read, cache-write and reasoning tokens (as OTel already does). |
| **Retry amplification** | During a provider outage, fallbacks and retries multiply load. Non-idempotent tool calls get repeated. | Retry budgets (token bucket per upstream). Circuit breakers with EWMA health. Never retry once the first byte has reached the client. Hedge requests only when the tenant opts in. |
| **Format translation lossiness** | OpenAI and Anthropic differ on tool calls, thinking blocks, cache_control, citations and multimodal parts, and features silently disappear in translation. | Canonical internal IR that is a superset of both. Translation matrix with explicit "unsupported" errors (no silent drop). Native passthrough when the client and provider formats match. |
| **Noisy neighbour on self-hosted pools** | One tenant's long-context batch drives up everyone's TTFT. | VTC-style weighted fair queueing in the gateway. Per-tenant concurrency caps. Priority classes tied to SLO tier. Admission control on prompt length. |
| **Supply chain** | Compromised dependency, model weights or image (see LiteLLM, March 2026). | Signed artefacts (binaries, Helm, weights), SBOM and AI-BOM, pinned and vendored dependencies, separation of publish credentials. |
| **Licence or regulatory mismatch** | An EU tenant is routed to Llama 4. A model without GPAI documentation gets used in a regulated workflow. | Licence and jurisdiction attributes in the model registry, enforced as routing constraints. |
| **Air-gap drift** | Offline sites fall behind on CVEs and model updates. Licence checks phone home. | Signed offline update bundles. Licence enforced with a signed, time-bound offline token. A "last-known-good" rollback. |
| **Router-model misclassification** | The intent classifier sends a hard query to a cheap model, and quality complaints follow. | Confidence threshold with escalation. Shadow evaluation. Feed tenant feedback back into routing (TensorZero-style bandits). |

---

## Recommended design for Caliban

### 1. Control plane / data plane split

- **Control plane (CP):** tenants, users, API keys, BYOK vault, policies, model registry (licence, region, price, capabilities), routing configs, connector configs, billing, audit and UI. Language is flexible, but Rust keeps one toolchain. Backed by Postgres.
- **Data plane (DP):** a single stateless Rust binary, `caliban-router`, on the request path. It holds no authoritative state. It reads an **immutable, signed, versioned config snapshot** from the CP, xDS-style: pushed over gRPC or pulled from a file or object store. Hot paths use only in-memory data plus Redis or Valkey (quotas, sessions) and the vector DB (cache, RAG).
- If the CP goes down, the DP keeps serving the last snapshot (fail-static). Usage events go to a durable local queue (disk WAL) and flush to the CP or billing pipeline asynchronously.

### 2. Rust data-plane request lifecycle (target added latency: p50 < 3 ms, p99 < 10 ms, excluding classifiers and RAG)

```
TLS/HTTP2 (axum on hyper; optional Pingora edge)
 → 1  AuthN: API key or JWT/mTLS → tenant_id, principal, key scopes        (in-memory key index, argon2/HMAC check, LRU)
 → 2  Tenant policy: allowed models/regions/licences, ZDR flag, budgets,    (config snapshot)
      PII policy, guardrails, SLO tier
 → 3  Pre-admission quota: request-rate GCRA (governor) + token-budget       (Redis Lua token bucket; reserve est. tokens)
      reservation (estimate input tokens with a local tokenizer)
 → 4  Parse → canonical IR (OpenAI / Anthropic / Responses → CalibanIR)
 → 5  Anonymize: NER/regex/dictionary (candle encoder) → reversible vault   (per-request map, encrypted with tenant DEK, TTL)
 → 6  Cache: exact (hash of normalised IR + tenant + model) → semantic      (Qdrant, tenant-filtered; opt-in)
      ↳ hit → stream cached response through steps 10–12
 → 7  Enrich: RAG / ontology context from tenant datasources (other doc)
 → 8  Route: intent classifier (in-process encoder) + policy constraints
      → candidate set → score (cost, latency EWMA, health, prefix affinity,
      tenant prefs) → primary + ordered fallbacks; VTC queue for self-hosted pools
 → 9  Provider adapter: IR → provider wire format; BYOK or pooled credential;
      HTTP/2 pool; retry budget; circuit breaker; no retry after first byte to client
 → 10 Stream: SSE passthrough with backpressure (bounded channels); translate
      chunks back to the client's dialect; capture usage
 → 11 Rehydrate: streaming placeholder restoration with hold-back buffer;
      output guardrails (optional, may add latency)
 → 12 Meter: settle the token reservation with actual usage (incl. cached/reasoning),
      emit idempotent UsageEvent (WAL → Kafka/NATS → ClickHouse/billing)
 → 13 Trace: OTel GenAI span tree (gateway span → classifier, cache, rag,
      provider spans); content capture only if the tenant opts in and not ZDR
```

Implementation notes:
- Every stage is a `tower::Layer`, so each SKU composes its own stack from config.
- Use `Bytes` and zero-copy throughout. Parse JSON only once, into the IR.
- Use cancellation-safe streaming. A client disconnect cancels the upstream request and settles metering as "estimated".

### 3. API surface

| Surface | Endpoints | Notes |
|---|---|---|
| OpenAI-compatible | `/v1/chat/completions`, `/v1/responses`, `/v1/embeddings`, `/v1/models`, `/v1/audio/*` (later) | Full streaming, tools, JSON schema, `stream_options.include_usage`. |
| Anthropic-compatible | `/v1/messages`, `/v1/messages/count_tokens` | Native passthrough to Anthropic. Translated for other providers, with explicit errors for unsupported features. |
| Caliban extensions | Header `x-caliban-*` or body field `caliban: {…}`: `route` (auto, `model:`, `policy:`), `pii` (off, mask, reversible), `cache` (off, exact, semantic), `datasources: [...]`, `node`/`agent` id, `region`, `zdr`, `max_cost`, `slo_tier`, `trace_id` | Ignored by clients that don't know them. Mirrors OpenRouter's `provider` object. Response headers report the routed model, cost and cache status. |
| Virtual models | `model: "caliban/auto"`, `caliban/fast`, `tenant/<alias>` | Aliases are resolved by policy and can be versioned and A/B tested. |
| Admin / CP | REST + gRPC: keys, budgets, policies, BYOK, registry, usage export | Not on the DP hot path. |

### 4. Multi-tenancy isolation model

- **Identity:** org → workspace → project → key. `tenant_id` is carried in all state keys, cache namespaces, vector filters, log partitions and metric labels.
- **Secrets / BYOK:** envelope encryption. Per-tenant DEKs are wrapped by a KEK held in the CP KMS: AWS KMS, GCP KMS or Azure Key Vault for SaaS, HSM or Vault for on-prem. Provider keys are decrypted only in DP memory and never logged.
- **Compute isolation tiers:** (a) shared DP plus shared pools with fair queueing; (b) shared DP plus **dedicated** self-hosted pools or adapters; (c) dedicated DP cell. For SaaS, use **cells**: shard tenants per region into independent stacks to limit blast radius and enforce residency.
- **Data isolation:** cache and RAG namespaces per tenant by default, with separate Qdrant collections for regulated tenants. Shared caching across tenants only for public, non-personalised content with explicit opt-in.
- **Fairness:** VTC weighted by plan tier for self-hosted capacity. Token quotas for third-party providers. Hard budgets in USD.

### 5. Deployment SKUs: one binary, three shapes

| SKU | Data plane | Control plane | Models | Connectivity |
|---|---|---|---|---|
| **SaaS** | Caliban-run multi-tenant cells (EU, US, …) | Caliban-hosted | Third-party APIs + Caliban-hosted open models | Public internet |
| **VPC / dedicated** | Customer cloud or K8s via Helm; single tenant | Caliban-hosted (DP connects **outbound only** for config and usage) **or** self-hosted | Customer BYOK + customer GPUs | Private link; egress to providers allowed per policy |
| **Air-gapped / sovereign** | Customer on-prem K8s or bare metal | **Embedded** CP (same Rust codebase, `--mode=standalone`, local Postgres) | Self-hosted open-weight only; signed weights | None. Offline signed licence and update bundles. |

How the same binary runs in all three: define `ConfigSource` (gRPC stream, file or S3, embedded), `UsageSink` (CP API, Kafka, local Postgres or ClickHouse), `SecretStore` (cloud KMS, Vault, local HSM or PKCS#11) and `ModelProvider` (HTTP providers, local engine pools) as traits. A build-time feature set plus runtime config select the implementation. FIPS build via `aws-lc-rs` for public-sector customers. Same Helm chart, with SKU-specific values. Same OTel output, pointed at a local collector in air-gapped sites.

### 6. On-prem model serving recommendation

- **Default engine:** vLLM (broadest model support, multi-LoRA via Punica/S-LoRA techniques, OpenAI API). **SGLang** for agent and RAG pools with heavy prefix reuse.
- **Orchestration:** llm-d (vendor-neutral, CNCF) as the default. NVIDIA Dynamo for NVIDIA-heavy, multi-node customers. Caliban treats each pool as a `ModelProvider` and passes **prefix-affinity hints**: a session or tenant prefix hash header for the inference scheduler. It does **not** re-implement KV routing.
- **Edge, dev and CPU:** llama.cpp `llama-server` or mistral.rs (could be embedded into the binary for a "Caliban-in-a-box" demo SKU).
- **Starter model set (Apache/MIT, EU-safe):** gpt-oss-120b (1×80 GB) for general reasoning; gpt-oss-20b or Qwen3-8B/30B-class for cheap routing tiers; Mistral (Apache line) for EU-origin preference; DeepSeek-V4 (MIT) only for customers with multi-node GPU budgets. Quantization: FP8 on Hopper or Blackwell, AWQ INT4 for memory-bound sites, SmoothQuant W8A8 on Ampere.
- **Tenant fine-tunes:** LoRA adapters on a shared base model, loaded only from a signed registry. dLoRA-style placement (pin hot adapters, migrate cold ones) is a later optimization.

---

## Open questions

1. **Pingora vs axum at the edge.** Caliban rewrites every request body (PII, IR translation), which makes it an application server rather than a pure proxy. Is Pingora's connection handling worth the added complexity? Benchmark both at 5k concurrent SSE streams.
2. **Rehydration correctness SLA.** What placeholder scheme survives all target models and languages? Measure the restoration failure rate per model, and decide whether a failure is fail-open or fail-closed by default.
3. **Billing truth.** Settle against provider-reported usage or Caliban's own tokenizer counts? How should reasoning and cache tokens from different providers map to one price book?
4. **EU AI Act role for "nodes"/agents.** If tenants build agents on Caliban for Annex III uses (HR, credit), does Caliban become a provider of a high-risk *system* component? What logging and export duties apply by 2027-12-02? This needs legal counsel.
5. **Semantic cache vs. RAG freshness.** How do we invalidate cached answers when the underlying datasource changes? Tie cache entries to datasource version vectors.
6. **Adapter governance.** Who signs tenant LoRAs, and can a malicious adapter attack a shared base model or its neighbours (e.g. through KV or prefix sharing)?
7. **Partner or compete with Agent Router and llm-d?** Shipping Caliban as an ext_proc or inference-extension target could unlock K8s-native buyers.
8. **Offline licensing and updates.** What cadence and format (signed OCI bundles?) work for classified or air-gapped customers? How do we deliver CVE patches within SLA?
9. **Open-weight model churn.** DeepSeek publishes new V4.x checkpoints monthly ([HF](https://huggingface.co/deepseek-ai)). How does the model registry handle evals, licence review and signing without manual work?
10. **Router-model differentiation.** Can Caliban's intent classifier beat Plano's orchestrator and vLLM Semantic Router on a public benchmark? Without proof, "intent routing" is not a moat.

---

### Source index (verified URLs)

Gateways: [OpenRouter routing](https://openrouter.ai/docs/guides/routing/provider-selection) · [OpenRouter BYOK](https://openrouter.ai/docs/guides/overview/auth/byok) · [LiteLLM architecture](https://docs.litellm.ai/docs/proxy/architecture) · [LiteLLM virtual keys](https://docs.litellm.ai/docs/proxy/virtual_keys) · [LiteLLM reliability](https://docs.litellm.ai/docs/proxy/reliability) · [LiteLLM /v1/messages](https://docs.litellm.ai/docs/anthropic_unified) · [LiteLLM incident](https://docs.litellm.ai/blog/security-update-march-2026) · [Datadog on LiteLLM](https://securitylabs.datadoghq.com/articles/litellm-compromised-pypi-teampcp-supply-chain-campaign/) · [Portkey gateway](https://github.com/portkey-ai/gateway) · [Helicone ai-gateway](https://github.com/Helicone/ai-gateway) · [TensorZero](https://github.com/tensorzero/tensorzero) · [TensorZero POMDP post](https://www.tensorzero.com/blog/think-of-llm-applications-as-pomdps-not-agents/) · [TensorZero bandits](https://www.tensorzero.com/blog/bandits-in-your-llm-gateway/) · [Agent Router rename](https://theagentrouter.ai/blog/envoy-ai-gateway-is-now-agent-router/) · [Envoy AI GW token rate limiting](https://aigateway.envoyproxy.io/docs/0.1/capabilities/usage-based-ratelimiting/) · [Kong AI PII Sanitizer](https://developer.konghq.com/plugins/ai-sanitizer.md) · [Cloudflare AI Gateway](https://developers.cloudflare.com/ai-gateway/) · [Cloudflare Infire](https://blog.cloudflare.com/cloudflares-most-efficient-ai-inference-engine/) · [Plano](https://github.com/katanemo/plano) · [vLLM Semantic Router](https://vllm-project.github.io/2025/09/11/semantic-router.html)

Serving: [vLLM LoRA](https://docs.vllm.ai/en/latest/features/lora.html) · [llm-d](https://github.com/llm-d/llm-d?tab=readme-ov-file) · [NVIDIA Dynamo](https://github.com/ai-dynamo/dynamo) · [TensorRT-LLM](https://github.com/NVIDIA/TensorRT-LLM) · [llama.cpp](https://github.com/ggml-org/llama.cpp) · [mistral.rs](https://github.com/EricLBuehler/mistral.rs) · [TGI migration](https://www.spheron.network/blog/migrate-tgi-to-vllm-sglang-2026/) · papers listed in the table above.

Sovereignty: [EC GPAI FAQ](https://digital-strategy.ec.europa.eu/en/faqs/guidelines-obligations-general-purpose-ai-providers) · [Gibson Dunn Omnibus](https://www.gibsondunn.com/eu-ai-act-omnibus-agreement-postponed-high-risk-deadlines-and-other-key-changes/) · [Jones Walker Art. 50](https://www.joneswalker.com/en/insights/blogs/ai-law-blog/yes-august-2-still-matters-the-eu-approved-a-high-risk-ai-delay-but-most-trans.html?id=102nbon) · [OpenSSF model signing](https://openssf.org/blog/2025/04/04/launch-of-model-signing-v1-0-openssf-ai-ml-working-group-secures-the-machine-learning-supply-chain/) · [Sigstore model validation operator](https://blog.sigstore.dev/model-validation-operator-v1.0.1/) · [Llama 4 licence](https://www.llama.com/llama4/license/) · [gpt-oss](https://github.com/openai/gpt-oss) · [HF offline env vars](https://huggingface.co/docs/huggingface_hub/package_reference/environment_variables) · [OTel GenAI spans](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/gen-ai-spans.md)
