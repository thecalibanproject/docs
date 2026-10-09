# 09 — Open-Weight Models On-Prem: Qwen and Friends

*Research snapshot: 2026-10-02. Scope: which open-weight chat, embedding and reranker models an on-prem Caliban site should run, under which licence, on which engine, with which flags, and on how much hardware.*

**Summary.** Qwen is the best-covered open family for Caliban's on-prem SKU. Every current
Qwen line that matters for a gateway is **Apache-2.0**: Qwen3.5 (0.8B–397B), Qwen3.6
(27B dense, 35B-A3B MoE) and **Qwen3.8-27B**. It also ships its own embedding and reranker
models (Qwen3-Embedding / Qwen3-Reranker, 0.6B–8B), official FP8 checkpoints and 262k native
context. **Two Qwen3.8 models are not Apache-2.0** (Qwen3.8-Flash-Next and the 2.4T "Max"). They use
new custom licences that restrict "Model as a Service" businesses, so they must stay out of the
default bundle. Elsewhere, the permissive set is gpt-oss (Apache-2.0), Gemma 4 (now
**Apache-2.0**, unlike Gemma 3), DeepSeek V3.2/V4 (MIT), most of GLM (MIT), IBM Granite 4.x
(Apache-2.0), Phi-4 (MIT), the Mistral Small/Ministral/Devstral-Small line (Apache-2.0) and
NVIDIA Nemotron 3.5 (OpenMDW-1.1). The models to flag are Mistral Medium 3.5 and Devstral 2 123B (revenue cap), MiniMax M2.7
(non-commercial), MiniMax M3, Kimi K3 and GLM-5.3 (custom terms), Llama 4 (EU exclusion for
the multimodal models), EmbeddingGemma and Gemma 3 (Gemma terms), and the Jina v5 embedder and Jina v3 reranker (CC-BY-NC).

On the serving side, three facts matter most to the gateway:
1. **vLLM v0.30.0 returns reasoning in a `reasoning` field.** The field used to be called
   `reasoning_content`. SGLang and llama.cpp still use `reasoning_content`, and Ollama's
   OpenAI endpoint uses `reasoning`.
2. **Prefix-cache isolation per tenant is real in both vLLM and SGLang** through a request
   `cache_salt`. vLLM limits it to 128 characters and rejects `/`, `@`, `\` and NUL, so use
   base64url, not base64.
3. **Qwen3.5+ dropped the `/think` / `/no_think` soft switch.** Thinking is controlled only
   through `chat_template_kwargs.enable_thinking`. Qwen3.8 adds `reasoning_effort`.

**Method.** Every model ID, licence, parameter count and context length below was read from the
Hugging Face API (`/api/models/<id>`: `cardData.license`, `safetensors.total`, and `config.json`)
or from the model card on 2026-10-02. Engine flags come from the source at release tags:
[vLLM v0.30.0](https://github.com/vllm-project/vllm/tree/v0.30.0) (2026-09-22),
[SGLang v0.5.21](https://github.com/sgl-project/sglang/tree/v0.5.21) (2026-10-02),
[llama.cpp b11346](https://github.com/ggml-org/llama.cpp/tree/b11346) (container `server-b11347`),
[Ollama v0.35.0](https://github.com/ollama/ollama/tree/v0.35.0) and
[TEI v1.9.4](https://github.com/huggingface/text-embeddings-inference/tree/v1.9.4).
Sizes are on-disk safetensors or GGUF sizes from the HF tree API. VRAM figures are **estimates**.

---

## 1. The Qwen lineup (October 2026)

All repos are under [huggingface.co/Qwen](https://huggingface.co/Qwen). "Hybrid attn" means
3 Gated DeltaNet (linear attention) layers per 1 full-attention layer. Only the full-attention
layers hold a per-token KV cache, so long contexts are much cheaper than the parameter count
suggests (see §4).

### 1.1 Chat / instruct models

| Repo | Released | Arch | Params (total / active) | Native ctx | Thinking | Licence | Official quants |
|---|---|---|---|---|---|---|---|
| [Qwen3.8-27B](https://huggingface.co/Qwen/Qwen3.8-27B) | 2026-08 | dense, hybrid attn, native VL | 27.8B | 262,144 (YaRN → 1M) | on by default; `enable_thinking`, `reasoning_effort` (`xhigh`/`medium`/`low`), `preserve_thinking` (on by default) | **apache-2.0** | [FP8](https://huggingface.co/Qwen/Qwen3.8-27B-FP8) (30.9 GB) |
| [Qwen3.8-Flash-Next](https://huggingface.co/Qwen/Qwen3.8-Flash-Next) | 2026-08 | MoE, VL | 125B / 6B (+51B n-gram embedding, 4B MTP; HF counts 180B) | 262,144 | hybrid | **other: `qwen-community-1.0`** ⚠ | [FP8](https://huggingface.co/Qwen/Qwen3.8-Flash-Next-FP8) |
| [Qwen3.8-2.4T-A95B](https://huggingface.co/Qwen/Qwen3.8-2.4T-A95B) | 2026-08 | MoE | 2.4T / 95B | 262,144 | thinking | **other: `qwen3.8-max`** ⚠ | [FP8](https://huggingface.co/Qwen/Qwen3.8-2.4T-A95B-FP8) |
| [Qwen3.6-35B-A3B](https://huggingface.co/Qwen/Qwen3.6-35B-A3B) | 2026-04 | MoE (256 experts, 8 routed + 1 shared), hybrid attn, VL | 36B / 3B | 262,144 (→ 1,010,000) | on by default; `enable_thinking`, `preserve_thinking` | apache-2.0 | [FP8](https://huggingface.co/Qwen/Qwen3.6-35B-A3B-FP8) (37.5 GB) |
| [Qwen3.6-27B](https://huggingface.co/Qwen/Qwen3.6-27B) | 2026-04 | dense, hybrid attn, VL | 27.8B | 262,144 | as above | apache-2.0 | [FP8](https://huggingface.co/Qwen/Qwen3.6-27B-FP8) (30.9 GB) |
| Qwen3.5 [0.8B](https://huggingface.co/Qwen/Qwen3.5-0.8B) · [2B](https://huggingface.co/Qwen/Qwen3.5-2B) · [4B](https://huggingface.co/Qwen/Qwen3.5-4B) · [9B](https://huggingface.co/Qwen/Qwen3.5-9B) | 2026-02/03 | dense, hybrid attn, VL | 0.87 / 2.3 / 4.7 / 9.7B | 262,144 | on by default; `enable_thinking` | apache-2.0 | none from Qwen for ≤9B (community: e.g. [RedHatAI w4a16](https://huggingface.co/RedHatAI/Qwen3.5-9B-quantized.w4a16)) |
| Qwen3.5 [27B](https://huggingface.co/Qwen/Qwen3.5-27B) · [35B-A3B](https://huggingface.co/Qwen/Qwen3.5-35B-A3B) · [122B-A10B](https://huggingface.co/Qwen/Qwen3.5-122B-A10B) · [397B-A17B](https://huggingface.co/Qwen/Qwen3.5-397B-A17B) | 2026-02 | dense / MoE, hybrid attn, VL | 27.8B; 36B/3B; 125B/10B; 403B/17B | 262,144 | as above | apache-2.0 | FP8 and GPTQ-Int4 for all four (e.g. [122B-A10B-FP8](https://huggingface.co/Qwen/Qwen3.5-122B-A10B-FP8) 127 GB, [397B-A17B-FP8](https://huggingface.co/Qwen/Qwen3.5-397B-A17B-FP8) 406 GB, [35B-A3B-GPTQ-Int4](https://huggingface.co/Qwen/Qwen3.5-35B-A3B-GPTQ-Int4) 24.4 GB) |
| [Qwen3-Coder-Next](https://huggingface.co/Qwen/Qwen3-Coder-Next) | 2026-02 | MoE (`qwen3_next`) | 80B / 3B | 262,144 | non-thinking | apache-2.0 | [FP8](https://huggingface.co/Qwen/Qwen3-Coder-Next-FP8), [GGUF](https://huggingface.co/Qwen/Qwen3-Coder-Next-GGUF) |
| [Qwen3-Next-80B-A3B-Instruct](https://huggingface.co/Qwen/Qwen3-Next-80B-A3B-Instruct) / [-Thinking](https://huggingface.co/Qwen/Qwen3-Next-80B-A3B-Thinking) | 2025-09 | MoE, hybrid attn | 81B / 3B | 262,144 | split checkpoints | apache-2.0 | FP8, GGUF |
| Qwen3-2507: [4B-Instruct](https://huggingface.co/Qwen/Qwen3-4B-Instruct-2507), [30B-A3B-Instruct](https://huggingface.co/Qwen/Qwen3-30B-A3B-Instruct-2507), [30B-A3B-Thinking](https://huggingface.co/Qwen/Qwen3-30B-A3B-Thinking-2507), [235B-A22B-Instruct/Thinking](https://huggingface.co/Qwen/Qwen3-235B-A22B-Instruct-2507) | 2025-07/08 | dense / MoE | 4B; 30.5B/3B; 235B/22B | 262,144 | **split**: Instruct = no thinking, Thinking = always (the template injects `<think>`) | apache-2.0 | FP8 |
| [Qwen3-Coder-30B-A3B-Instruct](https://huggingface.co/Qwen/Qwen3-Coder-30B-A3B-Instruct), [Qwen3-Coder-480B-A35B-Instruct](https://huggingface.co/Qwen/Qwen3-Coder-480B-A35B-Instruct) | 2025-07 | MoE | 30.5B/3B; 480B/35B | 262,144 | non-thinking | apache-2.0 | FP8 |
| Qwen3 (original): [0.6B](https://huggingface.co/Qwen/Qwen3-0.6B) … [8B](https://huggingface.co/Qwen/Qwen3-8B), [14B](https://huggingface.co/Qwen/Qwen3-14B), [32B](https://huggingface.co/Qwen/Qwen3-32B), [30B-A3B](https://huggingface.co/Qwen/Qwen3-30B-A3B), [235B-A22B](https://huggingface.co/Qwen/Qwen3-235B-A22B) | 2025-04 | dense / MoE | — | **32,768** native (131,072 with YaRN) | **hybrid**: `enable_thinking` + `/think` `/no_think` soft switch | apache-2.0 | FP8, AWQ, GPTQ, GGUF |
| Qwen3-VL [2B/4B/8B/32B](https://huggingface.co/Qwen/Qwen3-VL-8B-Instruct), [30B-A3B](https://huggingface.co/Qwen/Qwen3-VL-30B-A3B-Instruct), [235B-A22B](https://huggingface.co/Qwen/Qwen3-VL-235B-A22B-Instruct) | 2025-09/10 | dense / MoE, VL | — | 262,144 | split Instruct / Thinking | apache-2.0 | FP8, GGUF |
| [Qwen3Guard-Gen](https://huggingface.co/Qwen/Qwen3Guard-Gen-8B) / [-Stream](https://huggingface.co/Qwen/Qwen3Guard-Stream-4B) 0.6/4/8B | 2025-09 | safety classifiers | — | — | — | apache-2.0 | — |

⚠ **Qwen3.8-Flash-Next** ([LICENSE](https://huggingface.co/Qwen/Qwen3.8-Flash-Next/blob/main/LICENSE),
"Qwen Community License 1.0"): if the licensee or an affiliate runs a "Model as a Service" or
"AI Work Assistant" business, it needs a separate licence from Qwen for any commercial use.
Internal use is exempt only when the model and its outputs are not exposed to third parties.
Products above 100M MAU or US$20M monthly revenue must also display the model name.
**Qwen3.8-2.4T-A95B** ("Qwen3.8-Max License") has the same structure, but the MaaS clause only
triggers above US$50M revenue over 12 months. Caliban is literally a gateway, so whether a
customer counts as "MaaS" is a legal question. Keep both out of the default bundle
(`bundle.sh` refuses `other`).

**Qwen3.8-27B vs. Qwen3.6.** Qwen3.8-27B (2026-08) uses the same architecture as Qwen3.6-27B
(`model_type: qwen3_5`, 64 layers, 16 full-attention layers). It is the newest Apache-2.0 Qwen.
The only Apache-2.0 *small MoE* is still **Qwen3.6-35B-A3B**: 3B active parameters, which makes
it the fastest decoder per GPU and the best fit for CPU.

### 1.2 Embedding and reranker models

| Repo | Params | Max tokens | Output | Licence | Engines |
|---|---|---|---|---|---|
| [Qwen3-Embedding-0.6B](https://huggingface.co/Qwen/Qwen3-Embedding-0.6B) | 0.6B (1.2 GB) | 32k | 1024-d, MRL 32–1024, instruction-aware (queries need an `Instruct: …\nQuery:` prefix) | apache-2.0 | vLLM (`--runner pooling`), TEI (GPU and CPU), llama.cpp ([official GGUF](https://huggingface.co/Qwen/Qwen3-Embedding-0.6B-GGUF), `--embedding --pooling last`), Ollama `qwen3-embedding:0.6b` |
| [Qwen3-Embedding-4B](https://huggingface.co/Qwen/Qwen3-Embedding-4B) / [8B](https://huggingface.co/Qwen/Qwen3-Embedding-8B) | 4.0B (8 GB) / 7.6B (15 GB) | 32k | 2560-d / 4096-d, MRL | apache-2.0 | vLLM, TEI |
| [Qwen3-Reranker-0.6B](https://huggingface.co/Qwen/Qwen3-Reranker-0.6B) / [4B](https://huggingface.co/Qwen/Qwen3-Reranker-4B) / [8B](https://huggingface.co/Qwen/Qwen3-Reranker-8B) | 0.6 / 4.0 / 8.2B | 32k | yes/no logit score | apache-2.0 | **vLLM only** (needs `--hf-overrides`, below). Not in TEI's reranker list. |
| [Qwen3-VL-Embedding-2B/8B](https://huggingface.co/Qwen/Qwen3-VL-Embedding-2B), [Qwen3-VL-Reranker-2B/8B](https://huggingface.co/Qwen/Qwen3-VL-Reranker-2B) | 2.1B / 8B | 262k | multimodal | apache-2.0 | vLLM (`Qwen3VLForSequenceClassification` override for the reranker) |
| [BAAI/bge-m3](https://huggingface.co/BAAI/bge-m3) | 0.57B | 8192 | 1024-d dense + sparse + ColBERT | mit | vLLM (`BgeM3EmbeddingModel`), TEI, Ollama `bge-m3` |
| [BAAI/bge-reranker-v2-m3](https://huggingface.co/BAAI/bge-reranker-v2-m3) | 0.57B (2.3 GB) | 8192 | cross-encoder | apache-2.0 | TEI (XLM-RoBERTa: GPU **and CPU**), vLLM |
| [ibm-granite/granite-embedding-311m-multilingual-r2](https://huggingface.co/ibm-granite/granite-embedding-311m-multilingual-r2) / [97m](https://huggingface.co/ibm-granite/granite-embedding-97m-multilingual-r2) | 0.31B / 0.09B | 32k | ModernBERT | apache-2.0 | TEI (ModernBERT) |
| [Snowflake/snowflake-arctic-embed-l-v2.0](https://huggingface.co/Snowflake/snowflake-arctic-embed-l-v2.0) | 0.57B | 8192 | — | apache-2.0 | TEI, vLLM |
| [Alibaba-NLP/gte-multilingual-reranker-base](https://huggingface.co/Alibaba-NLP/gte-multilingual-reranker-base), [mixedbread-ai/mxbai-rerank-large-v2](https://huggingface.co/mixedbread-ai/mxbai-rerank-large-v2) | 0.3B / 1.5B | 8k / 32k | — | apache-2.0 | TEI / vLLM (with override) |
| ⚠ [jinaai/jina-embeddings-v5-text-small](https://huggingface.co/jinaai/jina-embeddings-v5-text-small), [jinaai/jina-reranker-v3](https://huggingface.co/jinaai/jina-reranker-v3) | 0.6B | — | — | **cc-by-nc-4.0** (non-commercial) | exclude |
| ⚠ [google/embeddinggemma-300m](https://huggingface.co/google/embeddinggemma-300m) | 0.3B | — | — | **gemma** (Gemma terms, gated) | exclude by default |

**Embedding dimension is a schema decision.** Qdrant collections are created for one
dimension. Switching from Qwen3-Embedding-0.6B (1024) to 8B (4096) means re-indexing, unless
you use MRL truncation to 1024. Pick one embedder per deployment and keep it.

---

## 2. Other open-weight families (October 2026)

| Family / repo | Arch, params (total / active) | Native ctx | Licence | Notes |
|---|---|---|---|---|
| [openai/gpt-oss-20b](https://huggingface.co/openai/gpt-oss-20b) / [gpt-oss-120b](https://huggingface.co/openai/gpt-oss-120b) | MoE, MXFP4 experts; 21B/3.6B and 117B/5.1B | 131,072 | apache-2.0 | 120b "fits into a single 80GB GPU" (card). Always reasons; effort low/medium/high. Also [gpt-oss-safeguard-20b/120b](https://huggingface.co/openai/gpt-oss-safeguard-20b) (apache-2.0). |
| [mistralai/Mistral-Small-4-119B-2603](https://huggingface.co/mistralai/Mistral-Small-4-119B-2603) | MoE, 128 experts / 4 active, 119B, VL | 256k (card) | apache-2.0 | `reasoning_effort` `none`/`high` per request. Card: TP2. |
| [Ministral-3 3B/8B/14B](https://huggingface.co/mistralai/Ministral-3-14B-Instruct-2512) (Instruct, Reasoning, Base; 2512) | dense, VL | 262,144 | apache-2.0 | Official GGUF repos (e.g. [14B-Instruct-2512-GGUF](https://huggingface.co/mistralai/Ministral-3-14B-Instruct-2512-GGUF)). |
| [Devstral-Small-2-24B-Instruct-2512](https://huggingface.co/mistralai/Devstral-Small-2-24B-Instruct-2512), [Magistral-Small-2509](https://huggingface.co/mistralai/Magistral-Small-2509), [Mistral-Small-3.2-24B-Instruct-2506](https://huggingface.co/mistralai/Mistral-Small-3.2-24B-Instruct-2506) | dense 24B | 393k / 128k / 131k | apache-2.0 | |
| [Mistral-Large-3-675B-Instruct-2512](https://huggingface.co/mistralai/Mistral-Large-3-675B-Instruct-2512), [Leanstral-1.5-119B-A6B](https://huggingface.co/mistralai/Leanstral-1.5-119B-A6B) | MoE 675B; 119B/6B (Lean prover) | — | apache-2.0 | |
| ⚠ [Mistral-Medium-3.5-128B](https://huggingface.co/mistralai/Mistral-Medium-3.5-128B), [Devstral-2-123B-Instruct-2512](https://huggingface.co/mistralai/Devstral-2-123B-Instruct-2512) | dense ~125B | 262,144 | **"Modified MIT"**: no rights at all if company monthly revenue > **US$20M** | Not permissive for enterprises. Exclude. |
| [deepseek-ai/DeepSeek-V4-Flash](https://huggingface.co/deepseek-ai/DeepSeek-V4-Flash) ([-0731](https://huggingface.co/deepseek-ai/DeepSeek-V4-Flash-0731)) / [V4-Pro](https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro) ([-0813](https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro-0813)) | MoE 284B/13B; 1.6T/49B; FP4 experts + FP8 | 1M | mit | V4-Flash is ~160 GB on disk. 3 reasoning modes (non-think / high / max). |
| [DeepSeek-V4.1-Flash](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash) (2026-09) | multimodal MoE, 552B backbone (+196B Engram memory) | 1M | mit | Needs the new `deepseek_v41` parsers. |
| [DeepSeek-V3.2](https://huggingface.co/deepseek-ai/DeepSeek-V3.2), [R1-0528-Qwen3-8B](https://huggingface.co/deepseek-ai/DeepSeek-R1-0528-Qwen3-8B) | 685B MoE; 8B distill | 163,840; 131,072 | mit | |
| Gemma 4: [E2B](https://huggingface.co/google/gemma-4-E2B-it), [E4B](https://huggingface.co/google/gemma-4-E4B-it), [12B](https://huggingface.co/google/gemma-4-12B-it), [26B-A4B](https://huggingface.co/google/gemma-4-26B-A4B-it), [31B](https://huggingface.co/google/gemma-4-31B-it) | dense + one MoE (25.8B / 3.8B active), multimodal | 128k (E2B/E4B), 256k (others) | **apache-2.0** ([licence page](https://ai.google.dev/gemma/docs/gemma_4_license)) | **Change from Gemma 3**, which uses the Gemma terms and is gated. Official QAT `q4_0` GGUF and `w4a16` repos. Thinking off by default. |
| GLM: [GLM-4.7-Flash](https://huggingface.co/zai-org/GLM-4.7-Flash) (30B-A3B), [GLM-4.5-Air](https://huggingface.co/zai-org/GLM-4.5-Air) (110B), [GLM-4.7](https://huggingface.co/zai-org/GLM-4.7) (358B), [GLM-5](https://huggingface.co/zai-org/GLM-5)/[5.1](https://huggingface.co/zai-org/GLM-5.1)/[5.2](https://huggingface.co/zai-org/GLM-5.2) (~754B), [GLM-5.3-Flash](https://huggingface.co/zai-org/GLM-5.3-Flash), [GLM-4.6V-Flash](https://huggingface.co/zai-org/GLM-4.6V-Flash) (10B VL) | MoE | 131k–1M | mit | GLM-4.7-Flash is a strong 1×80 GB candidate (62 GB BF16). |
| ⚠ [zai-org/GLM-5.3](https://huggingface.co/zai-org/GLM-5.3) | MoE ~753B | 1M | **other: `glm-5.3`** | A MaaS operator with > US$10B revenue must pass Z.AI's security review. Practically permissive, but not SPDX. Needs legal review. |
| IBM Granite 4.2: [3b](https://huggingface.co/ibm-granite/granite-4.2-3b), [8b](https://huggingface.co/ibm-granite/granite-4.2-8b), [30b](https://huggingface.co/ibm-granite/granite-4.2-30b) (2026-08) | dense | 131,072 | apache-2.0 | Official FP8, MXFP4, NVFP4 and GGUF. Thinking on by default (`enable_thinking`). |
| Phi: [phi-4](https://huggingface.co/microsoft/phi-4) (14.7B, 16k), [Phi-4-mini-instruct](https://huggingface.co/microsoft/Phi-4-mini-instruct) (3.8B, 128k), [Phi-4-reasoning(-plus)](https://huggingface.co/microsoft/Phi-4-reasoning-plus) (32k), [Phi-4-reasoning-vision-15B](https://huggingface.co/microsoft/Phi-4-reasoning-vision-15B) (2026-01) | dense | see left | mit | No Phi-5 on HF as of 2026-10-02. |
| [nvidia/NVIDIA-Nemotron-3.5-Lightning-30B-A3B-BF16](https://huggingface.co/nvidia/NVIDIA-Nemotron-3.5-Lightning-30B-A3B-BF16) (2026-08) | hybrid Mamba MoE, 31.6B/3B | 262,144 | **other: OpenMDW-1.1** (permissive: attribution and a patent-retaliation clause, no use restrictions) | Add `openmdw-1.1` to the allow-list after review. [Nemotron-3-Nano-30B-A3B](https://huggingface.co/nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B-BF16) uses the *NVIDIA Nemotron Open Model License*. |
| ⚠ Llama: [Llama-4-Scout](https://huggingface.co/meta-llama/Llama-4-Scout-17B-16E-Instruct) / Maverick, [Llama-3.3-70B](https://huggingface.co/meta-llama/Llama-3.3-70B-Instruct) | MoE 109B/17B; dense 70B | — | **llama4** / **llama3.3** (gated) | The [Llama 4 AUP](https://www.llama.com/llama4/use-policy/) withholds the §1(a) rights for **multimodal** Llama 4 models from EU-domiciled individuals and companies. The licence has a 700M-MAU clause. No newer Meta open weights on HF since 2025-04. |
| ⚠ [moonshotai/Kimi-K2.6](https://huggingface.co/moonshotai/Kimi-K2.6) (1T), [Kimi-K3](https://huggingface.co/moonshotai/Kimi-K3) (2.8T) | MoE | 262k / 1M | K2.6: modified MIT (display "Kimi K2.6" above 100M MAU or US$20M monthly revenue). K3: **custom**, MaaS > US$20M/yr needs an agreement. | [Kimi-Linear-48B-A3B-Instruct](https://huggingface.co/moonshotai/Kimi-Linear-48B-A3B-Instruct) is plain MIT. |
| ⚠ [MiniMaxAI/MiniMax-M2.7](https://huggingface.co/MiniMaxAI/MiniMax-M2.7), [MiniMax-M3](https://huggingface.co/MiniMaxAI/MiniMax-M3) | MoE 229B; 427B | 205k; 1M | M2.7: **NON-COMMERCIAL**. M3: MiniMax Community (commercial use needs a notice, or authorization above US$20M/yr) | Exclude. |

**Licence gate for Caliban.** The default bundle allows `apache-2.0` and `mit` only, which
`bundle.sh` enforces. `openmdw-1.1` (Nemotron) and `glm-5.3` are candidates for
`--allow-licence` after legal review. Everything marked ⚠ stays out.

---

## 3. Tool calling and reasoning per family

### 3.1 Engine flags (verified against the parser registries)

vLLM v0.30.0 registers its tool parsers in
[`vllm/tool_parsers/__init__.py`](https://github.com/vllm-project/vllm/blob/v0.30.0/vllm/tool_parsers/__init__.py)
and its reasoning parsers in [`vllm/reasoning/__init__.py`](https://github.com/vllm-project/vllm/blob/v0.30.0/vllm/reasoning/__init__.py).
SGLang v0.5.21 registers them in
[`function_call_parser.py`](https://github.com/sgl-project/sglang/blob/v0.5.21/python/sglang/srt/function_call/function_call_parser.py)
and [`reasoning_parser.py`](https://github.com/sgl-project/sglang/blob/v0.5.21/python/sglang/srt/parser/reasoning_parser.py).
Tool calling in vLLM always needs `--enable-auto-tool-choice` together with `--tool-call-parser`.

| Model family | vLLM `--tool-call-parser` | vLLM `--reasoning-parser` | SGLang `--tool-call-parser` / `--reasoning-parser` | Source |
|---|---|---|---|---|
| Qwen3.5 / 3.6 / 3.8 | `qwen3_coder` (card) or `qwen3_xml` (vLLM recipe). Both map to the same `qwen3_engine_tool_parser` in v0.30. | `qwen3` | `qwen3_coder` / `qwen3` | [Qwen3.6 card](https://huggingface.co/Qwen/Qwen3.6-35B-A3B), [vLLM recipe](https://recipes.vllm.ai/Qwen/Qwen3.8-27B) |
| Qwen3 (orig.), Qwen3-2507 | `hermes` | `qwen3` (cards also show `deepseek_r1`) | `qwen25` / `qwen3`; `qwen3-thinking` for -Thinking-2507 | [vLLM tool docs](https://github.com/vllm-project/vllm/blob/v0.30.0/docs/features/tool_calling.md#qwen-models) |
| Qwen3-Coder(-Next), Qwen3-Coder-30B/480B | `qwen3_xml` (= `qwen3_coder`) | — | `qwen3_coder` | vLLM tool docs |
| gpt-oss | `openai` | built-in harmony handling (recipe uses no flag; `openai_gptoss` is registered) | `gpt-oss` / `gpt-oss` | [vLLM gpt-oss recipe](https://recipes.vllm.ai/openai/gpt-oss-20b) |
| Mistral Small 4, Ministral 3, Devstral | `mistral` (+ `--tokenizer-mode mistral --config-format mistral --load-format mistral` for the Mistral-format checkpoints) | `mistral` (reasoning variants, Small 4) | `mistral` / `mistral` | [Ministral-3 card](https://huggingface.co/mistralai/Ministral-3-14B-Instruct-2512), [Mistral-Small-4 card](https://huggingface.co/mistralai/Mistral-Small-4-119B-2603) |
| DeepSeek-V4 / V4.1 | `deepseek_v4` / `deepseek_v41` | `deepseek_v4` / `deepseek_v41` | `deepseekv4` / `deepseek-v4` (and `v41`) | [vLLM recipe](https://recipes.vllm.ai/deepseek-ai/DeepSeek-V4-Flash) |
| DeepSeek-V3.1 / V3.2 | `deepseek_v31` / `deepseek_v32` | `deepseek_v3` | `deepseekv31` / `deepseekv32` | registry |
| Gemma 4 | `gemma4` | `gemma4` | `gemma4` / `gemma4` | [vLLM reasoning docs](https://github.com/vllm-project/vllm/blob/v0.30.0/docs/features/reasoning_outputs.md) |
| GLM-4.5 / 4.6 / 4.7 / 5.x | `glm45`, `glm47` | `glm45` (also registered as `glm47`) | `glm45`, `glm47` / `glm45` | [GLM-4.7-Flash card](https://huggingface.co/zai-org/GLM-4.7-Flash) |
| Granite 4.2 | `qwen3_coder` (per IBM card; `granite4` is for Granite 4.0/4.1) | plugin `granite_thinking_parser` shipped in the model card (`--reasoning-parser-plugin`) | `qwen3_coder` / `granite_thinking_parser` (built in) | [Granite 4.2 card](https://huggingface.co/ibm-granite/granite-4.2-8b) |
| Phi-4-mini | `phi4_mini_json` | — | — | registry |
| Nemotron 3 / 3.5 | `qwen3_coder` | `nemotron_v3` | `qwen3_coder` / `nemotron_3` | [Nemotron 3.5 card](https://huggingface.co/nvidia/NVIDIA-Nemotron-3.5-Lightning-30B-A3B-BF16) |
| Kimi K2 / K3, MiniMax M2 / M3 | `kimi_k2`, `kimi_k3`, `minimax_m2`, `minimax_m3` | same names (`minimax_m2_append_think` variant) | `kimi_k2`, `kimi_k3`, `minimax-m2`, `minimax-m3` | registry |
| Llama 3.x / 4 | `llama3_json`, `llama4_pythonic` (with the example chat template) | — | `llama3` / — | vLLM tool docs |

### 3.2 How reasoning is switched and returned

| Family | Default | Per-request control | Soft switch in prompt |
|---|---|---|---|
| Qwen3 (2025-04 hybrid) | thinking on | `chat_template_kwargs: {"enable_thinking": false}` | **yes**: `/think`, `/no_think` in user or system turns (works only while `enable_thinking=True`) |
| Qwen3-2507 Instruct / Thinking | fixed: off / on | none (separate checkpoints). The Thinking template inserts `<think>`, so the raw output has only `</think>`. | no |
| Qwen3.5 / 3.6 | thinking on | `chat_template_kwargs: {"enable_thinking": false}`; `preserve_thinking: true` keeps past reasoning (3.6) | **no**. Cards: "does not officially support the soft switch of Qwen3, i.e., `/think` and `/nothink`." |
| Qwen3.8-27B | thinking on, `preserve_thinking` on | `enable_thinking`, plus `reasoning_effort` = `xhigh` (default) / `medium` / `low` | no |
| gpt-oss | always reasons | `reasoning_effort` low / medium / high | — |
| Gemma 4 | thinking **off** | `enable_thinking: true`, or any `reasoning_effort` (vLLM auto-injects `enable_thinking`) | `<\|think\|>` token in the system prompt |
| Mistral Small 4 | none | `reasoning_effort` `none` / `high` | — |
| DeepSeek-V4 | — | vLLM: `reasoning_effort` auto-injects `enable_thinking` (V4-Pro listed as needing it) | — |
| Granite 4.2, Nemotron 3.5, GLM-4.7 | thinking on | `chat_template_kwargs: {"enable_thinking": false}` (GLM template also has `clear_thinking`) | — |

**Where the reasoning comes back.** This needs care in Caliban's response normaliser:

| Server | With a reasoning parser enabled | Without one |
|---|---|---|
| vLLM v0.30.0 | `message.reasoning` / `delta.reasoning`. Renamed from `reasoning_content`; the [docs](https://github.com/vllm-project/vllm/blob/v0.30.0/docs/features/reasoning_outputs.md) warn that old clients "could silently read an empty `reasoning_content`". `include_reasoning: false` drops it. `thinking_token_budget` caps it. | `<think>…</think>` stays inline in `content` |
| SGLang v0.5.21 | `reasoning_content` (`separate_reasoning: true` is the default) | inline |
| llama.cpp `llama-server` | `reasoning_content` (`--reasoning-format` defaults to `auto`; `deepseek` → `reasoning_content`, `none` → inline, `deepseek-legacy` → both) | — |
| Ollama v0.35 (OpenAI endpoint) | `reasoning` ([openai.go](https://github.com/ollama/ollama/blob/v0.35.0/openai/openai.go)); the native API uses `message.thinking` | — |

Caliban should read `reasoning` **or** `reasoning_content` from any OpenAI-compatible upstream,
and emit one canonical field to clients. Set `inline_think_tags = true` only for a deployment
that runs without a reasoning parser. That includes llama.cpp with `--reasoning-format none`,
and any vLLM pool where the parser was left off.

**How to pass the controls.**
- vLLM, SGLang and llama.cpp accept `chat_template_kwargs` in the request body.
- vLLM also accepts a server default `--default-chat-template-kwargs '{"enable_thinking": false}'`.
  A request-level value overrides it.
- Ollama's OpenAI endpoint does **not** take `chat_template_kwargs`. It maps `reasoning_effort`
  (`"none"` → thinking off). So the per-model `reasoning_control` must reflect the *server*,
  not only the model family.

---

## 4. Quantization and hardware tiers

### 4.1 Official quantized repos

| Family | FP8 | INT4 (AWQ/GPTQ) | GGUF | Other |
|---|---|---|---|---|
| Qwen3.5 / 3.6 / 3.8 | Qwen: 27B, 35B-A3B, 122B-A10B, 397B-A17B, 3.6-27B/35B-A3B, 3.8-27B | Qwen GPTQ-Int4 for 3.5 27B/35B-A3B/122B/397B only. Community AWQ for 3.6 (e.g. [cyankiwi/Qwen3.6-27B-AWQ-INT4](https://huggingface.co/cyankiwi/Qwen3.6-27B-AWQ-INT4), [QuantTrio/Qwen3.6-35B-A3B-AWQ](https://huggingface.co/QuantTrio/Qwen3.6-35B-A3B-AWQ)). | llama.cpp org: [ggml-org/Qwen3.6-35B-A3B-GGUF](https://huggingface.co/ggml-org/Qwen3.6-35B-A3B-GGUF) (Q4_K_M 20.4 GB, Q8_0 36.9 GB), [ggml-org/Qwen3.8-27B-GGUF](https://huggingface.co/ggml-org/Qwen3.8-27B-GGUF) (Q4_K_M 19.0 GB). Community: [unsloth/Qwen3.5-9B-GGUF](https://huggingface.co/unsloth/Qwen3.5-9B-GGUF) (Q4_K_M 5.7 GB). | NVFP4: [nvidia/Qwen3.6-35B-A3B-NVFP4](https://huggingface.co/nvidia/Qwen3.6-35B-A3B-NVFP4), [nvidia/Qwen3.8-27B-NVFP4](https://huggingface.co/nvidia/Qwen3.8-27B-NVFP4) |
| Qwen3 (2025) | FP8 | AWQ, GPTQ-Int4/Int8 | Qwen GGUF | MLX |
| gpt-oss | — (MXFP4 native) | — | [ggml-org/gpt-oss-20b-GGUF](https://huggingface.co/ggml-org/gpt-oss-20b-GGUF) (MXFP4, 12.1 GB) | |
| Gemma 4 | — | Google `-qat-w4a16-ct` | Google `-qat-q4_0-gguf` | |
| Granite 4.2 | IBM `-fp8` | — | IBM `-GGUF` | `-mxfp4`, `-nvfp4` |
| Mistral | Ministral/Devstral/Small-4 ship FP8 weights alongside BF16 | — | Mistral `-GGUF` for Ministral 3 and Magistral | Small-4 `-NVFP4` |
| DeepSeek V4 | FP8 + FP4 mixed natively | — | ggml-org GGUF for V4-Flash | NVIDIA NVFP4 |

**Engine support by GPU generation** (from the [vLLM quantization matrix](https://github.com/vllm-project/vllm/blob/v0.30.0/docs/features/quantization/README.md)):
- FP8 W8A8 (llm-compressor) needs Ada or Hopper and newer.
- Marlin kernels serve GPTQ, AWQ, FP8 and FP4 *weights* on Ampere and newer.
- Online quantization of a BF16 checkpoint: `--quantization fp8_per_tensor | fp8_per_block | mxfp8 | mxfp4` ([docs](https://github.com/vllm-project/vllm/blob/v0.30.0/docs/features/quantization/online.md)).

### 4.2 KV-cache arithmetic for the recommended models

KV bytes per token = 2 × (full-attention layers) × KV heads × head_dim × 2 B (BF16).
The layer counts below come from each model's `config.json`.

| Model | Full-attn layers × KV heads × head_dim | KV / token (BF16) | Weights on disk |
|---|---|---|---|
| Qwen3.5-9B | 8 × 4 × 256 (24 more layers are DeltaNet) | **32 KiB** | 19.3 GB BF16 |
| Qwen3.6-35B-A3B | 10 × 2 × 256 | **20 KiB** | 37.5 GB FP8 |
| Qwen3.8-27B / Qwen3.6-27B | 16 × 4 × 256 | **64 KiB** | 30.9 GB FP8 |
| Qwen3.5-122B-A10B | 12 × 2 × 256 | 24 KiB | 127 GB FP8 |
| Qwen3.5-397B-A17B | 15 × 2 × 256 | 30 KiB | 406 GB FP8 |
| gpt-oss-20b / 120b | 12 / 18 full (+ the same number of 128-token sliding-window layers) × 8 × 64 | 24 / 36 KiB | ~13 / ~65 GB MXFP4 |
| Qwen3-8B (2025, for comparison) | 36 × 8 × 128 | 144 KiB | 16.4 GB BF16 |

The DeltaNet layers add a **fixed** recurrent state per sequence, tens of MB regardless of
length. So Qwen3.5+ fits 128k–262k contexts on a single GPU where Qwen3 dense models could not.
FP8 KV (`--kv-cache-dtype fp8`) halves these figures.

### 4.3 Recommended model per hardware tier

These are estimates, assuming vLLM `--gpu-memory-utilization` as shown, ~1.5 GB per engine of
CUDA graphs and activations, and BF16 KV.

| Tier | Chat (engine) | Embedding | Reranker | VRAM / RAM math |
|---|---|---|---|---|
| **CPU only** (≥ 32 GB RAM, 16+ cores; 64 GB comfortable) | **Qwen3.6-35B-A3B Q4_K_M** GGUF (llama.cpp). 3B active, so token rate is close to a dense 3–4B model. Smaller: Qwen3.5-9B Q4_K_M (5.7 GB) or gpt-oss-20b MXFP4 (12.1 GB). | Qwen3-Embedding-0.6B (TEI `cpu-1.9.4`) | bge-reranker-v2-m3 (TEI `cpu-1.9.4`; Qwen3-Reranker is vLLM-only) | 20.4 GB weights + 20 KiB × 32k ctx ≈ 0.7 GB KV + DeltaNet state ≈ **22 GB**. Embedder ≈ 1.5 GB, reranker ≈ 2.5 GB. Total ≈ 26 GB RAM. Expect single-digit to low-double-digit tok/s per stream; benchmark. |
| **1 × 24 GB** (L4, A10, RTX 4090/3090) | **Qwen3.5-9B**, online FP8 (`--quantization fp8_per_tensor --language-model-only`), 128k ctx. Alt: **gpt-oss-20b** (≈ 13 GB). | Qwen3-Embedding-0.6B (TEI GPU) | Qwen3-Reranker-0.6B (vLLM, `--gpu-memory-utilization 0.10`) | Chat at util 0.70 = 16.8 GB. Weights ≈ 7.2 GB FP8 linears + 4.1 GB BF16 embeddings/lm_head ≈ 11.3 GB, leaving ≈ 4 GB KV ≈ **120k tokens** total. Reranker 2.4 GB; TEI ≈ 1.5–2 GB. ≈ 21 GB of 24. |
| **1 × 48 GB** (L40S, RTX 6000 Ada, A6000) | **Qwen3.8-27B-FP8** at 128k ctx with `--kv-cache-dtype fp8`. Throughput alternative: **Qwen3.6-35B-A3B-FP8** (3B active, 20 KiB/token). | Qwen3-Embedding-0.6B | Qwen3-Reranker-0.6B | 27B: util 0.80 = 38.4 GB − 30.9 GB weights − 1.5 GB ≈ 6 GB KV ≈ 100k tokens BF16, **≈ 200k FP8**. The vLLM recipe lists FP8 on "single 40 GB GPU (H100/H200/L40S)". 35B-A3B: 37.5 GB weights leave ≈ 3 GB KV ≈ 150k tokens. |
| **1 × 80 GB** (H100, A100-80G, H200) | **Qwen3.8-27B-FP8** at the full 262k context. Alternatives: Qwen3.6-35B-A3B-FP8 (throughput) or gpt-oss-120b (≈ 65 GB, all-reasoning). | **Qwen3-Embedding-4B** (2560-d) or keep 0.6B | **Qwen3-Reranker-4B** | Chat at util 0.65 = 52 GB − 30.9 − 1.5 ≈ 19.6 GB KV ≈ **320k tokens** (≈ 640k FP8). Embedder 4B ≈ 9 GB, reranker 4B at util 0.12 ≈ 9.6 GB. |
| **2 × 80 GB** | **Qwen3.5-122B-A10B-FP8**, TP2 | Qwen3-Embedding-8B | Qwen3-Reranker-8B | 160 GB × 0.90 = 144 − 127 − 3 ≈ 14 GB KV ≈ 600k tokens. Put the embedder/reranker on a third GPU or reduce util. |
| **4–8 × 80 GB** | **Qwen3.5-397B-A17B-FP8**, TP8 (406 GB), or **DeepSeek-V4-Flash(-0731)** (MIT, ≈ 160 GB). The recipe reports runs at 4–8 × H100. Keep a 1-GPU Qwen3.8-27B-FP8 pool as the cheap/fast tier. | Qwen3-Embedding-8B | Qwen3-Reranker-8B | 397B: 640 × 0.9 = 576 − 406 − 8 ≈ 160 GB KV ≈ 5M tokens. |

GPU-sharing note: vLLM checks at start-up that the requested `--gpu-memory-utilization × total`
is free. Several vLLM processes plus TEI on one GPU work only if the fractions add up to under
~0.95 of the card.

---

## 5. Serving engines

| Engine | Models per endpoint | OpenAI surface | Tenant prefix-cache isolation | Fit for Caliban |
|---|---|---|---|---|
| **vLLM** (`vllm/vllm-openai:v0.30.0`) | **one base model per server**. Multi-LoRA via `--enable-lora --max-loras --max-lora-rank`; the adapter is chosen by `model`. | `/v1/chat/completions`, `/v1/completions`, `/v1/embeddings`; with `--runner pooling`: `/rerank`, `/v1/rerank`, `/v2/rerank`, `/score` | **`cache_salt`** request field ([design doc](https://github.com/vllm-project/vllm/blob/v0.30.0/docs/design/prefix_caching.md#cache-isolation-for-security)). The salt is injected into the first block hash, so only same-salt requests share KV blocks. Validation ([source](https://github.com/vllm-project/vllm/blob/v0.30.0/vllm/entrypoints/generate/base/protocol.py)): non-empty, **≤ 128 chars, no `@ / \` NUL**. Pooling requests accept it too. | Default engine: one pool per model. |
| **SGLang** (`lmsysorg/sglang:v0.5.21`) | one model per server | OpenAI chat, embeddings | **`cache_salt`** (and `extra_key`) on chat/completions requests. It namespaces the radix tree, storage and KV events (`RadixKey.cache_salt`, [radix_cache.py](https://github.com/sgl-project/sglang/blob/v0.5.21/python/sglang/srt/mem_cache/radix_cache.py)). No server flag needed. | Agent/RAG pools with heavy prefix reuse. |
| **llama.cpp** `llama-server` (`ghcr.io/ggml-org/llama.cpp:server-b11347`) | one model, or router mode (`--models-dir`, `--models-preset`, `--models-max`, default 4) | chat, completions, `/v1/embeddings` (`--embedding --pooling last`), rerank (`--reranking`) | **none**. Per-slot prompt cache (`--cache-prompt`, `--cache-reuse`). Isolate by a dedicated server per tenant, or `--no-cache-prompt`. | CPU, edge, dev. `--offline` stops network access; `--no-webui` disables the UI. |
| **Ollama** (`ollama/ollama:0.35.0`) | **many models per endpoint**, loaded on demand (`OLLAMA_MAX_LOADED_MODELS`, default 3 × GPUs; `OLLAMA_KEEP_ALIVE`; `OLLAMA_NUM_PARALLEL`, default 1) | `/v1/chat/completions` (no `tool_choice`, no `logprobs`), `/v1/embeddings`, `/v1/models`. **No rerank.** | none | Small sites and evaluation. Set `OLLAMA_NO_CLOUD=1`. Offline import via a `Modelfile` `FROM /path/model.gguf` or a safetensors dir, or ship a pre-pulled `OLLAMA_MODELS` dir. |
| **TEI** (`ghcr.io/huggingface/text-embeddings-inference:cuda-1.9.4` / `cpu-1.9.4`; per-arch tags `89-1.9.4`, `hopper-1.9.4`, …) | one model | `/v1/embeddings` (OpenAI), `/embed`, `/rerank` (no `/v1/rerank`), `/predict` | n/a (no prefix cache) | Default for embeddings (Qwen3, ModernBERT, XLM-R) and XLM-R/GTE/ModernBERT rerankers, on GPU **and CPU**. Cannot serve Qwen3-Reranker. |
| Infinity (`michaelfeil/infinity`) | several | OpenAI embeddings + rerank | n/a | Last release 0.0.77 (2025-08). Treat as unmaintained. |
| TGI | — | — | — | **Archived 2026-03-21.** Do not use. |

**vLLM recipes Caliban relies on:**

```bash
# Qwen3.5 / 3.6 / 3.8 chat
vllm serve Qwen/Qwen3.8-27B-FP8 --max-model-len 131072 --kv-cache-dtype fp8 \
  --reasoning-parser qwen3 --enable-auto-tool-choice --tool-call-parser qwen3_coder
# text-only on a small GPU: add --language-model-only (skips the vision encoder)

# Qwen3-Embedding (pooling)
vllm serve Qwen/Qwen3-Embedding-0.6B --runner pooling

# Qwen3-Reranker (original checkpoint needs the override; from the vLLM scoring docs)
vllm serve Qwen/Qwen3-Reranker-0.6B --runner pooling \
  --hf-overrides '{"architectures":["Qwen3ForSequenceClassification"],"classifier_from_token":["no","yes"],"is_original_qwen3_reranker":true}'

# gpt-oss
vllm serve openai/gpt-oss-20b --tool-call-parser openai --enable-auto-tool-choice
```

**Prefix-cache isolation, concretely.** On vLLM and SGLang pools shared by several tenants,
Caliban sets `cache_salt` on every request. The value should be
`base64url(HMAC-SHA256(deployment_secret, tenant_id))`, 43 characters. Base64url avoids the
`/` that vLLM rejects. It must not be the bare tenant ID, because the salt has to be
unpredictable or a neighbour could reproduce it. Tenants that opt into sharing get a common
salt. Engines without salt support (llama.cpp, Ollama, TEI) should be dedicated per trust
group, or run with prompt caching off. This implements the "never share prefix cache across
tenants unless opted in" rule from note 07.

**Air-gap hygiene per engine.**
- **vLLM:** `HF_HUB_OFFLINE=1`, `VLLM_NO_USAGE_STATS=1`, `DO_NOT_TRACK=1`, and
  `TIKTOKEN_ENCODINGS_BASE` for gpt-oss.
- **TEI:** load the model from a local path (`--model-id /models/...`).
- **llama.cpp:** `--offline`.
- **Ollama:** `OLLAMA_NO_CLOUD=1`, plus a pre-populated `OLLAMA_MODELS` directory.

---

## 6. Recommended default bundle

All are Apache-2.0 or MIT, pinned by commit in `deploy/airgap/models.lock.yaml`.

| Tier | Chat | Embedding | Reranker | Bundled by default? |
|---|---|---|---|---|
| CPU | `ggml-org/Qwen3.6-35B-A3B-GGUF` (Q4_K_M) | `Qwen/Qwen3-Embedding-0.6B` | `BAAI/bge-reranker-v2-m3` | yes |
| 1×24 GB | `Qwen/Qwen3.5-9B` (online FP8) | `Qwen/Qwen3-Embedding-0.6B` | `Qwen/Qwen3-Reranker-0.6B` | yes |
| 1×24 GB alt | `openai/gpt-oss-20b` (+ tiktoken encodings) | — | — | yes (unchanged from v0.1) |
| 1×48 GB | `Qwen/Qwen3.8-27B-FP8` (alt `Qwen/Qwen3.6-35B-A3B-FP8`) | `Qwen/Qwen3-Embedding-0.6B` | `Qwen/Qwen3-Reranker-0.6B` | optional (`bundle: false`, 31–38 GB) |
| 1×80 GB | `Qwen/Qwen3.8-27B-FP8` (alt `openai/gpt-oss-120b`) | `Qwen/Qwen3-Embedding-4B` | `Qwen/Qwen3-Reranker-4B` | optional |
| 2–8×80 GB | `Qwen/Qwen3.5-122B-A10B-FP8` / `Qwen/Qwen3.5-397B-A17B-FP8` / `deepseek-ai/DeepSeek-V4-Flash-0731` | `Qwen/Qwen3-Embedding-8B` | `Qwen/Qwen3-Reranker-8B` | optional |

**Intent mapping used in `core/config/open-models.example.toml`.**
- `chat`, `summarize` → fast tier with thinking off.
- `code`, `analytics` → large tier with thinking on.
- `extraction` → small tier, thinking off, tools on.
- `default` → large, then small.
- gpt-oss is kept as a cross-family fallback, so one bad checkpoint does not take down every
  route.

---

## 7. Open questions

1. **Reasoning field normalisation.** vLLM changed `reasoning_content` → `reasoning`. Which name
   does the Caliban API emit? Recommendation: emit both for one release, like OpenRouter does.
   Then settle on one in `core/api/openapi.yaml`.
2. **Per-request `reasoning_effort` vs `enable_thinking` for Qwen3.8.** The config allows one
   `reasoning_control`. Qwen3.8 accepts both. Should `reasoning_effort` become a second,
   optional control?
3. **Ollama tag drift.** Ollama library tags (`qwen3.6:35b`) are re-quantized by Ollama and
   cannot be pinned by HF revision. For sovereign sites, prefer importing the pinned GGUF via a
   `Modelfile`.
4. **Qwen3.8 licences.** Get a legal opinion on whether an on-prem gateway serving one
   enterprise's own employees is "internal Use" under the Qwen Community License 1.0. If it is,
   Qwen3.8-Flash-Next could become an opt-in for 2×80 GB sites.
5. **Benchmarks.** Every tok/s and concurrency figure here is arithmetic, not measurement. Run
   `vllm bench serve` per tier and record the results in this note.

### Source index

Models: [Qwen org](https://huggingface.co/Qwen) · [Qwen3.6-35B-A3B card](https://huggingface.co/Qwen/Qwen3.6-35B-A3B) · [Qwen3.8-27B card](https://huggingface.co/Qwen/Qwen3.8-27B) · [Qwen3.8-Flash-Next LICENSE](https://huggingface.co/Qwen/Qwen3.8-Flash-Next/blob/main/LICENSE) · [Qwen3.8-2.4T-A95B LICENSE](https://huggingface.co/Qwen/Qwen3.8-2.4T-A95B/blob/main/LICENSE) · [Qwen3-8B card](https://huggingface.co/Qwen/Qwen3-8B) · [Qwen3-Embedding-0.6B card](https://huggingface.co/Qwen/Qwen3-Embedding-0.6B) · [gpt-oss-20b](https://huggingface.co/openai/gpt-oss-20b) · [Mistral-Medium-3.5 LICENSE](https://huggingface.co/mistralai/Mistral-Medium-3.5-128B/blob/main/LICENSE) · [DeepSeek-V4-Flash](https://huggingface.co/deepseek-ai/DeepSeek-V4-Flash) · [Gemma 4 licence](https://ai.google.dev/gemma/docs/gemma_4_license) · [GLM-5.3 LICENSE](https://huggingface.co/zai-org/GLM-5.3/blob/main/LICENSE) · [Granite 4.2](https://huggingface.co/ibm-granite/granite-4.2-8b) · [Nemotron 3.5 LICENSE](https://huggingface.co/nvidia/NVIDIA-Nemotron-3.5-Lightning-30B-A3B-BF16) · [MiniMax-M2.7 LICENSE](https://huggingface.co/MiniMaxAI/MiniMax-M2.7) · [Kimi-K3 LICENSE](https://huggingface.co/moonshotai/Kimi-K3) · [Llama 4 AUP](https://www.llama.com/llama4/use-policy/)

Engines: [vLLM v0.30.0 release](https://github.com/vllm-project/vllm/releases/tag/v0.30.0) · [tool calling](https://github.com/vllm-project/vllm/blob/v0.30.0/docs/features/tool_calling.md) · [reasoning outputs](https://github.com/vllm-project/vllm/blob/v0.30.0/docs/features/reasoning_outputs.md) · [prefix caching / cache_salt](https://github.com/vllm-project/vllm/blob/v0.30.0/docs/design/prefix_caching.md) · [pooling: scoring](https://github.com/vllm-project/vllm/blob/v0.30.0/docs/models/pooling_models/scoring.md) · [pooling: embed](https://github.com/vllm-project/vllm/blob/v0.30.0/docs/models/pooling_models/embed.md) · [quantization](https://github.com/vllm-project/vllm/blob/v0.30.0/docs/features/quantization/README.md) · [vLLM recipes](https://recipes.vllm.ai/) · [SGLang v0.5.21 protocol](https://github.com/sgl-project/sglang/blob/v0.5.21/python/sglang/srt/entrypoints/openai/protocol.py) · [llama.cpp server README](https://github.com/ggml-org/llama.cpp/blob/b11346/tools/server/README.md) · [Ollama OpenAI compatibility](https://github.com/ollama/ollama/blob/v0.35.0/docs/api/openai-compatibility.mdx) · [Ollama FAQ](https://github.com/ollama/ollama/blob/v0.35.0/docs/faq.mdx) · [TEI v1.9.4 README](https://github.com/huggingface/text-embeddings-inference/blob/v1.9.4/README.md) · [TGI (archived)](https://github.com/huggingface/text-generation-inference)
