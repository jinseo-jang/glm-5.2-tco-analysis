**English** | [한국어](TCO-GLM-5.2-Final-Report-KO.md)

# TCO Analysis: GLM-5.2 FP8 Self-Hosted (GCP) vs Z.ai SaaS
## AI Coding Agent Workloads — 50 to 500 Developers

**Pricing basis date:** 2026-07-01 (all prices and plan terms are as of this date unless stated otherwise)  
**Revision:** v2, 2026-11-12 — sections 3, 4, 6, 7, 8 and 10 rewritten after a fact-check (see [Revision history](#revision-history))  
**Scope:** Team sizes of 50, 100, 300 and 500 developers; Heavy User baseline (100M input / 4M output tokens per developer per month)

---

> **TL;DR**
>
> For teams of 50–500 developers, **Z.ai GLM Coding Plan Pro (72 USD per developer per month) is cheaper than every self-hosting option we analyzed**, and this holds regardless of how many developers one GPU node can serve. The cheapest self-hosted production option (one `a4-highgpu-8g` node on GKE/Compute Engine with a 3-year commitment) costs about 56,500 USD per month, while Pro for 500 developers costs 36,000 USD per month.
>
> Above 500 developers the answer depends on per-node capacity, which has **not been measured** under concurrent load. Self-hosting beats Pro only if one node can serve roughly 780–880 developers; it beats Z.ai Max (160 USD) at roughly 350–400 developers per node. A load test is required before any decision at that scale.

### What changed since v1

Version 1 (2026-07-01) concluded that self-hosting was "mathematically impossible at any team size". That conclusion rested on three premises that turned out to be wrong:

1. **A "GEAP management fee" of 83.772 USD/hr.** v1 obtained this by subtracting 64.44 USD/hr from the GEAP node price of 148.212 USD/hr. But 64.44 USD/hr is the A4 **DWS Flex-start** price (a discounted, at most 7-day provisioning mode), not an on-demand price. A4 has no Compute Engine on-demand price at all. The real management fee is not published for A4; we estimate it at about 19.33 USD/hr (section 6.2).
2. **"A4 has no Spot pricing and no CUD rates."** Both are published: Spot 37.7464 USD/hr, 1-year resource-based CUD 88.9272 USD/hr, 3-year 56.7072 USD/hr (Compute Engine pricing page, 2026-07-13 snapshot).
3. **A node throughput of ≈960 tokens/s.** v1 derived this from a single-request bandwidth bound and used it as the whole node's capacity. Batched serving processes many requests per decode step, so node throughput is a multiple of single-stream speed (section 7).

The recommendation for 50–500 developers (Pro) did not change, but its justification did.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Background & Scope](#2-background--scope)
3. [Technical Architecture & Feasibility](#3-technical-architecture--feasibility)
4. [Coding Agent Workload Baseline](#4-coding-agent-workload-baseline)
5. [Z.ai SaaS Cost Analysis](#5-zai-saas-cost-analysis)
6. [Self-Hosted TCO (GCP)](#6-self-hosted-tco-gcp)
7. [Node Capacity and Breakeven](#7-node-capacity-and-breakeven)
8. [Scenario Comparison Matrix](#8-scenario-comparison-matrix)
9. [Final Recommendation](#9-final-recommendation)
10. [Assumptions & Limitations](#10-assumptions--limitations)
11. [Changes After the Pricing Basis Date](#11-changes-after-the-pricing-basis-date)
12. [References](#12-references)

---

## 1. Executive Summary

This report answers one question: for a software engineering team of 50 to 500 developers using AI coding agents powered by GLM-5.2, is it cheaper to self-host the model on Google Cloud or to use Z.ai's SaaS offerings?

**Answer for 50–500 developers: use Z.ai Coding Plan Pro.**

### Bottom-line numbers (per month, one `a4-highgpu-8g` node)

| Option | Monthly cost of one node incl. operations | Notes |
|---|---|---|
| GEAP managed endpoint, on-demand | 115,776 USD | Published GEAP price 148.212 USD/hr |
| GEAP + 1-year CUD reservation | 86,611 USD | Management fee estimated (section 6.2) |
| GEAP + 3-year CUD reservation | 63,090 USD | Management fee estimated |
| GKE / Compute Engine, 3-year CUD (no GEAP) | 56,478 USD | Assumes 1.0 FTE operations instead of 0.5 |
| GEAP + Spot | 49,249 USD | Can be preempted; not suitable as the only node |
| **Z.ai Pro for 500 developers** | **36,000 USD** | 72 USD × 500 |

Even the cheapest node option (Spot, which Google can reclaim at any time) costs more than Pro for the largest team in scope. A production deployment also needs a second node for redundancy, which doubles the self-hosted figures. Therefore Pro is cheaper for every team size from 50 to 500, **independent of the unmeasured per-node capacity**.

### Where the answer becomes uncertain

The breakeven between self-hosting and SaaS is `node cost ÷ SaaS price per developer`, expressed as "developers one node must serve":

| Self-hosted option | vs Pro (72 USD) | vs API, 85% cache (60.70 USD) | vs Max (160 USD) |
|---|---|---|---|
| GEAP + 3-year CUD | 876 devs/node | 1,039 devs/node | 394 devs/node |
| GKE/GCE + 3-year CUD | 784 devs/node | 930 devs/node | 353 devs/node |

Our rough estimate of per-node capacity is **about 100–900 developers** (section 7.3). This estimate is not a measurement. The Max breakeven sits inside that range, and the Pro breakeven sits at its upper edge. For teams above 500 developers, the decision therefore depends on a concurrent-load benchmark that has not been run.

---

## 2. Background & Scope

### 2.1 What Is GLM-5.2?

[GLM-5.2](https://huggingface.co/zai-org/GLM-5.2) is a Mixture-of-Experts (MoE) language model released by Z.ai (Zhipu AI) on 2026-06-16 under the MIT license. Facts from the model's `config.json` and model cards (checked 2026-11-12):

- **Parameters:** 753B in total and 40B activated per token, per [NVIDIA's GLM-5.2-NVFP4 model card](https://huggingface.co/nvidia/GLM-5.2-NVFP4). Z.ai's own card does not state a count. The sum of tensor sizes in the checkpoint is 753.3B, which includes one multi-token-prediction (MTP) layer. (The earlier GLM-5 was described as "744B"; we could not confirm whether the 9B difference is the MTP layer.)
- **Experts:** 256 routed experts, 8 active per token, plus 1 shared expert. The first 3 layers are dense; the remaining 75 are MoE.
- **Context window:** 1,048,576 tokens (`max_position_embeddings`).
- **Attention:** Multi-head Latent Attention (MLA, as in DeepSeek-V3) combined with DeepSeek Sparse Attention (DSA). DSA uses a small "indexer" to select the 2,048 most relevant past tokens for each query (`index_topk: 2048`).

Checkpoint sizes, from the Hugging Face API (sum of `*.safetensors` files):

- **BF16:** 1,506.7 GB — larger than one node's 1,440 GB of GPU memory.
- **FP8:** **755.6 GB** — fits on one `a4-highgpu-8g` with 684.4 GB left. (v1 said 744 GB; that figure appears to be GLM-5's parameter count mistaken for a file size.)

GLM-5.2 was also the model behind Z.ai's API and Coding Plans on the pricing basis date, which makes the comparison like-for-like.

### 2.2 What Is GCP GEAP Self-Deployed?

Gemini Enterprise Agent Platform (GEAP) is the 2026 name of Vertex AI; `cloud.google.com/vertex-ai/pricing` now redirects to the GEAP pricing page. [GEAP Self-Deployed Models](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/model-garden/self-deployed-models) lets you run open models on dedicated GPU nodes inside your project and VPC, with a managed HTTPS endpoint, autoscaling, and Cloud Monitoring/Logging integration.

There are two ways to pay for a GEAP node:

- **GEAP node price** (for example 148.212 USD/hr for `a4-highgpu-8g`), billed as one SKU.
- **Bring your own capacity** (a Compute Engine reservation, Spot or Flex-start). Billing then splits into a Compute Engine SKU, which receives Compute Engine prices including CUDs, and a separate **Agent Platform management fee** SKU, which does not receive CUDs ([Use reservations with online inference](https://docs.cloud.google.com/gemini-enterprise-agent-platform/machine-learning/predictions/use-reservations)).

The alternative is to skip GEAP and run vLLM yourself on Compute Engine or GKE. That avoids the management fee but moves endpoint, scaling and monitoring work to your team.

### 2.3 What Is Z.ai SaaS?

Z.ai offers two ways to consume GLM-5.2:

**API metered (pay-as-you-go):** 1.40 USD per million input tokens, 0.26 USD per million cached input tokens and 4.40 USD per million output tokens ([Z.ai pricing](https://docs.z.ai/guides/overview/pricing), confirmed in the 2026-06-29 snapshot).

**GLM Coding Plans (subscription):** fixed monthly subscriptions for coding agent tools such as Claude Code, Cline, OpenCode and Cursor. On the pricing basis date there were three tiers: Lite 18 USD, Pro 72 USD and Max 160 USD per developer per month. Z.ai's own documentation only states "starting at just 18 USD per month"; the Pro and Max prices come from third-party reviews from June–July 2026 ([HyScaler](https://hyscaler.com/insights/glm-coding-plan-review/), [AI Pricing Guru](https://www.aipricing.guru/subscriptions/zai-glm-coding-pro/)), because the subscription page renders prices with JavaScript and could not be archived.

### 2.4 Scenarios Analyzed

Four team sizes: **50, 100, 300 and 500 developers**, all at the Heavy User baseline (100M input and 4M output tokens per developer per month; section 4).

---

## 3. Technical Architecture & Feasibility

### 3.1 Hardware: `a4-highgpu-8g`

All values below were confirmed against [GCP GPU documentation](https://docs.cloud.google.com/compute/docs/gpus) and the [accelerator-optimized machine documentation](https://docs.cloud.google.com/compute/docs/accelerator-optimized-machines).

| Attribute | Value |
|---|---|
| GPU | 8 × NVIDIA B200 (Blackwell) |
| GPU memory | 180 GB HBM3e per GPU, 1,440 GB total |
| GPU memory bandwidth | 8 TBps per GPU |
| GPU interconnect | NVLink full mesh, 1,800 GBps per GPU |
| vCPUs / system memory | 224 (Intel Emerald Rapids) / 3,968 GB |
| Local SSD | 12,000 GiB (included) |
| Network | 3,600 Gbps GPUDirect RDMA |

**How A4 capacity can be obtained (as of July 2026):**

- A4 **cannot be obtained on demand**; the Compute Engine pricing page lists the on-demand price as "N/A". Google's documentation says you must reserve capacity, use Spot VMs, use Flex-start VMs, or create a resize request in a managed instance group.
- Supported options: **Spot VMs**, **Flex-start** (up to 7 days), **calendar-mode future reservations**, **capacity blocks**, and **resource-based committed use discounts (CUDs)**.
- A4 does not receive sustained use discounts or flexible (spend-based) CUDs.
- A4 supports only Hyperdisk, not Persistent Disk.

### 3.2 GLM-5.2 FP8 Memory Footprint

```
Total GPU memory (a4-highgpu-8g):   1,440.0 GB
GLM-5.2 FP8 weights:              -   755.6 GB
                                    ──────────
Remaining:                            684.4 GB
```

With tensor parallelism across 8 GPUs, each GPU holds about 94.5 GB of weights and has about 85.5 GB left. The BF16 checkpoint (1,506.7 GB) does not fit on one node, so **FP8 is the smallest standard format that fits on a single node**.

### 3.3 KV Cache Size per Token

The KV cache stores per-token state from earlier tokens so that each new token does not recompute it. GLM-5.2's attention design makes it much smaller than in a conventional transformer.

**Conventional multi-head attention (for comparison only).** With 78 layers, 64 key/value heads, key dimension 256 (`qk_head_dim`) and value dimension 256 (`v_head_dim`), each token would need 78 × 64 × (256 + 256) = 2,555,904 elements. (v1 used 192 for the key dimension; 192 is only the non-RoPE part, `qk_nope_head_dim`.)

**MLA.** Each layer stores one compressed latent vector of 512 elements (`kv_lora_rank`) plus a 64-element positional key (`qk_rope_head_dim`), so 78 × 576 = 44,928 elements per token ([DeepSeek-V3 paper](https://arxiv.org/abs/2412.19437), [DeepSeek-V3 inference code](https://github.com/deepseek-ai/DeepSeek-V3/blob/main/inference/model.py)).

**DSA indexer.** The indexer keeps its own 128-element key (`index_head_dim`) per token. According to `indexer_types` in `config.json`, 21 of the 78 layers have their own indexer and the other 57 reuse one. This adds 21 × 128 = 2,688 elements.

**Total: about 47,616 elements per token**, which is 53.7 times smaller than conventional attention. At 1 byte per element (FP8) that is about 47.6 KB per token. The exact byte count depends on the serving engine; some engines store scaling factors or keep the positional key in BF16, which increases the size somewhat. Treat the figures below as lower bounds.

### 3.4 KV Cache Budget and Concurrent Requests

```
vLLM memory budget = 1,440 × 0.92   = 1,324.8 GB
Weights                             -   755.6 GB
Activations (chunked prefill)       -    40.0 GB
KV cache budget                     =   529.2 GB
```

| Context length per request | KV cache per request | Requests that fit at once |
|---|---|---|
| 50K tokens | 2.38 GB | 222 |
| **100K tokens** | **4.76 GB** | **111** |
| 128K tokens (131,072) | 6.24 GB | 84 |
| 200K tokens | 9.52 GB | 55 |
| 500K tokens | 23.81 GB | 22 |
| 1M tokens (1,048,576) | 49.93 GB | 10 |

At typical coding-agent context sizes (30K–100K tokens; section 4.2), one node can hold more than 100 requests in memory at the same time. Memory is therefore not the first limit; generation speed is (section 7).

### 3.5 vLLM Serving Configuration

```bash
vllm serve zai-org/GLM-5.2-FP8 \
  --tensor-parallel-size 8 \
  --kv-cache-dtype fp8 \
  --max-model-len 1048576 \
  --gpu-memory-utilization 0.92 \
  --max-num-batched-tokens 8192 \
  --enable-chunked-prefill \
  --enable-prefix-caching \
  --trust-remote-code \
  --port 8000
```

| Flag | Why |
|---|---|
| `--tensor-parallel-size 8` | Splits the model across all 8 GPUs over NVLink |
| `--kv-cache-dtype fp8` | Halves KV cache size compared with BF16 |
| `--max-num-batched-tokens 8192` | Processes long prompts in 8K-token chunks so activation memory stays bounded |
| `--enable-prefix-caching` | Reuses the KV cache for the repeated conversation prefix in each agent turn. **Essential**: without it, every turn reprocesses the whole conversation (section 7.2) |
| `--trust-remote-code` | Needed for the `GlmMoeDsaForCausalLM` architecture |

The concurrency limit (`--max-num-seqs`) should be set from a load test, not from the 1M-token worst case. v1 set it to 11, which would have limited the node to 11 simultaneous requests.

[SGLang](https://docs.sglang.ai/) is an alternative with radix-tree prefix caching and expert-parallel modes.

---

## 4. Coding Agent Workload Baseline

### 4.1 Why Coding Agents Consume So Many Input Tokens

Every tool call (read a file, run a test, search the repo) sends the whole conversation so far back to the model. Input therefore grows with every turn while each output (a patch, a short explanation) stays small. [Vantage](https://www.vantage.sh/blog/agentic-coding-costs) illustrates this with an example in which turn 1 sends about 5,000 input tokens and turn 30 carries 25,000–35,000 tokens of accumulated context.

Supporting observations:

- [LeanOps](https://leanopstech.com/blog/agentic-ai-cost-runaway-token-budget-2026/) reports that in its audit of 30 engineering teams (March–May 2026), "re-sent context is 62% of the bill". This is a vendor's own audit and its methodology is not published.
- [Jia.je](https://jia.je/kb/en/software/coding_plan.html) reports that in vibe-coding tests, input tokens were 99.5% of all tokens and cached tokens were about 90–95% of input plus output. The tool, model and sample size are not stated.
- An [academic study of OpenHands on SWE-bench](https://arxiv.org/abs/2604.22750) found that "input tokens dominate overall consumption and cost, even with token caching", and that token usage for the same task can vary by up to 10 times between runs.

### 4.2 Measured Request Sizes (Our Own Benchmark)

On 2026-06-29 we ran GLM-5.2-FP8 on a GEAP dedicated endpoint (`asia-southeast1`) through the OpenCode agent, one request at a time, on three small tasks (12 runs, 4–11 model steps each). The node's machine type was not recorded. Results:

| Measurement | Value |
|---|---|
| Input tokens of the first request in every run | 30,200–30,400 (OpenCode system prompt and tool definitions alone) |
| Input tokens of later requests in these short runs | grew to 33,000–70,000 |
| Output tokens per step | mostly 30–600; the final report-writing step 1,700–32,000 |
| Single-stream generation speed | **95–101 tokens/s**, stable on all long outputs |
| Prompt caching on the endpoint | Not enabled, according to the benchmark's README |

**Consequence for the baseline:** a request cannot be smaller than about 30K input tokens with this agent, and requests in long sessions grow well beyond that. v1 assumed both "50 turns ≈ 1M input tokens" (about 20K per request, below the measured floor) and "100K per request on average"; these two assumptions contradicted each other. This revision uses 30K–100K per request as the range.

### 4.3 Per-Developer Monthly Baseline

We keep v1's Heavy User baseline because it is a planning assumption, not a measurement, and the conclusions in section 9 do not depend on it for the 50–500 range:

| Parameter | Value | Basis |
|---|---|---|
| Monthly input tokens | 100,000,000 (100M) | Projection: about 5M per working day × 20 days |
| Monthly output tokens | 4,000,000 (4M) | Projection: input-to-output ratio about 25:1 ([Vantage](https://www.vantage.sh/blog/agentic-coding-costs) example) |
| Requests per month | about 1,000–3,300 | 100M ÷ 30K–100K tokens per request |

**Cross-check against published spending figures:**

- Anthropic's [Claude Code cost documentation](https://code.claude.com/docs/en/costs) states that "across enterprise deployments, the average cost is around 13 USD per developer per active day and 150–250 USD per developer per month". With prompt caching at typical rates, 13 USD per day corresponds to very roughly 6.5M–23M input tokens per day on Claude Sonnet/Opus pricing. Our baseline of 5M per day is therefore a moderate, not an extreme, assumption. (v1 derived "4M tokens per day" from 13 USD by ignoring caching; that inference was incorrect.)
- LeanOps reports a median of 480 USD per developer per month, a 90th percentile of 1,650 USD and a 99th percentile above 4,200 USD per month. (v1 attributed similar numbers to Vantage and described 4,000 USD as a single-day spike; both were incorrect.)

### 4.4 Load per Developer

For capacity planning, what matters is how many tokens per second each developer generates and how much new input must be processed.

- **Output:** 4M tokens over 20 working days × 8 hours averages **6.9 tokens/s per developer** during working hours. With a peak factor of 2, peak demand is about **14 tokens/s per developer**.
- **New input that must be processed (prefill):** with prefix caching at 85%, about 15M of the 100M monthly input tokens are new. Without prefix caching, all 100M must be reprocessed.

The peak factor of 2 is an assumption; real peaks depend on time zones and working patterns.

---

## 5. Z.ai SaaS Cost Analysis

### 5.1 API Metered Pricing

| Component | Price per 1M tokens |
|---|---|
| Input (not cached) | 1.40 USD |
| Cached input | 0.26 USD |
| Cached input storage | Free (limited time) |
| Output | 4.40 USD |

Caching is [implicit and automatic](https://docs.z.ai/guides/capabilities/cache); no cache-control header is needed.

**Per-developer monthly cost (100M input, 4M output):**

`Total = 157.60 − 114 × r`, where r is the share of input tokens served from cache.

| Cache-hit rate | Uncached input | Cached input | Output | **Total per developer per month** |
|---|---|---|---|---|
| 0% | 140.00 USD | 0.00 USD | 17.60 USD | **157.60 USD** |
| 70% | 42.00 USD | 18.20 USD | 17.60 USD | **77.80 USD** |
| 80% | 28.00 USD | 20.80 USD | 17.60 USD | **66.40 USD** |
| **85% (central estimate)** | **21.00 USD** | **22.10 USD** | **17.60 USD** | **60.70 USD** |
| 90% | 14.00 USD | 23.40 USD | 17.60 USD | **55.00 USD** |
| 95% | 7.00 USD | 24.70 USD | 17.60 USD | **49.30 USD** |

The 85% central estimate is slightly below the 90–95% reported by Jia.je, to allow for sessions that start fresh.

### 5.2 GLM Coding Plan Tiers (as of 2026-07-01)

Each developer needs their own subscription; [account sharing or multi-user access is prohibited](https://docs.z.ai/devpack/usage-policy).

| Plan | Monthly | Annual billing (30% off) | Prompts per 5-hour window | Prompts per week | Z.ai's recommended use |
|---|---|---|---|---|---|
| Lite | 18 USD | 12.60 USD | about 80 | about 400 | A single project at a time |
| Pro | 72 USD | 50.40 USD | about 400 | about 2,000 | 1–2 projects simultaneously |
| Max | 160 USD | 112 USD | about 1,600 | about 8,000 | 2 or more projects simultaneously |

Quota figures and recommended use come from the [Coding Plan overview](https://docs.z.ai/devpack/overview) (2026-07-01 snapshot), which also states that "each prompt is estimated to invoke the model 15–20 times". Prices are from third-party sources (section 2.3); one source describes the 30% annual discount as an introductory offer through September 2026.

### 5.3 GLM-5.2 Quota Multiplier (as of 2026-07-01)

GLM-5.2 usage was deducted at 3× during peak hours (14:00–18:00, UTC+8) and 2× off-peak; a promotion through the end of September 2026 reduced off-peak to 1×. For a developer in UTC+8 working 9:00–18:00 (5 hours off-peak, 4 hours peak), the weighted multiplier is (5 × 2 + 4 × 3) ÷ 9 = **2.44×** without the promotion.

| Plan | Weekly cap | Effective GLM-5.2 prompts per week (÷ 2.44) | Effective prompts per 5-hour window |
|---|---|---|---|
| Lite | 400 | 164 | about 33 |
| Pro | 2,000 | 820 | about 164 |
| Max | 8,000 | 3,279 | about 656 |

**Heavy User demand:** if one prompt drives about 350K input tokens (15–20 model calls), 100M input tokens per month is about 286 prompts per month, or 58–77 per week. Lite covers this about 2×, Pro about 10× and Max about 40×.

**No overage billing:** when the quota is used up, the developer waits for the next 5-hour cycle and "the system will not deduct from your account balance" ([FAQ](https://docs.z.ai/devpack/faq)).

**No spillover to the API:** the Coding Plan uses its own endpoints, and "API calls outside the plan are not available". A team that wants a fallback must fund a separate API account.

> Section 11 describes how the plan structure changed after July 2026 (credits instead of prompts, GLM-5.3 instead of GLM-5.2). The multiplier calculation above does not apply to the current plan.

### 5.4 Team Cost Matrix (USD per month)

| Option | Per developer | 50 devs | 100 devs | 300 devs | 500 devs |
|---|---|---|---|---|---|
| Coding Plan Lite | 18.00 | 900 | 1,800 | 5,400 | 9,000 |
| API metered, 85% cache | 60.70 | 3,035 | 6,070 | 18,210 | 30,350 |
| **Coding Plan Pro** | **72.00** | **3,600** | **7,200** | **21,600** | **36,000** |
| API metered, no cache | 157.60 | 7,880 | 15,760 | 47,280 | 78,800 |
| Coding Plan Max | 160.00 | 8,000 | 16,000 | 48,000 | 80,000 |

With annual billing (30% off), Pro costs 50.40 USD per developer: 2,520 / 5,040 / 15,120 / 25,200 USD per month for 50 / 100 / 300 / 500 developers.

### 5.5 Which Tier to Choose

Lite covers the Heavy User token volume, but Z.ai recommends it only for "a single project at a time". Developers who run parallel agents (Claude Code subagents, OpenCode `task()` calls, Cline parallel mode) work on several things at once, so **Pro is the smallest tier we consider safe for agentic teams**. Max is for developers who regularly run two or more projects simultaneously.

---

## 6. Self-Hosted TCO (GCP)

### 6.1 Published A4 Prices (us-central1, as of July 2026)

| Price | USD per hour | Source |
|---|---|---|
| GEAP node price, `a4-highgpu-8g` (prediction and custom training) | 148.212 | [GEAP pricing](https://cloud.google.com/products/gemini-enterprise-agent-platform/pricing), 2026-06-30 snapshot |
| Compute Engine on-demand | N/A (not offered) | [Accelerator-optimized pricing](https://cloud.google.com/products/compute/pricing/accelerator-optimized), 2026-07-13 snapshot |
| Compute Engine DWS Flex-start (at most 7 days) | 64.44 | same |
| Compute Engine calendar-mode reservation | 90.22 | same |
| Compute Engine Spot | 37.7464 | same |
| Compute Engine 1-year resource-based CUD | 88.9272 | same |
| Compute Engine 3-year resource-based CUD | 56.7072 | same |

Monthly figures use 730 hours, as Google's pricing pages do.

### 6.2 GEAP Management Fee (Estimate)

When a GEAP node runs on your own reserved, Spot or Flex-start capacity, you pay the Compute Engine price for the machine plus an Agent Platform management fee. The management fee does not receive CUDs. Google does not publish the fee for A4.

**Our estimate: about 19.33 USD/hr.** The reasoning:

1. For H100 machines the published management fee is exactly 15% of the machine price (1.4694826 ÷ 9.79655 USD).
2. If the same 15% applies to A4, the GEAP price of 148.212 USD/hr consists of a machine price of 148.212 ÷ 1.15 = 128.88 USD/hr plus a fee of 19.33 USD/hr.
3. A machine price of 128.88 USD/hr is consistent with the published A4 discounts: the 1-year CUD (88.9272) is exactly 31% off it, the 3-year CUD (56.7072) 56% off and Flex-start (64.44) 50% off.

This is an inference, not a published figure. The exact value can be read from the Cloud Billing Catalog API. A 50% error in the fee (±9.67 USD/hr) changes the monthly node cost by about ±7,000 USD and does not change any conclusion in section 9.

### 6.3 Monthly TCO per Node

Common overhead items (per node per month):

| Item | Monthly cost | Basis |
|---|---|---|
| Boot disk: Hyperdisk Balanced, 500 GiB | 40.00 USD | 0.08 USD per GiB-month ([disk pricing](https://cloud.google.com/compute/disks-image-pricing)) |
| Checkpoint in Cloud Storage Standard, 755.6 GB | 14.07 USD | 0.020 USD per GiB-month |
| Cloud Monitoring (estimate) | 25.00 USD | |
| Internet egress for 500 developers | 2.56 USD | 4M output tokens × 4 bytes × 3 (HTTP overhead) per developer; 0.12 USD per GiB after the first free GiB |
| Cloud Logging | 0 USD | Within 50 GiB free per project |
| Operations staff, GEAP | 7,500 USD | Estimate: 0.5 FTE at 180,000 USD per year |
| Operations staff, GKE/Compute Engine without GEAP | 15,000 USD | Estimate: 1.0 FTE, because the team also runs the endpoint, scaling and monitoring |

TCO per node:

| Option | Machine | Management fee | Operations and other | **Total per month** |
|---|---|---|---|---|
| GEAP, on-demand node price | 108,195 USD (incl. fee) | — | 7,582 USD | **115,776 USD** |
| GEAP + 1-year CUD reservation | 64,917 USD | 14,112 USD | 7,582 USD | **86,611 USD** |
| GEAP + 3-year CUD reservation | 41,396 USD | 14,112 USD | 7,582 USD | **63,090 USD** |
| GEAP + Spot | 27,555 USD | 14,112 USD | 7,582 USD | **49,249 USD** |
| GKE/GCE + 1-year CUD | 64,917 USD | 0 | 15,082 USD | **79,998 USD** |
| GKE/GCE + 3-year CUD | 41,396 USD | 0 | 15,082 USD | **56,478 USD** |

**Notes on each option:**

- **On-demand GEAP** has a published price, but we could not confirm from documentation whether A4 GEAP nodes can actually be obtained on demand, given that A4 Compute Engine capacity cannot.
- **Spot** capacity can be reclaimed by Google at short notice. It can supplement a reserved node but cannot be the only node for a production service.
- **Flex-start** (64.44 USD/hr) is limited to 7 days and suits benchmarks, not permanent serving.
- **CUDs** lock in 1 or 3 years of payment. GLM-5.2 was superseded by GLM-5.3 within five months of release (section 11). The hardware commitment survives a model change, but the risk of committing to a configuration that no longer matches the workload should be weighed.
- **Redundancy:** every figure above is for one node. A production service normally needs at least two nodes, or one node plus a SaaS fallback.

---

## 7. Node Capacity and Breakeven

### 7.1 Why v1's Capacity Figure Was Wrong

v1 computed "8 TBps ÷ 5 GB of active weights per GPU = 1,600 tokens/s" and treated 60% of that (960 tokens/s) as the throughput of the whole node, giving about 150 developers per node.

The 1,600 tokens/s figure is the upper bound for **one request at a time**: each generated token requires reading the active weights once. A serving engine, however, groups many requests into one batch and generates one token for every request in the batch in each step, reading the weights only once per step. Node throughput therefore grows with batch size until other limits (reading the KV cache, compute, expert-routing overhead) take over. Using the single-request bound as the node's capacity underestimates it by a large factor.

Our benchmark shows the gap between theory and practice for a single stream: the measured speed was **about 100 tokens/s**, roughly 6% of the theoretical 1,600 tokens/s. Had v1's logic been applied to the measured speed, one node would serve only about 15 developers. Neither number describes a batched node.

### 7.2 What Limits a Batched Node

- **Generation (decode).** With large batches, nearly all 256 experts are touched in every step, so each step reads close to the full 755.6 GB of weights. At 8 × 8 TBps that takes about 12 ms in theory. With 64–128 requests per batch this gives a theoretical ceiling of roughly 5,000–11,000 tokens/s. Real engines reach only part of a theoretical ceiling; how large a part on B200 with GLM-5.2 is not known without measurement.
- **Prompt processing (prefill).** Processing one input token costs about 80 GFLOP (2 × 40B active parameters), plus attention. Without prefix caching, 100M input tokens per developer per month average about 14 TFLOP/s per developer during working hours. With 85% prefix caching this falls to about 2 TFLOP/s. Eight B200 GPUs provide roughly 36 PFLOP/s of dense FP8 compute, and perhaps 40% of that in practice. **Prefix caching is therefore essential**: without it, prefill alone limits a node to a few hundred developers.
- **KV cache memory.** About 111 requests of 100K tokens fit at once (section 3.4), which is not the first limit for a few hundred developers.

### 7.3 Capacity Estimate (Unverified)

Combining a plausible real node throughput of about 1,500–6,000 tokens/s with a per-developer demand of 7–14 tokens/s (section 4.4) gives:

**Estimated capacity: about 100–900 developers per node.**

This range is wide because neither the real batched throughput nor the real peak demand has been measured. It should be replaced by a load test before any decision above 500 developers (section 10.4).

### 7.4 Breakeven by Per-Node Capacity

Breakeven capacity = node cost per month ÷ SaaS price per developer per month. A self-hosted node is cheaper than the SaaS option only if it can serve **more** developers than this number.

| Self-hosted option (per node per month) | Lite (18 USD) | API 85% cache (60.70 USD) | Pro (72 USD) | API no cache (157.60 USD) | Max (160 USD) |
|---|---|---|---|---|---|
| GEAP on-demand (115,776 USD) | 6,432 | 1,907 | 1,608 | 735 | 724 |
| GEAP + 1-year CUD (86,611 USD) | 4,812 | 1,427 | 1,203 | 550 | 541 |
| GEAP + 3-year CUD (63,090 USD) | 3,505 | 1,039 | 876 | 400 | 394 |
| GKE/GCE + 3-year CUD (56,478 USD) | 3,138 | 930 | 784 | 358 | 353 |
| GEAP + Spot (49,249 USD) | 2,736 | 811 | 684 | 312 | 308 |

Reading this table against the estimated capacity of 100–900 developers per node:

- **Against Lite**, no self-hosted option can break even.
- **Against Pro**, breakeven requires 784–876 developers per node with a 3-year commitment. That is at the upper edge of the estimate.
- **Against Max**, breakeven requires 353–394 developers per node with a 3-year commitment. That is inside the estimate, so self-hosting may well be cheaper than Max at large scale.

### 7.5 Per-Developer Cost at Different Capacities

| Developers per node | GEAP on-demand | GEAP + 3-year CUD | GKE/GCE + 3-year CUD |
|---|---|---|---|
| 150 (v1's assumption) | 772 USD | 421 USD | 377 USD |
| 300 | 386 USD | 210 USD | 188 USD |
| 500 | 232 USD | 126 USD | 113 USD |
| 800 | 145 USD | 79 USD | 71 USD |
| 1,200 | 96 USD | 53 USD | 47 USD |

Pro costs 72 USD per developer and Max 160 USD. The table shows that a 3-year CUD node beats Max from about 400 developers per node and approaches Pro only around 800–900.

---

## 8. Scenario Comparison Matrix

### 8.1 Monthly Cost by Team Size

Self-hosted figures below are the cost of **one node**, which is the minimum for any team. If the node's real capacity is below the team size, more nodes are needed and the cost rises proportionally. A second node for redundancy doubles the figure.

| Option | 50 devs | 100 devs | 300 devs | 500 devs |
|---|---|---|---|---|
| Z.ai Lite | 900 | 1,800 | 5,400 | 9,000 |
| Z.ai API, 85% cache | 3,035 | 6,070 | 18,210 | 30,350 |
| **Z.ai Pro** | **3,600** | **7,200** | **21,600** | **36,000** |
| Z.ai Max | 8,000 | 16,000 | 48,000 | 80,000 |
| Self-hosted, GEAP + Spot (1 node minimum) | 49,249 | 49,249 | 49,249 | 49,249 |
| Self-hosted, GKE/GCE + 3-year CUD (1 node minimum) | 56,478 | 56,478 | 56,478 | 56,478 |
| Self-hosted, GEAP + 3-year CUD (1 node minimum) | 63,090 | 63,090 | 63,090 | 63,090 |
| Self-hosted, GEAP on-demand (1 node minimum) | 115,776 | 115,776 | 115,776 | 115,776 |

### 8.2 Self-Hosted (GEAP + 3-year CUD, one node) vs Pro

| Team size | Z.ai Pro | Self-hosted, one node | Self-hosted ÷ Pro |
|---|---|---|---|
| 50 devs | 3,600 USD | 63,090 USD | 17.5× |
| 100 devs | 7,200 USD | 63,090 USD | 8.8× |
| 300 devs | 21,600 USD | 63,090 USD | 2.9× |
| 500 devs | 36,000 USD | 63,090 USD | 1.8× |

These ratios are lower bounds: they assume one node is enough and ignore redundancy.

### 8.3 Recommendation per Team Size

| Team size | Recommended | Monthly cost | Reason |
|---|---|---|---|
| 50 devs | Z.ai Pro | 3,600 USD | One self-hosted node costs at least 13.7× more (Spot) |
| 100 devs | Z.ai Pro | 7,200 USD | At least 6.8× more |
| 300 devs | Z.ai Pro | 21,600 USD | At least 2.3× more |
| 500 devs | Z.ai Pro | 36,000 USD | At least 1.4× more (Spot); 1.6× more with GKE/GCE 3-year CUD |

If developers work on one project at a time without parallel agents, Lite (900–9,000 USD per month for 50–500 developers) is the cheapest option.

---

## 9. Final Recommendation

### 9.1 For 50–500 Developers: Z.ai Coding Plan Pro

Pro (72 USD per developer per month) is cheaper than one self-hosted node for every team size in scope, and a production self-hosted deployment needs more than one node. This conclusion does not depend on per-node capacity, the management-fee estimate, or the throughput estimate. It does depend on Z.ai's plan prices (section 2.3) and on the plan's quotas being sufficient for the team (section 5.3).

Pro also provides:

- About 5× the quota of Lite and Z.ai's recommendation for 1–2 parallel projects.
- No overage billing; cost per developer is capped at the subscription price.
- No commitment beyond the billing period, which matters because the model generation changed within five months (section 11).

Lite (18 USD) is sufficient for single-project workflows without parallel agents. Max (160 USD) is for developers who regularly exhaust Pro's quota.

### 9.2 Annual Billing

| Plan | Monthly billing | Annual billing | Savings per year for 100 developers |
|---|---|---|---|
| Lite | 18 USD | 12.60 USD | 6,480 USD |
| Pro | 72 USD | 50.40 USD | 25,920 USD |
| Max | 160 USD | 112 USD | 57,600 USD |

### 9.3 When to Evaluate Self-Hosting

Self-hosting is worth a measured evaluation in these situations:

1. **The organization is well above 500 developers** and a load test shows that one node can serve more than about 800 developers (breakeven against Pro with a 3-year commitment) or more than about 400 (breakeven against Max).
2. **Code may not leave the organization's cloud project** for regulatory or contractual reasons. A GEAP or GKE deployment runs inside your project and VPC.
3. **A fine-tuned model is required.** Z.ai's SaaS serves only its own models.
4. **Very long contexts dominate the workload** and exhaust Coding Plan quotas.

### 9.4 Decision Flowchart

```
Must code stay inside your own cloud project, or do you need a fine-tuned model?
  YES → Self-host (GEAP or GKE); size it with a load test
  NO  ↓

Are there more than ~800 developers using agents heavily?
  YES → Run a concurrent-load benchmark; self-host only if one node
        serves more developers than the breakeven in section 7.4
  NO  ↓

Do developers run parallel agents or several projects at once?
  YES → Z.ai Pro (72 USD/dev/month)
  NO  → Z.ai Lite (18 USD/dev/month)
```

---

## 10. Assumptions & Limitations

### 10.1 Verified Facts

Checked against primary sources on 2026-11-12; July 2026 values were taken from Wayback Machine snapshots.

| Fact | Source |
|---|---|
| GEAP node price for `a4-highgpu-8g`: 148.212 USD/hr | [GEAP pricing](https://cloud.google.com/products/gemini-enterprise-agent-platform/pricing) (2026-06-30 snapshot) |
| A4 Compute Engine: on-demand N/A; Flex-start 64.44; calendar 90.22; Spot 37.7464; 1-year CUD 88.9272; 3-year CUD 56.7072 USD/hr | [Accelerator-optimized pricing](https://cloud.google.com/products/compute/pricing/accelerator-optimized) (2026-07-13 snapshot) |
| A4 supports Spot, Flex-start, calendar reservations, capacity blocks and resource-based CUDs; no on-demand | [Accelerator-optimized machines](https://docs.cloud.google.com/compute/docs/accelerator-optimized-machines) |
| A4 has no sustained use discounts or flexible CUDs | Accelerator-optimized pricing |
| A4 supports Hyperdisk only | Accelerator-optimized machines |
| GEAP on own capacity: Compute Engine SKU with CUDs plus management fee SKU without CUDs | [Use reservations with online inference](https://docs.cloud.google.com/gemini-enterprise-agent-platform/machine-learning/predictions/use-reservations) |
| `a4-highgpu-8g` hardware specifications | [GPU machine types](https://docs.cloud.google.com/compute/docs/gpus) |
| GLM-5.2: 1M context, 256 routed experts / 8 active / 1 shared, MLA + DSA parameters | [`config.json`](https://huggingface.co/zai-org/GLM-5.2/raw/main/config.json) |
| GLM-5.2: 753B total / 40B active | [NVIDIA GLM-5.2-NVFP4 card](https://huggingface.co/nvidia/GLM-5.2-NVFP4) |
| FP8 checkpoint 755.6 GB; BF16 1,506.7 GB | [Hugging Face API](https://huggingface.co/api/models/zai-org/GLM-5.2-FP8?blobs=true) |
| Z.ai API prices 1.40 / 0.26 / 4.40 USD per million tokens | [Z.ai pricing](https://docs.z.ai/guides/overview/pricing) (2026-06-29 snapshot) |
| Coding Plan quotas, multiplier, recommended use, no overage, single-subscriber use, separate endpoints | [Overview](https://docs.z.ai/devpack/overview), [FAQ](https://docs.z.ai/devpack/faq), [usage policy](https://docs.z.ai/devpack/usage-policy) (2026-06-29 to 07-01 snapshots) |
| Single-stream GLM-5.2-FP8 speed about 100 tokens/s; requests start at about 30K input tokens in OpenCode | Our benchmark, 2026-06-29 (section 4.2) |

### 10.2 Labeled Estimates

| Estimate | Value | Effect on the conclusion |
|---|---|---|
| Coding Plan Pro/Max prices | 72 / 160 USD | From third-party sources only. If Pro were 30% more expensive, it would still be cheaper than one node at 500 developers |
| GEAP management fee for A4 | about 19.33 USD/hr | ±50% changes node cost by about ±7,000 USD per month; no conclusion changes |
| Per-node capacity | about 100–900 developers | Determines the answer above 500 developers; does not affect the 50–500 recommendation |
| Peak demand per developer | 7–14 output tokens/s | Same as above |
| Operations staff | 0.5 FTE (GEAP), 1.0 FTE (GKE/GCE) | ±3,000 USD per month; no conclusion changes |
| Cache-hit rate for API metered | 85% | At 70%, API metered costs 77.80 USD, slightly more than Pro |

### 10.3 Sensitivity: What Would Make Self-Hosting Win for 500 Developers?

For one node to beat Pro for 500 developers (36,000 USD per month), the node's total monthly cost must fall below 36,000 USD. The cheapest reliable option today costs 56,478 USD (GKE/GCE, 3-year CUD). That would require either about a 50% cut in the 3-year A4 price, or Z.ai raising Pro above about 113 USD (56,478 ÷ 500) — in both cases with one node serving all 500 developers.

### 10.4 Open Item: Concurrent-Load Benchmark

To decide for organizations above 500 developers, run a load test such as `vllm bench serve` on one `a4-highgpu-8g` with GLM-5.2-FP8 (or its successor), prefix caching enabled, request sizes of 30K–100K input tokens, and 8 to 256 concurrent requests. The result should be the highest output token rate at which latency stays acceptable. Flex-start capacity (64.44 USD/hr, up to 7 days) is suitable for this.

### 10.5 Out of Scope

- Regions other than us-central1 (our benchmark ran in asia-southeast1).
- Fine-tuning costs.
- Latency targets; neither SaaS latency nor self-hosted latency under load was measured.
- The GEAP Flexible Savings Plan, which did not exist on the pricing basis date (section 11).

---

## 11. Changes After the Pricing Basis Date

The following changes happened after 2026-07-01. They are not errors in this analysis but limit how directly its numbers apply today (checked 2026-11-12):

- **Coding Plan quotas are now counted in credits, not prompts.** The current overview lists 5-hour and weekly credit limits (Lite 2,000 / 10,000, Pro 12,000 / 60,000, Max 28,000 / 140,000). Third-party sources date the change to about 2026-07-30.
- **GLM-5.2 is no longer served on the Coding Plan.** "Requests for GLM-5.2/GLM-5.1 will be automatically routed to GLM-5.3." The Hugging Face repository for GLM-5.2 now points to `zai-org/GLM-5.3-BF16` as the newer version. GLM-5.3 is listed at the same API price as GLM-5.2.
- **Plan prices:** third-party sources from October–November 2026 list Pro at 80 USD and Max at 168 USD (Lite unchanged at 18 USD). We could not confirm this from Z.ai's own pages.
- **GEAP Flexible Savings Plan:** the GEAP pricing page now shows 1-year (10% off) and 3-year (20% off) savings plan prices for GEAP nodes, for example 118.5696 USD/hr for `a4-highgpu-8g` with a 3-year plan.
- **A4 Spot price** rose from 37.7464 USD/hr (July) to 43.5968 USD/hr.

None of these changes reverses the 50–500 developer recommendation: with Pro at 80 USD, 500 developers cost 40,000 USD per month, which is still below the cheapest self-hosted node.

---

## 12. References

### GCP

| Source | URL |
|---|---|
| GEAP pricing (node prices, management fee SKUs) | https://cloud.google.com/products/gemini-enterprise-agent-platform/pricing |
| Compute Engine accelerator-optimized pricing (A4 consumption options) | https://cloud.google.com/products/compute/pricing/accelerator-optimized |
| Accelerator-optimized machines (A4 consumption options, Hyperdisk only) | https://docs.cloud.google.com/compute/docs/accelerator-optimized-machines |
| GPU machine types (`a4-highgpu-8g` specifications) | https://docs.cloud.google.com/compute/docs/gpus |
| AI Hypercomputer consumption options | https://docs.cloud.google.com/ai-hypercomputer/docs/consumption-models |
| GEAP Self-Deployed Models | https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/model-garden/self-deployed-models |
| GEAP: use reservations with online inference (CUDs and management fee) | https://docs.cloud.google.com/gemini-enterprise-agent-platform/machine-learning/predictions/use-reservations |
| GEAP: Spot VMs and Flex-start VMs for inference | https://docs.cloud.google.com/gemini-enterprise-agent-platform/machine-learning/predictions/use-spot-vms |
| Disk pricing (Hyperdisk Balanced) | https://cloud.google.com/compute/disks-image-pricing |
| Cloud Storage pricing | https://cloud.google.com/storage/pricing |
| Network pricing (internet egress) | https://cloud.google.com/vpc/network-pricing |
| Cloud Observability pricing | https://cloud.google.com/products/observability/pricing |

### GLM-5.2 and Serving

| Source | URL |
|---|---|
| GLM-5.2 model card and `config.json` | https://huggingface.co/zai-org/GLM-5.2 |
| GLM-5.2-FP8 model card | https://huggingface.co/zai-org/GLM-5.2-FP8 |
| NVIDIA GLM-5.2-NVFP4 card (753B / 40B) | https://huggingface.co/nvidia/GLM-5.2-NVFP4 |
| DeepSeek-V3 paper (MLA) | https://arxiv.org/abs/2412.19437 |
| DeepSeek-V3 inference code (MLA cache layout) | https://github.com/deepseek-ai/DeepSeek-V3/blob/main/inference/model.py |
| vLLM documentation | https://docs.vllm.ai/en/stable/ |
| SGLang documentation | https://docs.sglang.ai/ |

### Z.ai

| Source | URL |
|---|---|
| API pricing | https://docs.z.ai/guides/overview/pricing |
| Context caching | https://docs.z.ai/guides/capabilities/cache |
| Coding Plan overview (quotas, multiplier) | https://docs.z.ai/devpack/overview |
| Coding Plan FAQ (no overage, no API spillover) | https://docs.z.ai/devpack/faq |
| Coding Plan usage policy (single subscriber, recommended use) | https://docs.z.ai/devpack/usage-policy |
| Coding Plan quick start (endpoints) | https://docs.z.ai/devpack/quick-start |
| HyScaler GLM Coding Plan review (prices) | https://hyscaler.com/insights/glm-coding-plan-review/ |
| AI Pricing Guru, GLM Coding Pro (prices) | https://www.aipricing.guru/subscriptions/zai-glm-coding-pro/ |

### Token Consumption

| Source | URL |
|---|---|
| Vantage — agentic coding costs (1M input / 40K output example; turn-30 context) | https://www.vantage.sh/blog/agentic-coding-costs |
| LeanOps — audit of 30 teams (62% re-sent context; spending percentiles) | https://leanopstech.com/blog/agentic-ai-cost-runaway-token-budget-2026/ |
| Jia.je — coding plan analysis (input share, cache share) | https://jia.je/kb/en/software/coding_plan.html |
| Anthropic — Claude Code costs (13 USD per active day) | https://code.claude.com/docs/en/costs |
| "How do coding agents spend your money?" (OpenHands / SWE-bench study) | https://arxiv.org/abs/2604.22750 |

---

## Revision History

**v2 (2026-11-12).** Fact-check against primary sources and our own benchmark data. Main corrections:

- Replaced the "GEAP management fee of 83.772 USD/hr", which came from subtracting the A4 Flex-start price from the GEAP node price, with an estimate of about 19.33 USD/hr and added the published A4 Spot and CUD prices.
- Removed the claim that self-hosting is "mathematically impossible at any team size" and the iterative breakeven calculation built on it; replaced them with breakeven as a function of per-node capacity.
- Replaced the node throughput of ≈960 tokens/s (a single-request bound used as node capacity) with an explanation of batched throughput and an unverified capacity range.
- Added GKE/Compute Engine without GEAP as a self-hosted option.
- Corrected the FP8 checkpoint size (744 → 755.6 GB), the key head dimension (192 → 256) and the KV cache size (added the DSA indexer).
- Made request-size assumptions consistent with measured values and corrected citations: 62% re-sent context (LeanOps, not WebProNews), spending percentiles (LeanOps, not Vantage; monthly, not daily), 13 USD per active day (Anthropic documentation, without the uncached token inference).
- Added section 11 on changes after the pricing basis date.

**v1 (2026-07-01).** Initial report.
