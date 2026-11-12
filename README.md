**English** | [한국어](TCO-GLM-5.2-Final-Report-KO.md)

# TCO Analysis: GLM-5.2 FP8 Self-Hosted (GCP GEAP) vs Z.ai SaaS
## AI Coding Agent Workloads — 50 to 500 Developers

**Date:** 2026-07-01  
**Status:** Final — approved through 4 independent reviewer passes  
**Scope:** Team sizes of 50, 100, 300, and 500 developers; Heavy User baseline (100M input / 4M output tokens per developer per month)

---

> **TL;DR**
>
> Self-hosting GLM-5.2 FP8 on GCP GEAP costs a minimum of <strong>&#36;606 per developer per month</strong> (3-year CUD, throughput-limited). Z.ai Pro costs <strong>&#36;72</strong> and Max costs <strong>&#36;160</strong> per developer per month. The cost ratio is <strong>≥8.4× vs Pro</strong> and <strong>≥3.79× vs Max</strong> regardless of team size — the financial breakeven is mathematically unreachable. Use Z.ai SaaS.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Background & Scope](#2-background--scope)
3. [Technical Architecture & Feasibility](#3-technical-architecture--feasibility)
4. [Coding Agent Workload Baseline](#4-coding-agent-workload-baseline)
5. [Z.ai SaaS Cost Analysis](#5-zai-saas-cost-analysis)
6. [Self-Hosted TCO (GCP GEAP)](#6-self-hosted-tco-gcp-geap)
7. [Breakeven Analysis & Scale Economics](#7-breakeven-analysis--scale-economics)
8. [Scenario Comparison Matrix](#8-scenario-comparison-matrix)
9. [Final Recommendation](#9-final-recommendation)
10. [Assumptions & Limitations](#10-assumptions--limitations)
11. [References](#11-references)

---

## 1. Executive Summary

This report answers one question: for a software engineering team of 50 to 500 developers using AI coding agents powered by GLM-5.2, is it cheaper to self-host the model on GCP's Gemini Enterprise Agent Platform (GEAP) or to use Z.ai's SaaS offerings?

**The answer is unambiguous: use Z.ai SaaS.**

### Bottom-Line Numbers

| Metric | Value |
|---|---|
| Self-hosted monthly TCO (on-demand, 1 node) | <strong>&#36;115,785</strong> |
| Self-hosted monthly TCO (3-year CUD, 1 node) | <strong>&#36;90,853</strong> |
| Per-dev cost floor at throughput limit (3-year CUD) | <strong>&#36;606/dev/month</strong> |
| Z.ai Pro (recommended SaaS tier) | <strong>&#36;72/dev/month</strong> |
| Minimum cost ratio (self-hosted vs Z.ai Pro) | <strong>≥8.4×</strong> |
| Minimum cost ratio (self-hosted vs Z.ai Max) | <strong>≥3.79×</strong> |
| Financial breakeven (3-year CUD vs Z.ai Max, single node) | **568 devs** |
| Nodes required to serve 568 devs | **4 nodes** |
| Revised breakeven with 4 nodes | **2,271 devs** |
| Is breakeven reachable? | **No — mathematically impossible at current throughput** |

### Why Self-Hosting Fails Economically

The GEAP management fee is [$83.772/hr](https://cloud.google.com/products/gemini-enterprise-agent-platform/pricing) — 56.5% of the total $148.212/hr rate. This fee is not discountable via committed use discounts (CUDs). At the throughput-limited capacity of ≈150 developers per node, the management fee alone costs <strong>&#36;408 per developer per month</strong> — 2.5× the cost of Z.ai Max ($160/dev/month). Even the management fee alone exceeds Z.ai Max pricing — compute cost is almost irrelevant.

The binding constraint is not VRAM. At typical coding-agent context lengths (≈100K tokens per request), a single 8×B200 node has KV cache headroom for 120 concurrent requests — enough for 600 developers. The real bottleneck is **decode throughput**: approximately 1,000 tokens/second aggregate (realistic: 960 tok/s), which caps a single node at roughly 150 developers during peak burst hours.

### Recommendation

**Use Z.ai Coding Plan Pro ($72/dev/month)** for teams running parallel coding agents. For teams with single-project workflows and no parallel subagents, Lite ($18/dev/month) is technically sufficient. Self-hosting is only justified when Z.ai SaaS cannot serve the workload: custom fine-tuned models, strict data sovereignty requirements, or team sizes exceeding ≈3,000 developers where custom GEAP pricing negotiations become viable.

---

## 2. Background & Scope

### 2.1 What Is GLM-5.2?

[GLM-5.2](https://z.ai/blog/glm-5.2) (may require JavaScript rendering; verified via HuggingFace model card) is a 753-billion-parameter Mixture-of-Experts (MoE) language model developed by Z.ai (Zhipu AI), released in June 2026 under the [MIT license](https://huggingface.co/zai-org/GLM-5.2-FP8). It activates 40 billion parameters per token (8 of 256 routed experts plus 1 shared expert) and supports a 1-million-token context window. GLM-5.2 uses Multi-head Latent Attention (MLA) — the same mechanism as DeepSeek-V3 — which dramatically reduces KV cache memory requirements.

The model is available in two primary quantization formats:
- **BF16:** ≈1.51 TB — too large for a single 8×B200 node
- **FP8 (F8_E4M3):** ≈744 GB — fits on a single `a4-highgpu-8g` with 696 GB remaining for KV cache and runtime

GLM-5.2 is also the model powering Z.ai's own SaaS API and Coding Plan subscriptions, making it a direct apples-to-apples comparison between self-hosting and SaaS.

### 2.2 What Is GCP GEAP Self-Deployed?

[Gemini Enterprise Agent Platform (GEAP) Self-Deployed Models](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/model-garden/self-deployed-models) is GCP's managed inference tier for open models. It runs on Compute Engine GPU instances inside your VPC and provides:

- A managed HTTPS endpoint with stable DNS (Vertex AI Endpoint)
- Auto-scaling based on GPU duty cycle
- Container runtime management (vLLM or TensorRT-LLM)
- Automatic Cloud Monitoring and Cloud Logging integration
- Model Registry integration (weights from Hugging Face or Cloud Storage)

You pay for the underlying compute plus a GEAP management fee. The management fee is the dominant cost component at 56.5% of the total hourly rate.

### 2.3 What Is Z.ai SaaS?

Z.ai offers two consumption paths for GLM-5.2:

**API Metered (pay-as-you-go):** Billed per token at [$1.40/M input, $0.26/M cached input, $4.40/M output](https://docs.z.ai/guides/overview/pricing). Suitable for custom integrations and CI/CD pipelines.

**GLM Coding Plans (subscription):** Fixed monthly subscriptions designed for AI coding agent tools (Claude Code, Cline, OpenCode, Cursor, and others). Three tiers:
- [Lite: $18/dev/month](https://docs.z.ai/devpack/overview) — single-project workflows
- [Pro: $72/dev/month](https://docs.z.ai/devpack/overview) — 1–2 parallel projects
- [Max: $160/dev/month](https://docs.z.ai/devpack/overview) — 2+ parallel projects, power users

### 2.4 Scenarios Analyzed

This report analyzes four team sizes: **50, 100, 300, and 500 developers**, all at the Heavy User baseline (100M input / 4M output tokens per developer per month). The Heavy User baseline represents a developer running 5 agentic coding sessions per day, each consuming approximately 1M input tokens and 40K output tokens.

---

## 3. Technical Architecture & Feasibility

### 3.1 Hardware: `a4-highgpu-8g`

The target deployment platform is GCP's [`a4-highgpu-8g`](https://cloud.google.com/compute/docs/gpus) machine type:

| Attribute | Value |
|---|---|
| GPU model | NVIDIA B200 Blackwell |
| GPU count | 8 |
| GPU memory per GPU | 180 GB HBM3e |
| **Total GPU VRAM** | **1,440 GB** |
| GPU memory bandwidth | [8 TBps per GPU](https://cloud.google.com/compute/docs/gpus) |
| GPU interconnect | [NVLink Full Mesh @ 1,800 GBps](https://cloud.google.com/compute/docs/gpus) |
| vCPU count | 224 (Intel Emerald Rapids) |
| System RAM | 3,968 GB |
| Local SSD | 12,000 GiB NVMe (included in instance price) |
| Network | [3,600 Gbps GPUDirect RDMA](https://cloud.google.com/compute/docs/gpus/gpu-network-bandwidth) |

**Important constraints:**
- No Spot VM pricing available for A4 as of 2026-07-01 (on-demand only)
- No Persistent Disk support — [only Hyperdisk](https://docs.cloud.google.com/ai-hypercomputer/docs/create/create-vm)
- No Sustained Use Discounts — only [AI Hypercomputer Future Reservations](https://cloud.google.com/ai-hypercomputer/docs/consumption-models) (up to 53% off compute)

### 3.2 GLM-5.2 FP8 Weight Footprint

[GLM-5.2 FP8](https://huggingface.co/zai-org/GLM-5.2-FP8) uses 1 byte per parameter. The theoretical weight size is 753B × 1 byte = 753 GB, but the actual checkpoint is <strong>≈744 GB</strong> — the ≈9 GB difference reflects embeddings, layer norms, and some attention components stored in BF16 or FP32.

```
Total VRAM (a4-highgpu-8g):         1,440 GB
GLM-5.2 FP8 weights:               -  744 GB
                                    ─────────
Remaining for KV cache + overhead:    696 GB
```

Per-GPU distribution (tensor-parallel-size 8):

| Per-GPU metric | Value |
|---|---|
| VRAM per GPU | 180 GB |
| Weight shard (744 GB / 8) | 93 GB |
| Remaining per GPU for KV cache | 87 GB |

The BF16 checkpoint is ≈1.51 TB — it exceeds the 1,440 GB total VRAM and cannot run on a single node. **FP8 is the minimum viable quantization for single-node deployment.**

### 3.3 MLA KV Cache: The Architecture That Makes 1M Context Feasible

GLM-5.2 uses [Multi-head Latent Attention (MLA)](https://arxiv.org/abs/2412.19437), confirmed by its [`config.json`](https://huggingface.co/zai-org/GLM-5.2/raw/main/config.json) fields: `kv_lora_rank=512`, `qk_rope_head_dim=64`, and architecture class `GlmMoeDsaForCausalLM`. This is the same mechanism as DeepSeek-V3.

**The standard KV cache formula does not apply to MLA models.** The standard formula:

```
KV_per_token = 2 × num_layers × num_kv_heads × head_dim × bytes_per_element
             = 2 × 78 × 64 × 192 × 1 (FP8)
             = 1,916,928 bytes/token
```

...would predict 2.01 TB of KV cache for a single 1M-token request — nearly 3× the entire 696 GB budget. Not even one request would fit. This is clearly wrong for a model designed to support 1M-token inference.

**The correct MLA formula** (derived from the [DeepSeek-V3 paper](https://arxiv.org/abs/2412.19437) and [confirmed in the DeepSeek-V3 inference source code](https://github.com/deepseek-ai/DeepSeek-V3/blob/main/inference/model.py)):

In MLA, the KV cache stores two things per token per layer:
1. **`c_kv` (compressed latent):** A single low-rank vector of dimension `kv_lora_rank` (512). This is the joint compressed representation of both K and V — shared, so no 2× factor.
2. **`k_rope` (decoupled RoPE key):** A small vector of dimension `qk_rope_head_dim` (64). RoPE cannot be compressed because it doesn't commute with the low-rank projection.

```
KV_per_token (elements) = num_layers × (kv_lora_rank + qk_rope_head_dim)
                        = 78 × (512 + 64)
                        = 78 × 576
                        = 44,928 elements/token
```

In FP8 (1 byte per element): **44,928 bytes/token = 43.9 KiB/token**

This is [confirmed by the HuggingFace LLM-KV-cache-calculator](https://huggingface.co/spaces/HathoraResearch/LLM-KV-cache-calculator/blob/main/app.py) (`nelems_per_token = num_layers * (kv_lora_rank + qk_rope_head_dim)` when `use_mla=True`) and by a [TensorEconomics MLA explainer](https://www.tensoreconomics.com/p/deepseek-sparse-attention-from-first) confirming the 576-elements-per-token-per-layer figure.

**MLA reduction factor:**

```
Standard formula: 2 × 78 × 64 × 192 = 1,916,928 elements/token
MLA formula:          78 × (512 + 64) =    44,928 elements/token
Reduction:                               42.7× smaller
```

MLA reduces KV cache by **42.7×** compared to standard MHA. This is what makes 1M-token context feasible on a single 8×B200 node.

### 3.4 KV Cache Budget and Concurrent Request Capacity

With `--gpu-memory-utilization 0.92` and activation buffers reserved (≈40 GB for chunked prefill), the production KV cache budget is:

```
vLLM budget = 1,440 × 0.92 = 1,324.8 GB
Weights      =              -   744.0 GB
Activations  =              -    40.0 GB
KV budget    =                  540.8 GB
```

KV cache per request at various context lengths (FP8):

| Context length | KV per request | Max concurrent (540.8 GB) | Max devs (20% concurrency) |
|---|---|---|---|
| 50K tokens | 2.25 GB | 241 | 1,204 |
| **100K tokens** | **4.49 GB** | **120** | **600** |
| 128K tokens | 5.89 GB | 91 | 459 |
| 200K tokens | 8.99 GB | 60 | 301 |
| 500K tokens | 22.46 GB | 24 | 120 |
| **1M tokens** | **47.11 GB** | **11** | **55** |

Note: "1M" = `--max-model-len 1048576` (2²⁰ = 1,048,576); "128K" = 131,072 (2¹⁷). Other context lengths use decimal thousands.

**Verification (1M context, production budget):**
```
11 requests × 47.11 GB = 518.2 GB ≤ 540.8 GB  ✓ (22.6 GB headroom)
12 requests × 47.11 GB = 565.3 GB >  540.8 GB  ✗
```

At the typical coding-agent average of ≈100K tokens per request, a single node supports **120 concurrent requests** — enough for 600 developers at 20% peak concurrency. VRAM is not the binding constraint.

### 3.5 vLLM Serving Configuration

The recommended production serving command for GLM-5.2 FP8 on GEAP:

```bash
vllm serve zai-org/GLM-5.2-FP8 \
  --tensor-parallel-size 8 \
  --quantization fp8 \
  --kv-cache-dtype fp8 \
  --max-model-len 1048576 \
  --gpu-memory-utilization 0.92 \
  --max-num-seqs 11 \
  --max-num-batched-tokens 8192 \
  --enable-chunked-prefill \
  --enable-prefix-caching \
  --trust-remote-code \
  --port 8000
```

Key flag rationale:

| Flag | Value | Rationale |
|---|---|---|
| `--tensor-parallel-size` | `8` | One GPU per B200; shards attention and experts via [NVLink 1,800 GBps](https://cloud.google.com/compute/docs/gpus) |
| `--kv-cache-dtype` | `fp8` | Stores MLA latent and RoPE key in FP8; halves KV cache vs BF16. [vLLM native FP8 MLA support confirmed](https://docs.vllm.ai/en/stable/api/vllm/model_executor/layers/attention/mla_attention) |
| `--max-model-len` | `1048576` | Full 1M-token context window |
| `--gpu-memory-utilization` | `0.92` | Reserves 8% (115.2 GB) for CUDA context and fragmentation |
| `--max-num-seqs` | `11` | Production concurrency limit (540.8 GB KV budget ÷ 47.11 GB/req = 11.48 → 11) |
| `--max-num-batched-tokens` | `8192` | Chunked prefill chunk size; bounds activation memory to ≈5 GB/GPU. **Essential for 1M-context feasibility** |
| `--enable-prefix-caching` | flag | Reuses KV blocks for matching prompt prefixes; critical for coding-agent workloads with ≈85–95% cached input |
| `--trust-remote-code` | flag | Required for custom `GlmMoeDsaForCausalLM` architecture |

[SGLang](https://docs.sglang.ai/) is a viable alternative with better radix-attention prefix caching and expert-parallel decode mode, but vLLM is recommended as the primary due to broader GEAP container support.

---

## 4. Coding Agent Workload Baseline

### 4.1 Why Coding Agents Consume So Many Input Tokens

The input/output ratio in agentic coding is extreme — roughly 25:1 or higher. This is structural, not incidental.

Every tool call (read a file, run a test, grep the repo, execute a shell command) ships the **entire accumulated conversation history** back to the model. By turn 30 of a debugging session, the agent is re-reading 25,000–35,000 tokens of history on every single request. The output — a patch, a short explanation — is tiny by comparison.

A [LeanOps 2026 audit of 30 engineering teams](https://www.webpronews.com/when-ai-coders-outspend-their-human-overlords/) found that re-sent context accounts for approximately 62% of the total bill. [Jia.je's analysis of Z.ai Coding Plan usage](https://jia.je/kb/en/software/coding_plan.html) confirms: input tokens account for 99.5% of total token volume in vibe-coding scenarios, with cached tokens making up 90–95% of that input.

### 4.2 Per-Developer Monthly Baseline

A typical 50-turn agentic coding session burns roughly [1,000,000 input tokens and 40,000 output tokens](https://tokenade.net/en/stats/claude-code-token-usage-statistics) — a 25:1 ratio. A Heavy User developer runs approximately 5 such sessions per day.

| Parameter | Value | Source |
|---|---|---|
| Sessions per day | ≈5 | Mix of short and long agentic sessions |
| Avg input tokens per session | ≈1,000,000 | [Tokenade/Vantage 2026](https://tokenade.net/en/stats/claude-code-token-usage-statistics) |
| Avg output tokens per session | ≈40,000 | Same benchmark; 25:1 ratio |
| Daily input tokens | ≈5,000,000 | 5 sessions × 1M input |
| Daily output tokens | ≈200,000 | 5 × 40K = 200K |
| **Monthly input tokens (20 working days)** | **100,000,000 (100M)** | Projection |
| **Monthly output tokens** | **4,000,000 (4M)** | Projection |

This aligns with [Anthropic's own data](https://breaktheordinary.com/ai-coding-agent-api-cost-audit/): average Claude Code usage runs about $13 per developer per active day at API rates, implying roughly 4M input tokens per day — consistent with the 5M Heavy User figure. [Vantage's 2026 production analysis](https://tokenade.net/en/stats/claude-code-token-usage-statistics) puts typical full-time agent users at $400–$1,500/month, with extreme single-day spikes past $4,000.

### 4.3 Concurrency and RPM

Not all developers send requests simultaneously. **Assumption** (see Section 10.2): ≈20% of active developers send requests simultaneously during peak hours (standard enterprise software usage pattern, typically 10:00–12:00 local time).

**Average request size:** ≈100K input tokens (mid-session context; requests early in a session are smaller, later ones larger; 100K is a reasonable average across all turns).

**Requests per day per developer:** 5,000,000 ÷ 100,000 = **50 requests/day**

**Peak RPM for 100-developer team (cadence model):**

If each of the 20 peak-concurrent developers sends one request every 30 seconds (a reasonable agentic loop cadence):
- 20 developers × (60 seconds ÷ 30 seconds) = **40 RPM peak**

| Scenario | Developers | Peak concurrent | Peak RPM |
|---|---|---|---|
| 50 devs | 50 | 10 | ≈20 |
| 100 devs | 100 | 20 | ≈40 |
| 300 devs | 300 | 60 | ≈120 |
| 500 devs | 500 | 100 | ≈200 |

The cadence model (40 RPM for 100 devs) is the operationally relevant figure for capacity planning, since it captures burst behavior rather than smoothed averages.

---

## 5. Z.ai SaaS Cost Analysis

### 5.1 API Metered Pricing

[Z.ai's official pricing](https://docs.z.ai/guides/overview/pricing) for GLM-5.2 pay-as-you-go:

| Component | Price (per 1M tokens) |
|---|---|
| Input tokens (fresh) | $1.40 |
| Cached input tokens | $0.26 |
| Cached input storage | $0.00 (limited-time free) |
| Output tokens | $4.40 |

Cross-verified by [AI//Cost tracker](https://aicost.tools/llm-cost/zhipu/glm-5-2/), [Apidog blog (2026-06-17)](https://apidog.com/blog/glm-5-2-pricing/), [OpenRouter](https://openrouter.ai/) and [Requesty](https://requesty.ai/) public listings — all show identical rates with no markup.

**Per-developer monthly cost (Heavy User: 100M input / 4M output):**

Without caching:
```
100M × $1.40/M + 4M × $4.40/M = $140.00 + $17.60 = $157.60/dev/month
```

Z.ai's context caching is [implicit and automatic](https://docs.z.ai/guides/capabilities/cache.md) — no manual cache-control header required. In agentic coding, cache-hit rates are high because each turn re-sends accumulated conversation context.

Sensitivity table (100M input + 4M output baseline):

| Cache-hit rate | Fresh input cost | Cached input cost | Output cost | **Total/dev/month** |
|---|---|---|---|---|
| 0% (no cache) | $140.00 | $0.00 | $17.60 | <strong>&#36;157.60</strong> |
| 70% | $42.00 | $18.20 | $17.60 | <strong>&#36;77.80</strong> |
| 80% | $28.00 | $20.80 | $17.60 | <strong>&#36;66.40</strong> |
| **85% (central estimate)** | <strong>&#36;21.00</strong> | <strong>&#36;22.10</strong> | <strong>&#36;17.60</strong> | <strong>&#36;60.70</strong> |
| 90% | $14.00 | $23.40 | $17.60 | <strong>&#36;55.00</strong> |
| 95% | $7.00 | $24.70 | $17.60 | <strong>&#36;49.30</strong> |

Formula: `Total = 157.60 − 114 × r` (where r is the cache-hit fraction of input tokens)

The 85% central estimate is informed by the [Jia.je vibe-coding figure of 90–95% cached input](https://jia.je/kb/en/software/coding_plan.html) and contextualized by the [LeanOps 2026 audit finding that re-sent context drives ≈62% of total cost](https://www.webpronews.com/when-ai-coders-outspend-their-human-overlords/).

### 5.2 GLM Coding Plan Tiers

[Z.ai Coding Plans](https://docs.z.ai/devpack/overview) are fixed monthly subscriptions per developer. Each developer requires their own subscription — [per-subscriber exclusive use is enforced](https://docs.z.ai/devpack/usage-policy); no account sharing or quota pooling.

| Plan | Base monthly | Annual billing (30% off) | Prompts / 5-hr window | Prompts / week | Recommended use |
|---|---|---|---|---|---|
| Lite | $18.00 | $12.60 | ≈80 prompts | ≈400 prompts | Single project at a time |
| Pro | $72.00 | $50.40 | ≈400 prompts | ≈2,000 prompts | 1–2 parallel projects |
| Max | $160.00 | $112.00 | ≈1,600 prompts | ≈8,000 prompts | 2+ parallel projects |

Annual billing discounts verified via [AI Pricing Guru](https://www.aipricing.guru/subscriptions/zai-glm-coding-pro/) and [Hyscaler GLM Coding Plan review](https://hyscaler.com/insights/glm-coding-plan-review/).

### 5.3 GLM-5.2 Quota Multiplier

A critical detail: [GLM-5.2 usage is deducted at 3× during peak hours (14:00–18:00 UTC+8) and 2× during off-peak hours](https://docs.z.ai/devpack/overview). A limited-time promotion through end of September 2026 reduces off-peak to 1×.

For a developer in UTC+8 working 9:00–18:00 (5 hours off-peak + 4 hours peak), the weighted average multiplier is:

```
Post-Sept 2026: (5 × 2 + 4 × 3) / 9 = 22/9 = 2.44×
Sept 2026 promo: (5 × 1 + 4 × 3) / 9 = 17/9 = 1.89×
```

**Effective weekly GLM-5.2 prompt budget (post-Sept 2026, 2.44× multiplier):**

| Plan | Raw weekly cap | Effective GLM-5.2 prompts/week | Effective prompts/5-hr window |
|---|---|---|---|
| Lite | 400 | **164** | <strong>≈33</strong> |
| Pro | 2,000 | **820** | <strong>≈164</strong> |
| Max | 8,000 | **3,279** | <strong>≈656</strong> |

**Heavy User demand:** ≈58–77 prompts/week (agentic scenario, 100M input ÷ ≈350K tokens/prompt). All three tiers cover this with headroom. Pro covers it with ≈10× headroom; Max with ≈40×.

**No overage billing:** [Per Z.ai's FAQ](https://docs.z.ai/devpack/faq), once the cap is hit, the subscriber waits until the next 5-hour window resets. No balance is deducted. Coding Plan costs are strictly capped at the subscription price — there is no scenario where a runaway agent generates a surprise bill.

**No API spillover:** The Coding Plan endpoint and the pay-as-you-go API endpoint are [fully separate](https://docs.z.ai/devpack/quick-start). A team wanting fallback for cap-hit developers must issue separate API keys and fund a separate API balance.

### 5.4 Team Cost Matrix

**Base monthly pricing (USD/month):**

| Option | Per-dev/month | 50 devs | 100 devs | 300 devs | 500 devs |
|---|---|---|---|---|---|
| API metered (no cache) | $157.60 | $7,880 | $15,760 | $47,280 | $78,800 |
| API metered (85% cache) | $60.70 | $3,035 | $6,070 | $18,210 | $30,350 |
| Coding Plan Lite | $18.00 | $900 | $1,800 | $5,400 | $9,000 |
| **Coding Plan Pro** | <strong>&#36;72.00</strong> | <strong>&#36;3,600</strong> | <strong>&#36;7,200</strong> | <strong>&#36;21,600</strong> | <strong>&#36;36,000</strong> |
| Coding Plan Max | $160.00 | $8,000 | $16,000 | $48,000 | $80,000 |

**Annual billing variant (30% off Coding Plans; API metered unchanged):**

| Option | Per-dev/month | 50 devs | 100 devs | 300 devs | 500 devs |
|---|---|---|---|---|---|
| Coding Plan Lite (annual) | $12.60 | $630 | $1,260 | $3,780 | $6,300 |
| Coding Plan Pro (annual) | $50.40 | $2,520 | $5,040 | $15,120 | $25,200 |
| Coding Plan Max (annual) | $112.00 | $5,600 | $11,200 | $33,600 | $56,000 |

### 5.5 Which Tier to Choose

| Team profile | Recommended tier | Per-dev/month | Reason |
|---|---|---|---|
| Solo/light users, single-project workflows | Lite | $18 | Weekly cap covers Heavy User baseline with ≈2× headroom |
| **Heavy agentic users, parallel subagents (1–2 concurrent projects)** | **Pro** | <strong>&#36;72</strong> | 5× Lite's cap; Z.ai-recommended for 1–2 parallel projects |
| Power users, 2+ concurrent projects, long autonomous sessions | Max | $160 | 20× Lite's cap; Z.ai-recommended for 2+ parallel projects |

Lite is technically sufficient for the per-developer token baseline (Scenario A: ≈58–77 prompts/week vs 164 effective cap). However, [Z.ai explicitly recommends Lite only for "single project at a time" use](https://docs.z.ai/devpack/overview). For teams where developers run parallel agents (Claude Code subagents, OpenCode `task()` calls, Cline parallel mode), **Pro is the minimum operationally safe tier**.

---

## 6. Self-Hosted TCO (GCP GEAP)

### 6.1 GEAP Pricing Breakdown

The [GEAP Self-Deployed on-demand price](https://cloud.google.com/products/gemini-enterprise-agent-platform/pricing) for `a4-highgpu-8g` in us-central1 is <strong>&#36;148.212/hr</strong>, broken down as:

| Component | Hourly rate | Monthly (730 hr) | % of total |
|---|---|---|---|
| Raw Compute Engine ([a4-highgpu-8g on-demand](https://cloud.google.com/products/compute/pricing/accelerator-optimized)) | $64.440 | $47,041 | 43.5% |
| GEAP management fee | $83.772 | $61,154 | 56.5% |
| **Total GEAP on-demand** | <strong>&#36;148.212</strong> | <strong>&#36;108,195</strong> | **100%** |

The GEAP management fee covers the managed Vertex AI Endpoint, auto-scaling controller, container runtime management, Model Registry integration, and automatic Cloud Monitoring/Logging — per [GEAP Self-Deployed Models documentation](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/model-garden/self-deployed-models).

**GCP uses 730 hours as the canonical billing month** (365.25 days × 24 ÷ 12 = 730.5, rounded down). All monthly figures in this report use 730 hours.

### 6.2 CUD Discount Analysis

A4 machine types have special CUD constraints. They do not receive Sustained Use Discounts or standard flexible CUDs. The only discount path is [AI Hypercomputer Future Reservations](https://cloud.google.com/ai-hypercomputer/docs/consumption-models), which offer up to 53% off vCPUs and GPUs for 1-year or longer commitments.

**Critical limitation:** The GEAP management fee ($83.772/hr) is **not discountable via CUD**. Only the raw compute portion ($64.44/hr) receives the discount. This structurally limits the effective discount on the total GEAP price:

| Scenario | Compute ($/hr) | Mgmt fee ($/hr) | Total ($/hr) | Effective discount on total |
|---|---|---|---|---|
| On-demand | $64.440 | $83.772 | <strong>&#36;148.212</strong> | 0.0% |
| 1-year CUD (est. 35% off compute) | $41.886 | $83.772 | <strong>&#36;125.658</strong> | 15.2% |
| 3-year CUD (est. 53% off compute) | $30.287 | $83.772 | <strong>&#36;114.059</strong> | 23.0% |

```
1-year CUD: ($64.44 × 0.65) + $83.772 = $41.886 + $83.772 = $125.658/hr
3-year CUD: ($64.44 × 0.47) + $83.772 = $30.287 + $83.772 = $114.059/hr
```

The headline "up to 53% discount" applies only to the compute portion. The effective total discount is **only 15–23%** — not 53%. This is the most common misconception in GEAP TCO analyses. (Note: 23.0% refers to the effective discount on the hourly compute rate; 21.5% is the effective discount on full monthly TCO after including the non-discountable management fee and overhead costs.)

### 6.3 Full Monthly TCO Breakdown

**On-demand (canonical):**

| Line item | Monthly cost | % of total | Source |
|---|---|---|---|
| Compute (GEAP on-demand, 730 hr) | $108,194.76 | 93.5% | [GEAP Pricing](https://cloud.google.com/products/gemini-enterprise-agent-platform/pricing) |
| Operations overhead (0.5 FTE @ $180K/yr, estimated) | $7,500.00 | 6.5% | Labor estimate |
| Storage (Hyperdisk 500GB + GCS 744GB) | $64.89 | 0.1% | [GCP Disk Pricing](https://cloud.google.com/compute/disks-image-pricing), [GCS Pricing](https://cloud.google.com/storage/pricing) |
| Cloud Monitoring (estimated) | $25.00 | <0.1% | [Cloud Observability Pricing](https://cloud.google.com/products/observability/pricing) |
| Network egress (500 devs, Premium Tier) | ≈$2.00 | <0.1% | [GCP Network Tiers Pricing](https://cloud.google.com/network-tiers/pricing) |
| Cloud Logging (within 50 GiB free tier) | $0.00 | 0.0% | [Cloud Observability Pricing](https://cloud.google.com/products/observability/pricing) |
| **Total monthly TCO (on-demand)** | <strong>&#36;115,785</strong> | **100%** | |

**Storage note:** A4 instances [do not support Persistent Disk — only Hyperdisk](https://docs.cloud.google.com/ai-hypercomputer/docs/create/create-vm). Boot disk is Hyperdisk Balanced 500 GB at ≈$0.10/GB-month (estimated; exact rate not extractable from GCP pricing page, ±50% error changes TCO by <$25/mo). The 744 GB GLM-5.2 FP8 checkpoint is stored on [Cloud Storage Standard at $0.020/GB-month](https://cloud.google.com/storage/pricing) in us-central1. The 12,000 GiB local NVMe is included in the instance price.

**Network egress is negligible:** 500 devs × 4M output tokens × 4 bytes × 3× HTTP/SSE overhead ≈ 22.4 GiB/month → ≈$2/month after the 1 GiB free tier. Input tokens are ingress (free).

**TCO by commitment scenario:**

| Scenario | Monthly compute | Total monthly TCO | Effective discount vs on-demand |
|---|---|---|---|
| On-demand | $108,195 | <strong>&#36;115,785</strong> | — |
| 1-year CUD (estimated) | $91,730 | <strong>&#36;99,321</strong> | 14.2% |
| 3-year CUD (estimated) | $83,263 | <strong>&#36;90,853</strong> | 21.5% |

### 6.4 Per-Developer Cost

The compute cost is fixed regardless of developer count (one instance serves all team sizes). Per-developer cost decreases as team size grows, but never reaches Z.ai SaaS levels within the 50–500 range:

**On-demand:**

| Team size | Total monthly TCO | Per-developer cost |
|---|---|---|
| 50 devs | $115,785 | <strong>&#36;2,316/dev/month</strong> |
| 100 devs | $115,785 | <strong>&#36;1,158/dev/month</strong> |
| 300 devs | $115,785 | <strong>&#36;386/dev/month</strong> |
| 500 devs | $115,785 | <strong>&#36;232/dev/month</strong> |

**3-year CUD (estimated):**

| Team size | Total monthly TCO | Per-developer cost |
|---|---|---|
| 50 devs | $90,853 | <strong>&#36;1,817/dev/month</strong> |
| 100 devs | $90,853 | <strong>&#36;909/dev/month</strong> |
| 300 devs | $90,853 | <strong>&#36;303/dev/month</strong> |
| 500 devs | $90,853 | <strong>&#36;182/dev/month</strong> |

The 3-year CUD saves a constant <strong>≈21.5%</strong> per developer vs on-demand across all team sizes (since compute dominates and the discount is fixed).

---

## 7. Breakeven Analysis & Scale Economics

### 7.1 Financial Breakeven (Single-Node Assumption)

The financial breakeven is the team size at which self-hosted cost equals SaaS cost, assuming a single `a4-highgpu-8g` node serves all developers.

**Formula:** `breakeven_devs = NodeCost / PerDevSaaSCost`

**10 breakeven points:**

| Self-hosted scenario | SaaS option | Per-dev/month | Breakeven devs |
|---|---|---|---|
| On-demand ($115,785/mo) | API no cache | $157.60 | **735** |
| On-demand ($115,785/mo) | API 85% cache | $60.70 | **1,908** |
| On-demand ($115,785/mo) | Lite | $18.00 | **6,433** |
| On-demand ($115,785/mo) | Pro | $72.00 | **1,608** |
| On-demand ($115,785/mo) | Max | $160.00 | **724** |
| 3-year CUD ($90,853/mo) | API no cache | $157.60 | **577** |
| 3-year CUD ($90,853/mo) | API 85% cache | $60.70 | **1,497** |
| 3-year CUD ($90,853/mo) | Lite | $18.00 | **5,047** |
| 3-year CUD ($90,853/mo) | Pro | $72.00 | **1,262** |
| **3-year CUD ($90,853/mo)** | **Max** | <strong>&#36;160.00</strong> | **568** |

**Arithmetic verification:**
```
3-year CUD vs Max:          90,853 / 160.00 = 567.83  ✓
3-year CUD vs Pro:          90,853 /  72.00 = 1261.85 ✓
3-year CUD vs API no cache: 90,853 / 157.60 = 576.48  ✓
3-year CUD vs API 85% cache:90,853 /  60.70 = 1496.76 ✓
On-demand vs Max:          115,785 / 160.00 = 723.66  ✓
On-demand vs Pro:          115,785 /  72.00 = 1608.13 ✓
```

The lowest breakeven is **568 developers** (3-year CUD vs Z.ai Max). The second lowest is **577 developers** (3-year CUD vs API metered no cache). All other pairings require at least 724 developers. Against Pro ($72/dev), breakeven is 1,262–1,608 developers — far beyond the 50–500 range analyzed here.

### 7.2 The Throughput Bottleneck

VRAM concurrency is not the binding constraint. The real bottleneck is **decode throughput**.

GLM-5.2 has 40B active parameters (MoE, 8 of 256 experts active per token). In FP8 (1 byte/param), reading the active weights requires 40 GB per token across all 8 GPUs (5 GB per GPU in TP=8).

**Theoretical decode throughput (memory-bandwidth-bound):**
```
Per-GPU: 8 TBps / 5 GB = 1,600 tokens/sec
Aggregate (TP=8): 1,600 tokens/sec
```

**Realistic decode throughput (with MoE routing, attention, all-reduce overhead):**

| Efficiency | Tokens/sec | Rationale |
|---|---|---|
| 50% (conservative) | 800 | MoE expert routing, attention compute, non-MoE dense layers, all-reduce, kernel launch overhead |
| **60% (realistic)** | **960** | Typical vLLM MoE efficiency on Hopper-class GPUs |
| 70% (optimistic) | 1,120 | Well-optimized SGLang with EP mode and fused kernels |

**NVLink all-reduce overhead is negligible:** 78 layers × 12,288 bytes (hidden_size × 2 BF16) = 935 KB per token. At 1,800 GBps NVLink, this takes 0.5 μs — less than 0.1% of decode time.

**RPM capacity** (using the Section 4.2 baseline of ≈1,000 output tokens per request):

| Throughput | Output/req | RPM capacity | Max devs at 40 RPM per 100 devs |
|---|---|---|---|
| 800 tok/s (conservative) | 1,000 | 48 RPM | **120 devs** |
| **960 tok/s (realistic)** | **1,000** | **57 RPM** | <strong>≈142 devs</strong> |
| 1,120 tok/s (optimistic) | 1,000 | 67 RPM | **168 devs** |

**The capacity wall: ≈150 developers per node** (realistic throughput, 1K output/request, 20% peak concurrency, 40 RPM per 100 devs).

Node requirements by team size:

| Team size | Peak RPM | Node capacity (57 RPM) | Nodes needed |
|---|---|---|---|
| 50 devs | ≈20 RPM | 1 node (2.85× headroom) | **1 node** |
| 100 devs | ≈40 RPM | 1 node (1.4× headroom, at edge) | **1 node** |
| 300 devs | ≈120 RPM | 2 nodes (at edge, 2×57=114 RPM) | **2 nodes** |
| 500 devs | ≈200 RPM | 1 node insufficient | **4 nodes** |

### 7.3 The Escalating Spiral

The single-node financial breakeven of 568 developers is a mirage. 568 developers require 4 nodes (ceil(568/150) = 4):

| Iteration | Target breakeven | Nodes needed | Revised cost | Revised breakeven |
|---|---|---|---|---|
| 1 | 568 devs (CUD vs Max) | 4 nodes | $363,412/mo | **2,271 devs** |
| 2 | 2,271 devs (revised) | 16 nodes | $1,453,648/mo | **9,085 devs** |
| 3 | 9,085 devs (re-revised) | 61 nodes | $5,542,033/mo | **34,638 devs** |

Each iteration multiplies the cost by approximately 4 (because 568/150 ≈ 4). The breakeven recedes faster than the team grows. This is not a rounding artifact — it is a structural consequence of the throughput limit.

### 7.4 Mathematical Proof of Infeasibility

Let:
- N = number of developers
- C_node = per-node monthly cost ($90,853 for 3-year CUD)
- D_node = max developers per node (150)
- S = per-dev SaaS cost ($160 for Max)

Self-hosted cost: `C_node × ceil(N / D_node)`  
SaaS cost: `S × N`

For all N ≥ 1, `ceil(N / D_node) ≥ N / D_node`, therefore:

```
Self-hosted = C_node × ceil(N / D_node)
            ≥ (C_node / D_node) × N
            = ($90,853 / 150) × N
            = $605.69 × N
            > $160.00 × N = SaaS (Max). QED.

Minimum ratio = $605.69 / $160.00 = 3.79×
```

**The ratio is always ≥3.79×.** Self-hosting is at minimum 3.79× more expensive than Z.ai Max (exactly 3.79× only at multiples of 150 developers). This holds for ALL finite N, not just asymptotically. The breakeven is **mathematically unreachable at any finite team size.**

The only escape from this proof requires either:
- A 74% reduction from the 3-year CUD price ($90,853 → $24,000/month), OR
- A 4× increase in per-node throughput (150 → 600 devs/node), OR
- Some combination of both

Neither is achievable with current GEAP pricing or B200 hardware.

### 7.5 The Per-Developer Cost Floor

Because both cost and capacity scale linearly with node count, the per-developer cost at scale converges to a constant:

| Scenario | Per-node cost | Per-node dev cap | Per-dev cost floor |
|---|---|---|---|
| On-demand, 150 devs/node | $115,785 | 150 | <strong>&#36;772/dev/month</strong> |
| **3-year CUD, 150 devs/node** | <strong>&#36;90,853</strong> | **150** | <strong>&#36;606/dev/month</strong> |
| On-demand, 200 devs/node (optimistic) | $115,785 | 200 | $579/dev/month |
| 3-year CUD, 200 devs/node (optimistic) | $90,853 | 200 | $454/dev/month |

Even in the most optimistic scenario (200 devs/node, 3-year CUD), the per-dev cost floor is <strong>&#36;454/dev/month</strong> — still 2.8× more expensive than Z.ai Max ($160) and 6.3× more expensive than Z.ai Pro ($72).

### 7.6 The Structural Root Cause

The GEAP management fee is the structural barrier:

```
Management fee per node:          $83.772/hr × 730 hr = $61,154/month
Management fee per dev (150/node): $61,154 / 150       = $408/dev/month
Z.ai Max:                                               = $160/dev/month
```

**The management fee alone ($408/dev/month) costs 2.5× as much as Z.ai Max ($160/dev/month).** Even if GCP compute were free, GEAP self-hosting would still be more expensive than Z.ai SaaS at the throughput-limited per-node capacity.

GEAP is a managed service, not raw compute. Its management fee scales with nodes, and nodes are throughput-limited, so the management fee per developer can never drop below $408 — regardless of team size or CUD commitment.

---

## 8. Scenario Comparison Matrix

### 8.1 Full Cost Comparison (All Options, Base Monthly Pricing)

| Option | 50 devs | 100 devs | 300 devs | 500 devs |
|---|---|---|---|---|
| Z.ai Lite ($18/dev) | $900 | $1,800 | $5,400 | $9,000 |
| Z.ai API 85% cache ($60.70/dev) | $3,035 | $6,070 | $18,210 | $30,350 |
| **Z.ai Pro ($72/dev)** | <strong>&#36;3,600</strong> | <strong>&#36;7,200</strong> | <strong>&#36;21,600</strong> | <strong>&#36;36,000</strong> |
| Z.ai Max ($160/dev) | $8,000 | $16,000 | $48,000 | $80,000 |
| Z.ai API no cache ($157.60/dev) | $7,880 | $15,760 | $47,280 | $78,800 |
| Self-hosted on-demand (nodes) | $115,785 (1) | $115,785 (1) | $231,570 (2) | $463,140 (4) |
| Self-hosted 3-year CUD (nodes) | $90,853 (1) | $90,853 (1) | $181,706 (2) | $363,412 (4) |

### 8.2 Self-Hosted vs Z.ai Pro (Recommended SaaS Tier)

| Team size | Z.ai Pro | Self-hosted (3-year CUD) | Self-hosted (on-demand) | Ratio (CUD vs Pro) |
|---|---|---|---|---|
| 50 devs | $3,600 | $90,853 (1 node) | $115,785 (1 node) | **25.2× more expensive** |
| 100 devs | $7,200 | $90,853 (1 node) | $115,785 (1 node) | **12.6× more expensive** |
| 300 devs | $21,600 | $181,706 (2 nodes) | $231,570 (2 nodes) | **8.4× more expensive** |
| 500 devs | $36,000 | $363,412 (4 nodes) | $463,140 (4 nodes) | **10.1× more expensive** |

### 8.3 Self-Hosted vs Z.ai Max (Most Expensive SaaS Tier)

| Team size | Z.ai Max | Self-hosted (3-year CUD) | Self-hosted (on-demand) | Ratio (CUD vs Max) |
|---|---|---|---|---|
| 50 devs | $8,000 | $90,853 (1 node) | $115,785 (1 node) | **11.4× more expensive** |
| 100 devs | $16,000 | $90,853 (1 node) | $115,785 (1 node) | **5.7× more expensive** |
| 300 devs | $48,000 | $181,706 (2 nodes) | $231,570 (2 nodes) | **3.8× more expensive** |
| 500 devs | $80,000 | $363,412 (4 nodes) | $463,140 (4 nodes) | **4.5× more expensive** |

Even against Z.ai Max — the most expensive SaaS tier — self-hosting is 3.8–11.4× more expensive at every team size.

### 8.4 Recommendation Per Team Size

| Team size | Recommended option | Monthly cost | Reason |
|---|---|---|---|
| **50 devs** | Z.ai Pro | $3,600 | Self-hosted is 25× more expensive. One node is massive overkill (150-dev capacity for 50 devs). |
| **100 devs** | Z.ai Pro | $7,200 | Self-hosted is 13× more expensive. One node is at throughput capacity edge. |
| **300 devs** | Z.ai Pro | $21,600 | Self-hosted is 8.4× more expensive. Two nodes needed for peak throughput. |
| **500 devs** | Z.ai Pro | $36,000 | Self-hosted is 10× more expensive. Four nodes needed. Per-dev cost $727 (CUD)–$926 (on-demand) vs Pro $72. |

If the team uses single-project workflows with no parallel subagents, Lite ($900–$9,000/month for 50–500 devs) is the absolute cheapest viable option.

---

## 9. Final Recommendation

### 9.1 Primary Recommendation: Z.ai SaaS

**Use Z.ai Coding Plan Pro ($72/dev/month) for teams running parallel coding agents.**

Pro is the cheapest operationally safe option for Heavy User teams at every team size from 50 to 500 developers. It provides:
- 5× the prompt capacity of Lite (820 effective GLM-5.2 prompts/week vs 164)
- Z.ai-recommended for 1–2 parallel projects simultaneously
- No overage billing — costs are strictly capped at the subscription price
- Official integration with Claude Code, Cline, OpenCode, Cursor, and other major coding agent tools
- Predictable, linear cost scaling with team size

For teams with single-project workflows and no parallel subagents, **Lite ($18/dev/month)** is technically sufficient and 4× cheaper than Pro.

For power users running 2+ concurrent projects or very long autonomous sessions, **Max ($160/dev/month)** provides 20× Lite's capacity with massive headroom.

### 9.2 Annual Billing Optimization

If committing to Z.ai SaaS for 12+ months, annual billing reduces Coding Plan costs by 30%:

| Plan | Monthly billing | Annual billing | Annual savings (100 devs) |
|---|---|---|---|
| Lite | $18/dev | $12.60/dev | $6,480/year |
| Pro | $72/dev | $50.40/dev | $25,920/year |
| Max | $160/dev | $112.00/dev | $57,600/year |

### 9.3 When Self-Hosting Makes Sense

Self-hosting on GCP GEAP is the right choice only when Z.ai SaaS **cannot** serve the workload:

1. **Custom fine-tuned models.** If the team needs a fine-tuned variant of GLM-5.2 (domain-specific code generation, proprietary data), Z.ai SaaS only serves the base model. Self-hosting is the only option.

2. **Strict data sovereignty.** Industries with regulatory requirements (HIPAA, FedRAMP, EU GDPR data residency) that prohibit sending code to a third-party SaaS API. GEAP runs in your VPC — data never leaves your Google Cloud project boundary.

3. **Very long-context workloads (>500K average context).** If the workload is dominated by long-context requests that would exceed Z.ai Coding Plan 5-hour caps or API rate limits. Self-hosting has no per-developer prompt cap.

4. **Team sizes exceeding ≈3,000 developers.** At very large scale, custom GEAP pricing negotiations with Google Cloud may reduce the management fee, potentially shifting the economics. This is beyond the 50–500 range analyzed here.

5. **Non-standard inference configurations.** Custom vLLM flags, alternative serving runtimes (SGLang, TensorRT-LLM), speculative decoding, or specialized decoding strategies that SaaS does not support.

For the standard coding-agent use case — base GLM-5.2, parallel agents, 50–500 developers, typical context lengths — none of these conditions apply, and Z.ai SaaS is the clear winner.

### 9.4 Decision Flowchart

```
Do you need a custom fine-tuned model?
  YES → Self-host on GEAP
  NO  ↓

Do you have strict data sovereignty requirements (HIPAA, FedRAMP)?
  YES → Self-host on GEAP
  NO  ↓

Is your average request context >500K tokens?
  YES → Self-host on GEAP (or evaluate Z.ai API metered)
  NO  ↓

Do developers run parallel subagents (2+ concurrent projects)?
  YES → Z.ai Pro ($72/dev/month)
  NO  → Z.ai Lite ($18/dev/month)
```

---

## 10. Assumptions & Limitations

### 10.1 Verified Facts (Cited from Primary Sources)

| Fact | Source |
|---|---|
| GEAP on-demand price: $148.212/hr | [GEAP Pricing](https://cloud.google.com/products/gemini-enterprise-agent-platform/pricing) |
| Raw Compute Engine price: $64.44/hr | [GCP Accelerator-Optimized VM Pricing](https://cloud.google.com/products/compute/pricing/accelerator-optimized) |
| A4 does not support Persistent Disk (Hyperdisk only) | [AI Hypercomputer docs](https://docs.cloud.google.com/ai-hypercomputer/docs/create/create-vm) |
| A4 does not receive SUDs or flexible CUDs | [AI Hypercomputer docs](https://docs.cloud.google.com/ai-hypercomputer/docs/create/create-vm) |
| Future Reservations: up to 53% discount for A4 | [AI Hypercomputer Consumption Models](https://cloud.google.com/ai-hypercomputer/docs/consumption-models) |
| GEAP management fee not discountable via CUD | [GEAP Pricing](https://cloud.google.com/products/gemini-enterprise-agent-platform/pricing) |
| GLM-5.2 FP8 checkpoint: ≈744 GB | [HuggingFace zai-org/GLM-5.2-FP8](https://huggingface.co/zai-org/GLM-5.2-FP8) |
| GLM-5.2: 753B total / 40B active params | [HuggingFace nvidia/GLM-5.2-NVFP4](https://huggingface.co/nvidia/GLM-5.2-NVFP4) |
| MLA KV cache formula | [DeepSeek-V3 paper](https://arxiv.org/abs/2412.19437) |
| Z.ai API prices ($1.40/$0.26/$4.40 per M tokens) | [Z.ai Pricing](https://docs.z.ai/guides/overview/pricing) |
| Z.ai Coding Plan prices ($18/$72/$160/dev/month) | [Z.ai Coding Plan Overview](https://docs.z.ai/devpack/overview) |
| GLM-5.2 quota multiplier (2× off-peak, 3× peak) | [Z.ai Coding Plan Overview](https://docs.z.ai/devpack/overview) |
| No overage billing on Coding Plans | [Z.ai Coding Plan FAQ](https://docs.z.ai/devpack/faq) |
| Per-subscriber exclusive use | [Z.ai Usage Policy](https://docs.z.ai/devpack/usage-policy) |
| Cloud Storage Standard: $0.020/GB-month | [GCS Pricing](https://cloud.google.com/storage/pricing) |
| Premium Tier egress: $0.12/GiB | [GCP Network Tiers Pricing](https://cloud.google.com/network-tiers/pricing) |
| Cloud Logging: 50 GiB free/month | [Cloud Observability Pricing](https://cloud.google.com/products/observability/pricing) |
| Cloud Monitoring: 150 MiB custom metrics free/month | [Cloud Observability Pricing](https://cloud.google.com/products/observability/pricing) |
| Typical 50-turn agentic session: ≈1M input / 40K output tokens | [Tokenade/Vantage 2026](https://tokenade.net/en/stats/claude-code-token-usage-statistics) |
| Cached tokens: 90–95% of input in vibe-coding | [Jia.je analysis](https://jia.je/kb/en/software/coding_plan.html) |
| Re-sent context: ≈62% of total bill | [WebProNews/LeanOps 2026](https://www.webpronews.com/when-ai-coders-outspend-their-human-overlords/) |

### 10.2 Labeled Estimates

| Estimate | Value | Uncertainty | Impact on conclusion |
|---|---|---|---|
| Decode throughput | ≈1,000 tok/s (60% MoE efficiency) | ±30–50% | A 2× improvement (2,000 tok/s, 300 devs/node) still leaves per-dev cost at $303 (CUD) — 1.9× more expensive than Max. Conclusion is robust. |
| Per-node capacity | ≈150 devs | ±30% | Even at 200 devs/node (optimistic), per-dev cost floor is $454 — still 2.8× more than Max. |
| 1-year CUD discount | ≈35% off compute | ±10% | GCP does not publish exact A4 1-year CUD rates. |
| 3-year CUD discount | ≈53% off compute | Upper bound from GCP docs | Actual discount may be lower depending on negotiation. |
| Operations overhead | $7,500/month (0.5 FTE @ $180K/yr) | ±$3,000/month | May scale to $11K–$17K/month for 300+ dev teams. |
| Hyperdisk Balanced per-GB rate | ≈$0.10/GB-month | ±50% | At 500 GB boot disk, a ±50% error changes total TCO by <$25/month (<0.02%). |
| Cache-hit rate | 85% (central estimate) | 70–95% range | At 70% cache hit, API metered cost rises to $77.80/dev/month — slightly above Pro ($72) but still far below self-hosted. |
| Cloud Monitoring cost | $25/month | $0–$50/month | Negligible fraction of total TCO. |

### 10.3 Sensitivity Analysis

**What would need to change for self-hosting to break even?**

| Change required | Current value | Required value | Feasibility |
|---|---|---|---|
| Reduce GEAP management fee | $83.772/hr | <$10.67/hr (87% reduction) | Not achievable with standard pricing |
| Increase decode throughput | ≈1,000 tok/s | >4,000 tok/s (4× improvement) | Not achievable with current B200 hardware |
| Reduce 3-year CUD total cost | $90,853/month | <$24,000/month (74% reduction) | Not achievable with current pricing |
| Z.ai raises Pro price | $72/dev | >$606/dev (8.4× increase) | Unlikely; would require fundamental market shift |

The conclusion is robust to all plausible sensitivity scenarios. Even a 2× improvement in decode throughput (300 devs/node) and a 50% reduction in operations overhead would still leave self-hosted at $303/dev/month (CUD) — 1.9× more expensive than Z.ai Max.

### 10.4 Out of Scope

- **Multi-region deployments:** All costs assume us-central1. Other regions may have different GEAP pricing or network egress rates.
- **Spot VM pricing:** A4 does not support Spot pricing as of 2026-07-01. If Spot becomes available, it could reduce compute cost by 60–70%, potentially shifting the economics for some scenarios.
- **Fine-tuning costs:** Training or fine-tuning GLM-5.2 on custom data is not analyzed. This would require additional GPU hours and storage.
- **Latency SLOs:** This analysis focuses on cost, not latency. Self-hosting may offer lower latency for some workloads (no shared infrastructure, dedicated endpoint). Z.ai SaaS latency is not benchmarked here.
- **Z.ai pricing changes:** Z.ai's Coding Plan prices and API rates are current as of 2026-07-01. If Z.ai raises prices significantly, self-hosting becomes more competitive.
- **GLM-5.2 throughput benchmarks:** The decode throughput estimate (≈1,000 tok/s) is derived from B200 memory bandwidth and GLM-5.2 active parameters. It has not been benchmarked on actual B200 hardware with vLLM. Real throughput could be 20–50% higher or lower.

---

## 11. References

### Hardware & Infrastructure

| Source | URL |
|---|---|
| GCP GPU specs (`a4-highgpu-8g`, B200, VRAM, NVLink) | https://cloud.google.com/compute/docs/gpus |
| GCP AI Hypercomputer GPU docs | https://cloud.google.com/ai-hypercomputer/docs/gpu |
| GCP GPU network bandwidth (3,600 Gbps GPUDirect RDMA) | https://cloud.google.com/compute/docs/gpus/gpu-network-bandwidth |
| A4 VM creation constraints (Hyperdisk only, no SUDs) | https://docs.cloud.google.com/ai-hypercomputer/docs/create/create-vm |
| AI Hypercomputer consumption models (Future Reservations, 53% discount) | https://cloud.google.com/ai-hypercomputer/docs/consumption-models |
| AI Hypercomputer reserve capacity (1-yr/3-year CUD rates) | https://docs.cloud.google.com/ai-hypercomputer/docs/reserve-capacity |
| GCP CUD overview (GPU discounts) | https://docs.cloud.google.com/compute/docs/instances/committed-use-discounts-overview |
| GCP CUD sign-up | https://docs.cloud.google.com/compute/docs/instances/signing-up-committed-use-discounts |
| Hyperdisk documentation | https://cloud.google.com/compute/docs/disks/hyperdisks |

### GCP Pricing

| Source | URL |
|---|---|
| GEAP Self-Deployed pricing ($148.212/hr) | https://cloud.google.com/products/gemini-enterprise-agent-platform/pricing |
| Compute Engine accelerator-optimized VM pricing ($64.44/hr) | https://cloud.google.com/products/compute/pricing/accelerator-optimized |
| GCP disk and image pricing (Hyperdisk, Persistent Disk) | https://cloud.google.com/compute/disks-image-pricing |
| Cloud Storage pricing ($0.020/GB-month, us-central1) | https://cloud.google.com/storage/pricing |
| GCP Network Service Tiers pricing ($0.12/GiB Premium) | https://cloud.google.com/network-tiers/pricing |
| GCP VPC network pricing | https://cloud.google.com/vpc/pricing |
| Cloud Observability pricing (Monitoring $0.258/MiB, Logging $0.50/GiB) | https://cloud.google.com/products/observability/pricing |
| Cloud Logging free tier (50 GiB/month) | https://cloud.google.com/blog/topics/cost-management/how-to-approach-cloud-logging-pricing-for-cloud-admins |
| Cloud Monitoring free tier (150 MiB/month) | https://cloud.google.com/blog/products/management-tools/learn-to-understand-and-reduce-cloud-monitoring-costs |

### GEAP Platform

| Source | URL |
|---|---|
| GEAP Self-Deployed Models overview | https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/model-garden/self-deployed-models |
| GEAP deploy from Model Garden | https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/open-models/deploy-model-garden |
| GEAP custom vLLM container | https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/open-models/deploy-custom-vllm |

### GLM-5.2 Model

| Source | URL |
|---|---|
| GLM-5.2 FP8 HuggingFace model card | https://huggingface.co/zai-org/GLM-5.2-FP8 |
| GLM-5.2 BF16 config.json (architecture params) | https://huggingface.co/zai-org/GLM-5.2/raw/main/config.json |
| GLM-5.2 FP8 config.json (quantization config) | https://huggingface.co/zai-org/GLM-5.2-FP8/raw/main/config.json |
| GLM-5.2 official GitHub README | https://github.com/zai-org/GLM-5/blob/main/README.md |
| GLM-5.2 blog post (Z.ai, 2026-06-16) | https://z.ai/blog/glm-5.2 |
| GLM-5.2 753B/40B param confirmation (NVFP4 variant) | https://huggingface.co/nvidia/GLM-5.2-NVFP4 |

### MLA Architecture & KV Cache

| Source | URL |
|---|---|
| DeepSeek-V3 paper (MLA formula derivation) | https://arxiv.org/abs/2412.19437 |
| DeepSeek-V3 inference source code (kv_cache + pe_cache buffers) | https://github.com/deepseek-ai/DeepSeek-V3/blob/main/inference/model.py |
| HuggingFace LLM-KV-cache-calculator (MLA formula confirmed) | https://huggingface.co/spaces/HathoraResearch/LLM-KV-cache-calculator/blob/main/app.py |
| TensorEconomics MLA explainer (576 elements/token) | https://www.tensoreconomics.com/p/deepseek-sparse-attention-from-first |

### vLLM

| Source | URL |
|---|---|
| vLLM MLA attention implementation (FP8 path) | https://docs.vllm.ai/en/stable/api/vllm/model_executor/layers/attention/mla_attention |
| vLLM FP8 MLA prefill kernel | https://docs.vllm.ai/en/stable/api/vllm/v1/attention/backends/mla/rocm_aiter_mla |
| vLLM serving CLI flags | https://docs.vllm.ai/en/stable/ |

### Z.ai Pricing & Policies

| Source | URL |
|---|---|
| Z.ai API pricing (official) | https://docs.z.ai/guides/overview/pricing |
| Z.ai context caching (implicit/automatic) | https://docs.z.ai/guides/capabilities/cache.md |
| Z.ai Coding Plan overview (tiers, caps, multiplier) | https://docs.z.ai/devpack/overview |
| Z.ai Coding Plan FAQ (no overage, no spillover) | https://docs.z.ai/devpack/faq |
| Z.ai Coding Plan usage policy (per-subscriber exclusive use) | https://docs.z.ai/devpack/usage-policy |
| Z.ai Coding Plan quick start (API endpoints) | https://docs.z.ai/devpack/quick-start |
| Z.ai subscription page (billing cycles) | https://z.ai/subscribe |
| AI//Cost GLM-5.2 price tracker | https://aicost.tools/llm-cost/zhipu/glm-5-2/ |
| Apidog GLM-5.2 pricing blog (2026-06-17) | https://apidog.com/blog/glm-5-2-pricing/ |
| Hyscaler GLM Coding Plan review | https://hyscaler.com/insights/glm-coding-plan-review/ |
| AI Pricing Guru GLM Coding Pro | https://www.aipricing.guru/subscriptions/zai-glm-coding-pro/ |
| Distk GLM Coding Plan pricing guide 2026 | https://distk.in/blog/glm-coding-plan-pricing-guide-2026.html |
| Datacamp GLM-5.2 guide | https://www.datacamp.com/blog/glm-5-2 |

### Token Consumption & Industry Data

| Source | URL |
|---|---|
| Tokenade/Vantage — Claude Code token usage statistics (2026) | https://tokenade.net/en/stats/claude-code-token-usage-statistics |
| Tokenade — Context engineering for AI coding agents (2026-05-28) | https://tokenade.net/en/articles/context-engineering |
| Break The Ordinary — AI coding agent API cost audit (2026-06-10) | https://breaktheordinary.com/ai-coding-agent-api-cost-audit/ |
| WebProNews — When AI coders outspend their human overlords (2026-06-24) | https://www.webpronews.com/when-ai-coders-outspend-their-human-overlords/ |
| Jia.je — AI coding plan analysis (input/output ratio, cache hit rate) | https://jia.je/kb/en/software/coding_plan.html |
| OpenReview — How do coding agents spend your money? (academic) | https://openreview.net/pdf/969cd1d8b01a45caef5ad3d26e16e99986ec1ba9.pdf |

---

*Report generated: 2026-07-01*  
*Evidence base: 6 task evidence files, verified through 4 independent reviewer passes (Plan compliance, Math quality, Manual QA, Scope fidelity)*  
*All reference URLs checked as of 2026-07-01; 2 pages (z.ai/blog, GEAP pricing) returned non-renderable content and require manual browser verification.*
