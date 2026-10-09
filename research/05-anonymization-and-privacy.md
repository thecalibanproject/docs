# Anonymization and Privacy for the Caliban Gateway

The 2023–2026 literature agrees on four points. (1) **No single PII detector is enough.** Rule-based tools (Presidio) miss most context-dependent and obfuscated PII. Encoder NER models break on out-of-distribution text. LLM detectors have the best recall but are slow and unstable. Hybrid cascades do best ([Mind the Gap 2026](https://arxiv.org/abs/2609.03464), [REDACT 2026](https://arxiv.org/pdf/2606.19881), [LLM-Redactor 2026](https://arxiv.org/pdf/2604.12064)). (2) **Placeholder redaction costs answer quality, but consistent, type-preserving surrogates recover most of it.** Judges preferred unredacted answers about 75–80% of the time, while surrogates gave +13 pp BERTScore over placeholders ([SurrogateShield 2026](https://arxiv.org/abs/2606.29567)). (3) **Removing direct identifiers is not anonymization.** LLMs infer age, location and income from "anonymized" text with up to 85% top-1 accuracy ([Beyond Memorization](https://arxiv.org/abs/2310.07298)). Under GDPR, pseudonymized data is still personal data ([EDPB Guidelines 01/2025](https://www.edpb.europa.eu/system/files/2025-01/edpb_guidelines_202501_pseudonymisation_en.pdf)). (4) **Caches and embeddings are first-class leakage channels.** Embeddings can be inverted to text (92% exact recovery for 32-token inputs, [vec2text](https://arxiv.org/abs/2310.06816)), including zero-shot and across models ([ZSinvert](https://arxiv.org/abs/2504.00147), [vec2vec](https://arxiv.org/abs/2505.12540)). Adding noise does not save them ([DAEI 2026](https://arxiv.org/abs/2608.18610)). Gateways that pool upstream credentials collapse prompt-cache isolation between customers ([KeyPooling 2026](https://arxiv.org/abs/2608.17485), [CacheProbe 2026](https://arxiv.org/abs/2605.30613)).

For Caliban, the plan has five parts:
- **Detection:** a Rust-native tiered detector (patterns, then a small ONNX NER model, then an optional LLM verifier).
- **Pseudonymization:** reversible, vault-backed surrogates with per-tenant keys.
- **Routing:** privacy-aware routing. Sensitive work goes to local or sovereign models, and only external calls get strict scrubbing.
- **Cache isolation:** strict per-tenant isolation of every cache, including upstream provider caches.
- **Embeddings and logs:** embeddings are handled as personal data, and logs contain no PII by construction.

---

## Key papers

| Paper | Year | Core idea (1 sentence) | Why it matters for Caliban | Verdict |
|---|---|---|---|---|
| [Hide and Seek (HaS)](https://arxiv.org/abs/2309.03057v1) | 2023 | A small local "Hide" model anonymizes entities and a "Seek" model de-anonymizes the LLM's output. | This is the reference pattern for anonymizing before the provider and restoring afterwards; it validates the gateway-side design. | Adopt (pattern) |
| [DP-Prompt: Locally DP Document Generation via Zero-Shot Prompting](https://arxiv.org/abs/2310.16111) | 2023 | A local LLM paraphrases the document with temperature-controlled sampling to get document-level local DP. | Provides formal DP for stylometry and author de-anonymization, but paraphrasing loses utility; later work reports garbled outputs at low ε ([DP-Fusion](https://arxiv.org/pdf/2507.04531)). | Watch |
| [SanText / SanText+](https://arxiv.org/pdf/2106.01221) | 2021 | Word-level metric-LDP token substitution via embedding-similarity exponential mechanism. | Baseline DP sanitization. Word-level DP is reconstructable by LLMs ([ACM 2025 follow-up](https://dl.acm.org/doi/10.1145/3733802.3764058)). | Skip for prompts |
| [DP-GTR](https://arxiv.org/abs/2503.04990) | 2025 | Group text rewriting with local DP plus in-context learning for prompt privatization. | Represents the state of the art for DP prompt rewriting. It is still lossy and needs a local LLM call. | Watch |
| [PrivacyRestore](https://arxiv.org/abs/2406.01394) | 2024 | Removes private spans from the input and restores their semantics via activation-steering vectors on the server. | Requires control of the serving model's activations. It fits only Caliban-hosted open-weight models. | Watch |
| [Split-and-Denoise](https://arxiv.org/abs/2310.09130) | 2023 | The client computes token embeddings locally, adds LDP noise, and denoises the server's output. | Needs a split model; it does not work with closed APIs. | Skip |
| [PAPILLON (Privacy-Conscious Delegation)](https://arxiv.org/abs/2410.17127) | 2024/25 | A local model rewrites the query, uses the cloud LLM as a tool, and composes the answer locally. | Kept quality on 85.5% of queries with 7.5% leakage, so local-model rewriting is viable. It maps directly onto Caliban's router and sub-agents. | Prototype |
| [Beyond Memorization](https://arxiv.org/abs/2310.07298) | 2023 (ICLR'24) | LLMs infer personal attributes from text, up to 85% top-1 on real Reddit profiles. | Shows that removing named entities is not enough: quasi-identifiers leak. It defines the threat model for "strict" mode. | Adopt (threat model) |
| [LLMs are Advanced Anonymizers](https://arxiv.org/abs/2402.13846) | 2024 (ICLR'25) | Adversarial anonymization: an LLM attacker guides an LLM anonymizer, beating commercial anonymizers on privacy and utility. | Gives the design for an optional L2 "inference-risk" rewriter on high-sensitivity tenants. A distilled version exists: [SEAL](https://arxiv.org/pdf/2506.01420). | Prototype |
| [ConfAIde (Can LLMs Keep a Secret?)](https://arxiv.org/abs/2310.17884) | 2023 (ICLR'24) | Contextual-integrity benchmark: GPT-4 leaks private info in 39% of cases where humans would not. | When agents and nodes see data from many datasources, the model itself cannot be trusted to enforce information-flow norms. Policy must sit in the gateway. | Adopt (principle) |
| [Text Embeddings Reveal (Almost) As Much As Text (vec2text)](https://arxiv.org/abs/2310.06816) | 2023 | Iterative inversion recovers 92% of 32-token inputs exactly, including names from clinical notes. | Caliban's vector caches store embeddings, which must therefore be treated as plaintext-equivalent. | Adopt (threat model) |
| [Universal Zero-shot Embedding Inversion (ZSinvert)](https://arxiv.org/abs/2504.00147) | 2025 | Inverts any encoder with black-box query access and no per-model training. | Using an obscure embedding model gives no protection. | Adopt (threat model) |
| [Harnessing the Universal Geometry of Embeddings (vec2vec)](https://arxiv.org/abs/2505.12540) | 2025 | Translates embeddings between models without paired data, which enables attribute inference from leaked vectors. | A stolen vector DB dump is exploitable even without the encoder. | Adopt (threat model) |
| [Denoising-Aware Inversion (DAEI)](https://arxiv.org/abs/2608.18610) | 2026 | Adaptive attacks defeat Gaussian-noise-protected embeddings (+154% relative BLEU over baselines). | Do not rely on noise; use encryption plus isolation. | Adopt (threat model) |
| [Is My Data in Your Retrieval Database? (RAG MIA)](https://arxiv.org/abs/2405.20446) / [Mask-based MIA](https://arxiv.org/abs/2410.20142) / [CIG-MIA](https://arxiv.org/html/2609.14649v1) | 2024–26 | Black-box queries reveal whether a document is in a RAG index (AUC up to 0.99 gray-box for CIG-MIA). | Shared RAG and ontology indexes leak membership across users. Retrieval must be scoped per tenant and per ACL, and outputs rate-limited. | Adopt (threat model) |
| [Auditing Prompt Caching in LM APIs](https://arxiv.org/abs/2502.07776) | 2025 (ICML) | Timing audits found global cross-user prompt-cache sharing in 7 of 17 providers. | Any cache Caliban shares across tenants is a timing oracle. | Adopt |
| [KeyPooling](https://arxiv.org/abs/2608.17485) | 2026 | All 5 open-source gateways tested exposed cross-customer cache reads because they pooled upstream credentials. | **Directly about Caliban's product category.** Bind upstream cache namespaces to authenticated tenant identity; the cost is about 1.7–2.5%. | Adopt |
| [CacheProbe](https://arxiv.org/abs/2605.30613) | 2026 | Audits whether gateway shared org credentials create cross-account cache sharing (OpenRouter). | Same lesson; Caliban will be audited this way. | Adopt |
| [Shadow in the Cache (KV-Cloak)](https://arxiv.org/abs/2508.09442) / [InputSnatch](https://arxiv.org/html/2411.18191v1) / [Selective KV sharing](https://arxiv.org/abs/2508.08438) | 2024–26 | KV caches can be inverted, and prefix-sharing timing leaks inputs. | Relevant when Caliban self-hosts models (sovereign), where it controls prefix caching. | Adopt (isolation) / Watch (KV-Cloak) |
| [LLM-Redactor](https://arxiv.org/abs/2604.12064) | 2026 | Compares 8 techniques (local, redaction, rephrase, TEE, split, FHE, MPC, DP). Local routing plus redaction plus rephrasing gave 0.6% PII leak. | Empirical support for combining routing with redaction. Redaction alone costs quality (judges preferred baseline about 75–80% of the time). | Adopt (architecture) |
| [SurrogateShield](https://arxiv.org/abs/2606.29567) | 2026 | Type-consistent surrogate values (not placeholders) with local encrypted mapping and restoration. | +13.26 pp BERTScore over placeholders, and no originals were recovered by an LLM adversary. Supports the surrogate design. | Adopt |
| [Mind the Gap: Robustness Risks in PII Detection](https://arxiv.org/abs/2609.03464) | 2026 | spaCy, Presidio and Qwen2.5-3B all degrade on out-of-distribution inputs, with complementary failure modes. | Supports a hybrid cascade and an out-of-distribution regression suite. | Adopt |
| [REDACT benchmark](https://arxiv.org/pdf/2606.19881) / [CAPID](https://arxiv.org/pdf/2602.10074) | 2026 | Presidio recall is 0.02 on partial mentions and 0.07 on obfuscated mentions; high-tier recall is 0.07 vs 0.74–0.77 for frontier LLMs. | Presidio must not be the main detector. | Adopt (as eval) |
| [GLiNER2-PII](https://arxiv.org/abs/2605.09973) | 2026 | 0.3B zero-shot-label model with 42 PII types; best span F1 on SPY among 5 systems, including OpenAI Privacy Filter. | Strong candidate for the in-process L1 model (ONNX via `ort`). It is recall-leaning. | Adopt |
| [OpenAI Privacy Filter eval across 32 benchmarks](https://arxiv.org/abs/2608.02616) | 2026 | OPF (1.5B MoE, 50M active) beats Presidio zero-shot, but collapses on non-Latin scripts (F1 0.03–0.04). Fine-tuned XLM-R surpasses it with about 500 examples. | Use OPF as an English L1 candidate; non-English tenants need another model. Fine-tuning on tenant data pays off. | Prototype |
| [Institution-Specific LLM Prompting Recovers Missed PHI](https://arxiv.org/abs/2608.17051) | 2026 | Detectors and even gold standards miss "institutional" PHI (building names, internal codes); tenant-specific LLM prompting recovers 79% of it. | Supports tenant-supplied dictionaries from the ontology layer and per-tenant detector prompts. | Adopt |
| [Confidential Computing on Hopper GPUs](https://arxiv.org/abs/2409.03992) / [CPU+GPU TEE LLM cost](https://arxiv.org/abs/2509.18886) | 2024–25 | H100 CC-mode overhead is below 7% for typical LLM queries, mostly from PCIe transfer. | TEE-hosted inference is affordable for the sovereign tier. | Prototype |
| [Apple Private Cloud Compute](https://security.apple.com/blog/private-cloud-compute/) | 2024 | Five requirements: stateless compute, enforceable guarantees, no privileged runtime access, non-targetability, verifiable transparency. | Use these as the checklist for the "sovereign router". | Adopt (requirements) |

## Open-source implementations

| Project | Language | Use | Notes (Rust-usable?) |
|---|---|---|---|
| [microsoft/presidio](https://github.com/microsoft/presidio) | Python | Regex, NER and context recognizers plus anonymizer operators | Not in-process for Rust; useful as an eval baseline and as a source of recognizer patterns to port. Low recall on context-dependent PII (see REDACT). |
| [GLiNER2-PII](https://arxiv.org/abs/2605.09973) (HF release) | PyTorch → ONNX | 42-type zero-shot-label PII NER, 0.3B | Yes, via ONNX export and `ort`. Label set is configurable per tenant at inference time. |
| [nvidia/gliner-pii](https://huggingface.co/nvidia/gliner-pii) | PyTorch | 55+ PII/PHI categories, GLiNER large-v2.1 base | Yes, via ONNX export. NVIDIA Open Model License (commercial OK). F1 0.64–0.87 depending on dataset. |
| OpenAI Privacy Filter ([announcement coverage](https://www.helpnetsecurity.com/2026/04/23/openai-privacy-filter-personally-identifiable-information/), [details](https://grepture.com/blog/openai-privacy-filter-pii-redaction)) | PyTorch | 8 fixed categories, including secrets and API keys | Apache-2.0. MoE with 50M active parameters, so it is cheap on CPU. ONNX/`ort` export not confirmed; test before committing. English-first. |
| [gline-rs](https://docs.rs/crate/gline-rs/0.9.2) / [gliner2-rs](https://github.com/codesoda/gliner2-rs) / [fast_gliner](https://github.com/talmago/fast_gliner) | Rust | GLiNER/GLiNER2 inference on ONNX Runtime | **Yes, native.** Pure Rust on `ort` and `tokenizers`; reported about 4× faster than PyTorch on CPU. |
| [pykeio/ort](https://github.com/pykeio/ort) | Rust | ONNX Runtime bindings (CPU/CUDA/TensorRT/CoreML) | **Yes.** The standard choice; it is used by HF Text Embeddings Inference. Also covers embedding models and the intent classifier. |
| [BurntSushi/aho-corasick](https://github.com/BurntSushi/aho-corasick) + `regex` | Rust | Multi-pattern dictionary matching (tenant entity lists, surrogate re-hydration) | **Yes.** Supports streaming search, which suits re-hydrating streamed responses. MIT/Unlicense. |
| [str4d/fpe](https://github.com/str4d/fpe) / [fpr-ff1](https://docs.rs/fpr-ff1/latest/fpr_ff1/) | Rust | NIST FF1 format-preserving encryption | **Yes.** Use FF1 only: FF3/FF3-1 are dropped in the SP 800-38G Rev.1 2nd draft. |
| [protectai/llm-guard](https://github.com/protectai/llm-guard) | Python | Anonymize / Deanonymize scanners with a vault | Reference design only. **Archived July 2026**, so do not depend on it. |
| [IronCore Cloaked AI](https://ironcorelabs.com/docs/cloaked-ai/how-it-works/) | Rust core, multi-lang | Approximate distance-comparison-preserving encryption of embeddings | Encrypted vectors still support kNN in any vector DB. Commercial/AGPL-style licensing; check terms. Leaks relative distances by design. |
| [vec2text](https://github.com/vec2text/vec2text) | Python | Embedding inversion | Use in red-team CI against Caliban's own embedding model and cache dumps. |
| [confaide](https://github.com/skywalker023/confaide) | Python | Contextual-integrity benchmark | Eval for agent/node information-flow policy. |
| [SanText](https://github.com/xiangyue9607/SanText) | Python | Word-level DP sanitization baseline | Research baseline only. |

## Risks & failure modes

| Risk | Evidence | Mitigation in Caliban |
|---|---|---|
| **Incomplete redaction (false negatives)** | Presidio recall is 0.02–0.07 on partial and obfuscated mentions ([REDACT](https://arxiv.org/pdf/2606.19881)). Every detector family degrades out of distribution ([Mind the Gap](https://arxiv.org/abs/2609.03464)). OPF collapses on non-Latin scripts ([2608.02616](https://arxiv.org/abs/2608.02616)). Institutional PHI is missed even by gold labels ([2608.17051](https://arxiv.org/abs/2608.17051)). | Recall-biased union of tiers. Tenant dictionaries from the ontology. Per-language model selection. Continuous out-of-distribution eval. Fail closed in strict mode. |
| **Re-identification by inference (quasi-identifiers)** | LLMs reach 85% top-1 attribute inference ([Beyond Memorization](https://arxiv.org/abs/2310.07298)). Anonymizers must be adversarially evaluated ([LLMs are Advanced Anonymizers](https://arxiv.org/abs/2402.13846)). | Strict tier: generalize quasi-identifiers (age → band, city → region, date → shifted), plus an optional adversarial LLM check. Route to a local model when residual risk is high. |
| **Contextual-integrity violations by agents** | GPT-4 leaks in 39% of cases where humans would not ([ConfAIde](https://arxiv.org/abs/2310.17884)). | Gateway-enforced flow policy: datasource × node × destination model. Never rely on prompt instructions alone. |
| **Embedding inversion of cached vectors** | 92% exact recovery ([vec2text](https://arxiv.org/abs/2310.06816)), zero-shot ([ZSinvert](https://arxiv.org/abs/2504.00147)), cross-model ([vec2vec](https://arxiv.org/abs/2505.12540)), noise defeated ([DAEI](https://arxiv.org/abs/2608.18610)). | Embed only pseudonymized text. Per-tenant encryption at rest. Optional DCPE ([Cloaked AI](https://ironcorelabs.com/docs/cloaked-ai/how-it-works/)). Classify vector stores as personal-data systems. |
| **RAG membership inference** | AUC up to 0.99 ([CIG-MIA](https://arxiv.org/html/2609.14649v1)); masked-text attacks ([MBA](https://arxiv.org/abs/2410.20142)); stealthy variants ([Riddle Me This](https://arxiv.org/abs/2502.00306)). | Per-tenant and per-ACL retrieval scopes. Anomaly detection on probing patterns. Optionally return no verbatim chunks to untrusted users. |
| **Cross-tenant cache leakage (semantic cache, upstream prompt cache, KV cache)** | 7 of 17 providers had global cache sharing ([Gu et al.](https://arxiv.org/abs/2502.07776)). All 5 gateways tested leaked across customers ([KeyPooling](https://arxiv.org/abs/2608.17485)). Gateway-level credential pooling has been audited ([CacheProbe](https://arxiv.org/abs/2605.30613)). KV inversion ([Shadow in the Cache](https://arxiv.org/abs/2508.09442)). Timing input theft ([InputSnatch](https://arxiv.org/html/2411.18191v1)). | Tenant-bound upstream credentials or cache namespaces. Semantic-cache namespace = tenant (and user for personal scopes). No cross-tenant prefix sharing on self-hosted models. |
| **Logs, traces and analytics leaking PII** | Common failure in gateways and observability stacks. Pseudonymized data is still personal data ([EDPB 01/2025](https://www.edpb.europa.eu/system/files/2025-01/edpb_guidelines_202501_pseudonymisation_en.pdf)). | Logs are written only after scrubbing. Raw-prompt capture is opt-in, encrypted with the tenant key, and has a short TTL. Values are stored as keyed HMACs, never plaintext. |
| **Vault compromise means full re-identification** | The surrogate mapping is the "additional information" under GDPR Art. 4(5) ([EDPB 01/2025](https://www.edpb.europa.eu/system/files/2025-01/edpb_guidelines_202501_pseudonymisation_en.pdf)). | Envelope encryption per tenant. Vault kept separate from logs and caches. Crypto-shredding for erasure. Customer-held keys in the sovereign tier. |
| **Utility loss and re-hydration errors** | Placeholders reduce quality ([LLM-Redactor](https://arxiv.org/abs/2604.12064)). Surrogates recover it ([SurrogateShield](https://arxiv.org/abs/2606.29567)). Models paraphrase or inflect surrogates. | Type-consistent surrogates. FPE for structured IDs. Fuzzy, coreference-aware re-hydration. Eval suite on answer quality. |
| **Word-level DP gives false comfort** | LLMs reconstruct word-level DP-sanitized text ([ACM 2025](https://dl.acm.org/doi/10.1145/3733802.3764058)). | Do not market DP sanitization as anonymization. |

## Recommended design for Caliban

### 1. Tiered detection pipeline (Rust, in the request path)

| Tier | What | Implementation | Latency target* |
|---|---|---|---|
| **L0 deterministic** | Regex plus validators (Luhn, IBAN mod-97, SSN/NIN rules, E.164 phones, emails, IPs, JWT/API-key formats with an entropy check). **Tenant dictionaries** compiled from the ontology/datasource layer (customer names, account IDs, employee lists, internal codes). | `regex` + `aho-corasick` (leftmost-longest), rebuilt on ontology change | < 1 ms |
| **L1 small NER in-process** | GLiNER2-PII (multilingual, configurable labels) as the default. OpenAI Privacy Filter as an English and secrets-focused option. Fine-tune XLM-R per vertical when the data exists (about 500 examples beat OPF zero-shot). | `ort` (INT8, CPU; GPU optional), `gline-rs`-style decoding, sliding 512-token windows | 5–30 ms per 1k tokens on CPU (to benchmark) |
| **L2 LLM verifier (optional)** | A local small LLM, never an external one, that (a) catches context-dependent and institutional PII L0/L1 missed, and (b) in strict mode scores quasi-identifier inference risk (adversarial-anonymization style) and proposes generalizations. | Self-hosted model behind Caliban's own router. Blocking in `strict`; async and sampled in `standard` to measure drift. | 100–500 ms |

*Latency targets are engineering estimates, not measured results.

- **Merge rule:** take the union of spans and resolve overlaps by preferring the longest span and the higher-risk type. The bias is toward recall, because a false negative costs more than a false positive.
- **Fail closed:** if L1 or L2 times out in `strict` mode, block the request or route it to a local model; never forward it unscrubbed.
- **Secrets:** API keys and passwords are always blocked or removed. They are never pseudonymized or sent out.

### 2. Reversible pseudonymization (vault)

- **Surrogate types:**
  - **Names, orgs, locations:** realistic surrogates drawn from curated per-locale pools. Keep gender and culture plausible where that is detectable.
  - **Structured IDs (accounts, phones, card numbers):** **FF1 FPE** (`fpe` crate) under a per-tenant tweak, so format and checksum reasoning still work. Recompute the Luhn digit for surrogate card numbers.
  - **Dates:** a consistent per-session date shift, which preserves intervals (HIPAA-style).
  - **Emails and URLs:** structured surrogates such as `user7@example.net`.
  - **Fallback:** typed placeholders like `⟦PERSON_3⟧`. Use them where surrogates are risky, for example when the destination model is asked to output exact strings.
- **Consistency:** the same normalized entity maps to the same surrogate within a **scope**: request, conversation/session (default), or tenant (needed for semantic-cache hits and agent memory). Derive tenant-scope surrogates as `HMAC(tenant_key, type‖normalized_value)` → pool index or FPE input, so no lookup is needed to stay consistent. The vault still stores the reverse map.
- **Coreference:** cluster "John Smith", "John", "Mr. Smith" and "J. Smith" before substitution, so each part maps to the matching surrogate part. A first name maps to the surrogate's first name.
- **Vault storage:** encrypted reverse map `(tenant, scope_id, surrogate) → original`. Use envelope encryption: a per-tenant KEK in KMS/HSM (customer-held in the sovereign tier) and per-scope DEKs. Keep it in a store separate from logs and caches, with a TTL equal to the scope lifetime. Deleting a DEK or KEK crypto-shreds the data, which serves GDPR erasure.
- **Collision guard:** reject a surrogate that already appears in the original prompt or in tenant dictionaries, so re-hydration cannot rewrite genuine text.

### 3. Re-hydration of responses, including streaming

- Build an Aho-Corasick automaton over the active surrogates of the scope, including part-forms (first name, last name, possessives, upper and lower case).
- **Streaming:** keep a holdback buffer equal to the longest surrogate length minus 1. Emit only the prefix that cannot be the start of a match, and flush on stream end. This adds a few tokens of delay, not a full buffer. Handle SSE chunk and tokenizer splits by matching on the decoded text buffer, not on tokens.
- **Structured outputs and tool calls:** re-hydrate JSON string values after parsing. For **tool calls to customer datasources**, re-hydrate the arguments inside the Caliban trust boundary before executing them. Re-pseudonymize the tool results before they go back to an external model.
- **Fuzzy fallback:** if the model mangles a surrogate (for example "Mr. Lopez's" or a lowercase form), a normalized matcher catches it. Unmatched placeholder tokens are flagged in telemetry as a re-hydration miss, which is a quality metric.

### 4. Policy engine (tenant × datasource × destination model)

| Destination trust tier | Example | Default treatment |
|---|---|---|
| **T0 sovereign/local** | Caliban-hosted open-weight model on customer premises or VPC | No redaction needed. Data does not leave the boundary. Logs are still scrubbed. Caches are still isolated. |
| **T1 attested TEE** | Model in H100 CC mode or a CPU TEE with verified attestation, under a PCC-like design | Same as T0 if attestation verifies against a pinned measurement; otherwise demote to T3. |
| **T2 contracted external** | Provider under DPA/BAA with zero data retention | `standard`: pseudonymize direct identifiers (Safe Harbor-like 18-identifier coverage), drop secrets. |
| **T3 public/unknown** | Any other route | `strict`: T2 treatment plus quasi-identifier generalization and an L2 verifier; fail closed. |

- **Privacy-aware routing:** when the intent classifier and detector flag high sensitivity and a capable T0 model exists, the router prefers T0 (PAPILLON-style delegation). Otherwise it pseudonymizes and sends to the best external model. Tie this to Caliban's existing routing cost function: add a privacy penalty term.
- **Datasource labels:** each ontology field is labeled (PII/PHI/PCI/confidential/public), and labels propagate into RAG chunks. Example rule: `datasource=ehr → never T3`.
- **Agents/nodes:** each node declares which datasource labels it may receive and which destinations it may call. The gateway enforces this as a contextual-integrity rule; it is not left to the model.
- **Policies are versioned**, and the policy version is stamped on every audit record.

### 5. Protecting the vector cache and RAG index

1. **Never embed raw PII.** Embed the pseudonymized text using **tenant-scope** surrogates, so semantically equal queries still hit the cache. Store cached responses in pseudonymized form and re-hydrate them on a hit within the same scope only.
2. **Per-tenant namespaces as physically separate collections or indexes**, not a shared index with a metadata filter. User-personal data gets a per-user sub-namespace.
3. **Encryption at rest** with the per-tenant DEK, covering the vectors as well as the payloads. For hosted or shared vector DBs, consider DCPE ([Cloaked AI](https://ironcorelabs.com/docs/cloaked-ai/how-it-works/)), noting that it leaks relative distances.
4. **Treat embeddings as personal data** in the DPIA and retention policy, with TTLs, erasure support and crypto-shredding.
5. **Upstream prompt cache:** use per-tenant provider credentials or provider-supported cache-isolation keys derived from the authenticated tenant ID. Never pool keys across tenants ([KeyPooling](https://arxiv.org/abs/2608.17485)). Shared public prefixes (system prompts) may sit before the boundary.
6. **Self-hosted KV/prefix cache:** no cross-tenant prefix sharing; selective sharing only for public prefixes.
7. **Red-team CI:** run vec2text or ZSinvert against Caliban's embedding model and sample cache dumps every release, and run MIA probes against RAG.

### 6. Audit log design

- **Content:** `request_id, tenant, user/principal, node, datasource labels, destination model and tier, policy version, detector versions, entity-type counts, span HMACs (keyed per tenant), decision (forward/pseudonymize/block/reroute), re-hydration miss count, latency per tier`. No raw values.
- **Integrity:** append-only and hash-chained (tamper-evident), exportable to the customer SIEM.
- **Separate re-identification log:** a separate, access-restricted log records every vault read beyond automatic re-hydration (for example support or debugging), with a justification field.
- **Debug capture:** raw prompt capture is off by default, per-tenant opt-in, encrypted with the tenant key, and kept for at most N days. Provide an "explain redaction" view built from spans and HMACs.

### 7. What "sovereign" deployment must guarantee (mapped to PCC)

| Guarantee | Concrete requirement |
|---|---|
| Stateless processing | Prompts and responses are not persisted beyond the configured scope TTL; vault and caches are inside the customer boundary. |
| Enforceable, not policy | No egress except allow-listed model endpoints, enforced by network policy. All detector and embedding models run in-process; no external calls for PII detection. |
| No privileged runtime access | No vendor remote shell. Support works through signed diagnostic bundles that are already scrubbed. |
| Customer-held keys | BYOK/HYOK: tenant KEK in the customer's HSM/KMS. Revoking it crypto-shreds the vault, caches and debug captures. |
| Non-targetability and isolation | Per-tenant namespaces for caches, vault and indexes; no shared upstream credentials. |
| Verifiable transparency | Signed, reproducible builds with an SBOM. Published detector model hashes. Optional TEE attestation of the Caliban data plane and model servers. |
| Air-gap capable | Offline model and policy updates through signed bundles; no telemetry by default. |

### 8. Regulatory positioning (be precise in marketing)

- **GDPR:** Caliban's output is **pseudonymization, not anonymization**. It is still personal data and still in GDPR scope, but it counts as a recognized safeguard ([EDPB Guidelines 01/2025](https://www.edpb.europa.eu/system/files/2025-01/edpb_guidelines_202501_pseudonymisation_en.pdf)). On whether models themselves are anonymous, see [EDPB Opinion 28/2024](https://www.edpb.europa.eu/system/files/2024-12/edpb_opinion_202428_ai-models_en.pdf).
- **HIPAA:** the `standard` tier should cover all 18 Safe Harbor identifiers ([HHS guidance](https://www.hhs.gov/hipaa/for-professionals/special-topics/de-identification/index.html)). Do not claim "de-identified" unless Expert Determination has been done for the tenant's workload. Offer a BAA plus the T0/T1 tiers for PHI.
- **EU AI Act:** Art. 10(5) requires pseudonymization and strict access control when special-category data is processed for bias work ([Art. 10](https://artificialintelligenceact.eu/article/10/)). Caliban's audit log plus pseudonymization supports customers' high-risk-system obligations.

## Open questions

1. **Surrogates vs typed placeholders per task.** Realistic surrogates help reasoning, but may mislead when the model is asked to "verify" facts or generate legal documents. Run per-intent A/B tests through the intent classifier.
2. **Tenant-scope deterministic surrogates enable semantic-cache hits but create linkability across sessions.** Is that acceptable per tenant, or should it be opt-in?
3. **Which L1 model wins on Caliban's real traffic mix** (multilingual, code, tables): GLiNER2-PII, nvidia/gliner-pii, OPF, or a fine-tuned XLM-R? Can OPF's MoE be exported cleanly to ONNX for `ort`?
4. **How to measure residual inference risk cheaply** enough to run on every strict request, rather than only in sampled audits.
5. **Re-hydration correctness under agentic loops:** surrogates flow into tool calls, memory and sub-agent prompts. Where exactly does the boundary sit, and does any surrogate ever reach a customer system unresolved?
6. **DCPE vs plain encrypted-at-rest plus physical isolation:** is the distance leakage of DCPE acceptable for regulated tenants?
7. **Upstream provider cache isolation:** does each provider expose a per-end-customer cache key, or does Caliban need one provider account per tenant? This has cost and quota implications.
8. **Images, audio and documents (OCR'd PDFs):** the same pipeline is needed for multimodal inputs. This is out of scope here and needs its own survey.
9. **Legal:** can the T1 TEE tier be contractually treated as "data does not leave the customer's control" for GDPR transfer and Schrems II purposes?
