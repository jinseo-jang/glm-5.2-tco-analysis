[English](README.md) | **한국어**

# GLM-5.2 FP8 자체 호스팅 vs Z.ai SaaS — TCO 최종 분석 보고서

**작성일:** 2026-07-01  
**분석 대상:** GCP GEAP `a4-highgpu-8g` 자체 호스팅 vs Z.ai SaaS (API + Coding Plans)  
**분석 범위:** AI 코딩 에이전트 워크로드, 개발자 50/100/300/500명 시나리오  
**언어:** 한국어 (기술 용어는 영어 병기)  
**검토 상태:** 독립 검토자 4명 승인 완료 (계획 준수, 수학 품질, 수동 QA, 범위 충실도)

---

> **TL;DR**
>
> GLM-5.2 FP8을 GCP GEAP에 자체 호스팅하면 월 $90,853(3년 CUD 기준)이 고정 지출된다. Z.ai Pro는 개발자 1인당 $72/월로 선형 확장된다. 수학적으로 손익분기점은 어떤 팀 규모에서도 도달 불가능하다. 자체 호스팅의 개발자 1인당 비용 하한선은 $606/월로, Z.ai Max($160)보다 3.79배 이상 비싸다. 50–500명 규모의 모든 팀에서 **Z.ai SaaS(Pro 플랜)를 사용하라.**

---

## 목차

1. [요약 (Executive Summary)](#1-요약-executive-summary)
2. [배경 및 분석 범위](#2-배경-및-분석-범위)
3. [기술 아키텍처 및 타당성](#3-기술-아키텍처-및-타당성)
4. [코딩 에이전트 워크로드 베이스라인](#4-코딩-에이전트-워크로드-베이스라인)
5. [Z.ai SaaS 비용 분석](#5-zai-saas-비용-분석)
6. [GCP 자체 호스팅 TCO](#6-gcp-자체-호스팅-tco)
7. [손익분기점 및 규모의 경제](#7-손익분기점-및-규모의-경제)
8. [시나리오별 비교 매트릭스](#8-시나리오별-비교-매트릭스)
9. [최종 권고사항](#9-최종-권고사항)
10. [가정 및 한계점](#10-가정-및-한계점)
11. [참조 링크](#11-참조-링크)

---

## 1. 요약 (Executive Summary)

### 핵심 결론

**Z.ai SaaS를 사용하라. 자체 호스팅은 50–500명 규모의 어떤 팀에서도 경제적으로 타당하지 않다.**

GCP GEAP에서 GLM-5.2 FP8을 자체 호스팅하면 단일 노드 기준 월 [$115,785(온디맨드)](https://cloud.google.com/products/gemini-enterprise-agent-platform/pricing) 또는 [$90,853(3년 CUD 추정치)](https://cloud.google.com/ai-hypercomputer/docs/consumption-models)이 고정 지출된다. Z.ai Pro 플랜은 개발자 1인당 [$72/월](https://docs.z.ai/devpack/overview)로 팀 규모에 따라 선형 확장된다.

수학적 증명에 따르면 자체 호스팅의 개발자 1인당 비용 하한선은 <strong>&#36;606/월(3년 CUD, 처리량 한계 기준)</strong>이다. 이는 Z.ai Max($160/월)보다 최소 3.79배, Z.ai Pro($72/월)보다 8.4배 비싸다. 이 비율은 팀 규모와 무관하게 항상 3.79배 이상으로 유지된다.

근본 원인은 GEAP 관리 수수료($83.772/hr, 총 비용의 56.5%)가 CUD 할인 대상이 아니라는 점이다. 150명/노드 기준 관리 수수료만 $408/인/월로, Z.ai Max($160)보다 2.5배 비싸다. 컴퓨팅 비용이 0원이어도 자체 호스팅이 SaaS보다 비싸다.

### 핵심 수치 요약

| 지표 | 값 | 출처 |
|---|---|---|
| GEAP 온디맨드 가격 | [$148.212/hr](https://cloud.google.com/products/gemini-enterprise-agent-platform/pricing) | GCP 공식 |
| 월 TCO (온디맨드, 730시간) | $115,785 | 계산값 |
| 월 TCO (3년 CUD 추정치) | $90,853 | 계산값 (추정치) |
| 실효 CUD 할인율 | 21.5% (명목 53% 아님) | GEAP 관리 수수료 비할인 |
| Z.ai Pro 비용 (100명) | $7,200/월 | [$72/인 × 100](https://docs.z.ai/devpack/overview) |
| 자체 호스팅 비용 (100명) | $90,853/월 | 단일 노드 고정 비용 |
| 자체 호스팅 vs Pro 배율 (100명) | **12.6배** | $90,853 ÷ $7,200 |
| 단일 노드 처리 가능 개발자 수 | ≈150명 (추정치) | 디코드 처리량 한계 |
| 개발자 1인당 비용 하한선 (CUD) | $606/월 | $90,853 ÷ 150 |
| 손익분기점 (단일 노드 가정) | 568명 (3년 CUD vs Max) | 수학적으로 도달 불가 |

### 팀 규모별 권고사항 요약

| 팀 규모 | 권고 옵션 | 월 비용 | 자체 호스팅 대비 절감 |
|---|---|---|---|
| 50명 | Z.ai Pro | $3,600 | $87,253 절감 (25.2배 저렴) |
| 100명 | Z.ai Pro | $7,200 | $83,653 절감 (12.6배 저렴) |
| 300명 | Z.ai Pro | $21,600 | $160,106 절감 (8.4배 저렴) |
| 500명 | Z.ai Pro | $36,000 | $327,412 절감 (10.1배 저렴) |

---

## 2. 배경 및 분석 범위

### 2.1 GLM-5.2란 무엇인가

GLM-5.2는 [Z.ai(智谱AI)](https://z.ai/blog/glm-5.2) (브라우저에서 확인 필요)가 2026년 6월 16일 공개한 대규모 언어 모델이다. MIT 라이선스로 배포되어 지역 제한 없이 상업적 사용이 가능하다.

주요 특징:

- **아키텍처:** `GlmMoeDsaForCausalLM` (MoE + DeepSeek Sparse Attention)
- **파라미터:** [총 753B, 토큰당 활성 40B (MoE)](https://huggingface.co/zai-org/GLM-5.2-FP8)
- **컨텍스트 길이:** [최대 1,048,576 토큰 (1M)](https://huggingface.co/zai-org/GLM-5.2/raw/main/config.json)
- **레이어 수:** [78개](https://huggingface.co/zai-org/GLM-5.2/raw/main/config.json)
- **MoE 구조:** [256개 라우팅 전문가](https://huggingface.co/zai-org/GLM-5.2/raw/main/config.json), 토큰당 8개 활성, 공유 전문가 1개
- **라이선스:** MIT (상업적 사용 가능, 지역 제한 없음)
- **FP8 체크포인트:** [`zai-org/GLM-5.2-FP8`](https://huggingface.co/zai-org/GLM-5.2-FP8) (F8_E4M3 형식)

GLM-5.2의 핵심 기술적 차별점은 **Multi-head Latent Attention (MLA)** 아키텍처다. DeepSeek-V3와 동일한 메커니즘으로, 표준 MHA 대비 KV 캐시를 42.7배 압축한다. 이 덕분에 단일 8×B200 노드에서 1M 토큰 컨텍스트 추론이 가능하다.

### 2.2 GCP GEAP Self-Deployed Model이란 무엇인가

GEAP(Gemini Enterprise Agent Platform)의 Self-Deployed Model은 [Vertex AI 관리형 엔드포인트](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/model-garden/self-deployed-models)에 사용자가 직접 모델 가중치를 배포하는 방식이다. 서버리스 MaaS(Model-as-a-Service)와 달리 기반 컴퓨팅 비용을 직접 부담한다.

**GEAP가 관리하는 것:**
- Vertex AI 엔드포인트 (안정적인 HTTPS DNS, TLS 종료)
- 자동 스케일링 컨트롤러 (GPU 사용률 기반)
- 컨테이너 런타임 관리 (vLLM 컨테이너 배포, 헬스 체크)
- Cloud Monitoring/Logging 자동 통합
- VPC 격리 및 네트워크 관리

**사용자가 관리하는 것:**
- 머신 타입 선택 (`a4-highgpu-8g`)
- vLLM 서빙 설정 (텐서 병렬화, KV 캐시 설정 등)
- 모델 가중치 소싱 및 스테이징
- 최소/최대 레플리카 수 설정
- 기반 컴퓨팅 인스턴스 비용 부담

### 2.3 Z.ai SaaS란 무엇인가

Z.ai는 GLM-5.2를 두 가지 방식으로 제공한다:

**API 종량제 (pay-as-you-go):**
- 엔드포인트: `https://api.z.ai/api/paas/v4`
- 토큰 단위 과금, 사용량에 따라 선형 증가
- 자동 컨텍스트 캐싱 지원 (별도 설정 불필요)

**GLM Coding Plan (구독형):**
- 월정액 구독, 5시간 롤링 윈도우 기반 프롬프트 한도
- Lite($18), Pro($72), Max($160) 3개 티어
- 초과 시 추가 과금 없음 (다음 5시간 윈도우까지 대기)
- 지원 도구: Claude Code, Cline, OpenCode, Cursor 등

### 2.4 분석 범위

이 보고서는 다음 조건을 전제로 한다:

- **워크로드:** AI 코딩 에이전트 (Claude Code, OpenCode, Cursor 등)
- **사용자 프로필:** Heavy User (개발자 1인당 월 100M 입력 / 4M 출력 토큰)
- **팀 규모:** 50명, 100명, 300명, 500명
- **배포 지역:** us-central1 (GCP 기준)
- **분석 기준일:** 2026-07-01
- **자체 호스팅 모델:** 단일 `a4-highgpu-8g` 노드 (처리량 한계에 따라 다중 노드 분석 포함)

---

## 3. 기술 아키텍처 및 타당성

### 3.1 하드웨어 사양: `a4-highgpu-8g`

[GCP 공식 문서](https://cloud.google.com/compute/docs/gpus)에서 확인된 `a4-highgpu-8g` 사양:

| 속성 | 값 |
|---|---|
| GPU 모델 | NVIDIA B200 Blackwell |
| GPU 수 | 8개 |
| GPU당 VRAM | 180 GB HBM3e |
| **총 VRAM** | **1,440 GB** |
| GPU 메모리 대역폭 | [8 TBps/GPU](https://cloud.google.com/compute/docs/gpus) |
| GPU 인터커넥트 | [NVLink Full Mesh @ 1,800 GBps](https://cloud.google.com/compute/docs/gpus) |
| vCPU | 224개 (Intel Emerald Rapids) |
| 시스템 RAM | 3,968 GB |
| 로컬 SSD | 12,000 GiB (NVMe, 인스턴스 가격에 포함) |
| 네트워크 대역폭 | [3,600 Gbps (GPUDirect RDMA)](https://cloud.google.com/compute/docs/gpus/gpu-network-bandwidth) |

**중요 제약사항:**
- [Spot VM 미지원](https://docs.cloud.google.com/ai-hypercomputer/docs/create/create-vm) (온디맨드 전용)
- [Persistent Disk 미지원](https://docs.cloud.google.com/ai-hypercomputer/docs/create/create-vm) (Hyperdisk만 사용 가능)
- [Sustained Use Discount(SUD) 미적용](https://docs.cloud.google.com/ai-hypercomputer/docs/create/create-vm)

### 3.2 GLM-5.2 FP8 가중치 메모리 계산

[공식 HuggingFace 카드](https://huggingface.co/zai-org/GLM-5.2-FP8) 및 [Z.ai GitHub README](https://github.com/zai-org/GLM-5/blob/main/README.md) 기준:

```
FP8 가중치 크기 계산:
753B 파라미터 × 1 바이트/파라미터 = 753 GB (이론값)
실제 체크포인트: ≈744 GB
  (임베딩, 레이어 노름 등 일부 텐서가 BF16/FP32로 저장되어 ≈9 GB 차이)

BF16 비교: 753B × 2 바이트 = ≈1.51 TB → 단일 노드 VRAM 초과
FP8이 단일 노드 배포 가능한 최소 양자화 수준
```

### 3.3 VRAM 예산 계산

```
총 VRAM (a4-highgpu-8g):     1,440 GB
GLM-5.2 FP8 가중치:          -  744 GB
                              ─────────
KV 캐시 + 오버헤드 가용:          696 GB
```

프로덕션 설정 (`--gpu-memory-utilization 0.92` + 활성화 버퍼 40 GB 공제):

```
vLLM 할당 (92%):  1,440 × 0.92 = 1,324.8 GB
가중치 공제:                    -   744.0 GB
활성화 버퍼 공제:               -    40.0 GB
                               ─────────────
KV 캐시 가용 예산:                  540.8 GB
```

GPU당 분배 (TP=8):

| GPU당 지표 | 값 |
|---|---|
| GPU당 VRAM | 180 GB |
| 가중치 샤드 (744 GB ÷ 8) | 93 GB |
| KV 캐시 가용 (프로덕션) | 67.6 GB |

### 3.4 MLA KV 캐시: 42.7배 압축의 수학

GLM-5.2는 [DeepSeek-V3 논문](https://arxiv.org/abs/2412.19437)에서 제안된 **Multi-head Latent Attention (MLA)** 아키텍처를 사용한다. 표준 MHA/GQA와 달리 K와 V를 별도로 저장하지 않고, 단일 저랭크 잠재 벡터(`c_kv`)로 압축하여 캐싱한다.

**MLA KV 캐시 공식** ([HuggingFace KV 캐시 계산기](https://huggingface.co/spaces/HathoraResearch/LLM-KV-cache-calculator/blob/main/app.py) 및 [DeepSeek-V3 소스코드](https://github.com/deepseek-ai/DeepSeek-V3/blob/main/inference/model.py) 검증):

```
토큰당 요소 수 = num_layers × (kv_lora_rank + qk_rope_head_dim)
              = 78 × (512 + 64)
              = 78 × 576
              = 44,928 요소/토큰
```

[GLM-5.2 config.json](https://huggingface.co/zai-org/GLM-5.2-FP8/raw/main/config.json)에서 확인된 파라미터:
- `kv_lora_rank = 512` (K+V 공유 압축 잠재 벡터 차원)
- `qk_rope_head_dim = 64` (분리된 RoPE 키, 압축 불가)
- `num_hidden_layers = 78`

**FP8 기준 (1 바이트/요소):**

```
토큰당 KV 캐시 = 44,928 바이트 ≈ 43.9 KiB/토큰

1M 토큰 요청 1건:
  44,928 × 1,048,576 = 47,114,387,456 바이트 = 47.11 GB

128K 토큰 요청 1건:
  44,928 × 131,072 = 5,888,802,816 바이트 = 5.89 GB
```

**표준 MHA 공식과의 비교 (MLA에는 적용 불가):**

```
표준 MHA 공식 (잘못된 적용):
  2 × 78 × 64 × 192 × 1 = 1,916,928 요소/토큰
  1M 토큰 요청 = 2.01 TB → 단일 요청도 불가

MLA 실제 공식:
  78 × (512 + 64) = 44,928 요소/토큰
  1M 토큰 요청 = 47.11 GB → 11건 동시 처리 가능

압축 비율: 1,916,928 ÷ 44,928 = 42.7배
```

MLA가 1M 토큰 컨텍스트를 단일 8×B200 노드에서 실현 가능하게 만드는 핵심 기술이다.

### 3.5 최대 동시 요청 수

| 예산 수준 | FP8 KV (47.11 GB/요청) | 128K FP8 KV (5.89 GB/요청) |
|---|---|---|
| A: 696 GB (이론 최대) | **14건** | **118건** |
| B: 656 GB (런타임 오버헤드 공제) | **13건** | **111건** |
| C: 540.8 GB (프로덕션) | **11건** | **91건** |

**프로덕션 검증 (FP8, 예산 C):**
```
11건 × 47.11 GB = 518.2 GB ≤ 540.8 GB ✓ (22.6 GB 여유)
12건 × 47.11 GB = 565.3 GB > 540.8 GB ✗ (예산 초과)
```

### 3.6 vLLM 서빙 설정

[vLLM 공식 문서](https://docs.vllm.ai/en/stable/)에 기반한 권장 서빙 명령어:

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

주요 플래그 설명:

| 플래그 | 값 | 이유 |
|---|---|---|
| `--tensor-parallel-size` | 8 | 8개 B200 GPU에 어텐션 레이어와 전문가 분산 |
| `--kv-cache-dtype` | fp8 | MLA 잠재 벡터를 FP8로 저장, BF16 대비 KV 캐시 절반 |
| `--max-model-len` | 1048576 | 1M 토큰 컨텍스트 지원 |
| `--gpu-memory-utilization` | 0.92 | 총 VRAM의 92% 할당, 8%는 CUDA 컨텍스트 예비 |
| `--max-num-seqs` | 11 | 프로덕션 동시 시퀀스 한계 (예산 C 기준) |
| `--max-num-batched-tokens` | 8192 | 청크 프리필 크기, 1M 컨텍스트 OOM 방지 필수 |
| `--enable-chunked-prefill` | (플래그) | 1M 컨텍스트 프리필을 8K 청크로 분할, 필수 |
| `--enable-prefix-caching` | (플래그) | 코딩 에이전트 반복 컨텍스트 KV 재사용 |
| `--trust-remote-code` | (플래그) | `GlmMoeDsaForCausalLM` 커스텀 아키텍처 로드 필요 |

GEAP 배포 명령어:

```bash
gcloud ai endpoints deploy-model $ENDPOINT_ID \
  --machine-type=a4-highgpu-8g \
  --accelerator=type=nvidia-b200,count=8 \
  --container-image-uri=<vllm-container> \
  --container-args="--model=zai-org/GLM-5.2-FP8 \
    --tensor-parallel-size=8 \
    --kv-cache-dtype=fp8 \
    --gpu-memory-utilization=0.90"
```

위 설정을 전제로, 다음 섹션에서는 코딩 에이전트의 실제 토큰 소비 패턴을 분석한다.

---

## 4. 코딩 에이전트 워크로드 베이스라인

### 4.1 왜 코딩 에이전트는 입력 토큰을 이렇게 많이 소비하는가

AI 코딩 에이전트의 입력/출력 비율은 구조적으로 극단적이다. 약 25:1 이상이다.

모든 도구 호출(파일 읽기, 테스트 실행, grep, 셸 명령)은 **누적된 전체 대화 히스토리**를 모델에 다시 전송한다. 디버깅 세션 30번째 턴에서 에이전트는 매 요청마다 25,000–35,000 토큰의 히스토리를 재전송한다. 반면 출력(패치, 짧은 설명)은 상대적으로 매우 작다.

[Tokenade/Vantage 2026 분석](https://tokenade.net/en/stats/claude-code-token-usage-statistics)에 따르면:
- 50턴 에이전트 세션 1건: 입력 ≈1,000,000 토큰, 출력 ≈40,000 토큰 (25:1 비율)
- 재전송 컨텍스트가 전체 비용의 약 62% 차지 (LeanOps 2026 감사, 비공개 리포트, 30개 팀 대상)

[Jia.je 분석](https://jia.je/kb/en/software/coding_plan.html)에 따르면:
- 바이브 코딩 시나리오에서 입력 토큰이 전체 토큰 볼륨의 99.5%
- 캐시 토큰이 입력+출력의 90–95%

### 4.2 Heavy User 베이스라인 도출

| 파라미터 | 값 | 근거 |
|---|---|---|
| 일일 세션 수 | ≈5회 | 짧은 세션 2–3회 + 긴 세션 2–3회 혼합 |
| 세션당 평균 입력 토큰 | ≈1,000,000 | [Tokenade/Vantage 50턴 벤치마크](https://tokenade.net/en/stats/claude-code-token-usage-statistics) |
| 세션당 평균 출력 토큰 | ≈40,000 | 동일 벤치마크, 25:1 비율 |
| **일일 입력 토큰** | <strong>≈5,000,000</strong> | 5세션 × 1M |
| **일일 출력 토큰** | <strong>≈200,000</strong> | 5세션 × 40K |
| **월간 입력 토큰 (20 근무일)** | **100,000,000 (100M)** | 일일 × 20 |
| **월간 출력 토큰** | **4,000,000 (4M)** | 일일 × 20 |

**외부 검증:** [Break The Ordinary 분석 (Anthropic 데이터 인용)](https://breaktheordinary.com/ai-coding-agent-api-cost-audit/)에 따르면 Claude Code 평균 사용량은 개발자 활동일당 약 $13(API 환산). Sonnet 4.6 요금($3/M 입력) 기준으로 일일 약 4M 입력 토큰에 해당하며, 5M Heavy User 수치와 일치한다.

### 4.3 피크 동시성 분석

표준 엔터프라이즈 소프트웨어 사용 패턴 기준:

| 파라미터 | 값 |
|---|---|
| 피크 동시성 비율 | 활성 개발자의 20% |
| 100명 팀 피크 동시 개발자 | 20명 |
| 평균 요청 크기 (입력) | ≈100,000 토큰 (세션 중반 컨텍스트) |
| 개발자 1인당 일일 요청 수 | ≈50건 (5M ÷ 100K) |

### 4.4 RPM 분석

**케이던스 모델 (운영 관련성 높음):**

피크 시간대 개발자 1인당 30초마다 요청 1건 (에이전트 루프: 파일 읽기 → 모델 대기 → 행동 → 다음 파일 읽기):

| 시나리오 | 개발자 수 | 피크 동시성 | 피크 RPM |
|---|---|---|---|
| 100명, 피크 (케이던스 모델) | 100 | 20% | <strong>≈40 RPM</strong> |
| 300명, 피크 | 300 | 20% | <strong>≈120 RPM</strong> |
| 500명, 피크 | 500 | 20% | <strong>≈200 RPM</strong> |

---

## 5. Z.ai SaaS 비용 분석

### 5.1 API 종량제 가격

[Z.ai 공식 가격 페이지](https://docs.z.ai/guides/overview/pricing) 기준 (2026-07-01 확인):

| 구성 요소 | 가격 (USD/1M 토큰) | 비고 |
|---|---|---|
| 입력 토큰 (신규) | <strong>&#36;1.40</strong> | 표준 신규 입력 |
| 캐시 입력 토큰 | <strong>&#36;0.26</strong> | 자동 암묵적 캐시, 신규 대비 81.4% 할인 |
| 캐시 입력 저장 | <strong>&#36;0.00</strong> | 현재 무료 (한시적 프로모션) |
| 출력 토큰 | <strong>&#36;4.40</strong> | 표준 출력 |

**개발자 1인당 월 API 비용 (Heavy User 100M 입력 / 4M 출력):**

| 캐시 히트율 | 신규 입력 비용 | 캐시 입력 비용 | 출력 비용 | **합계/인/월** |
|---|---|---|---|---|
| 0% (캐시 없음) | $140.00 | $0.00 | $17.60 | <strong>&#36;157.60</strong> |
| 70% | $42.00 | $18.20 | $17.60 | <strong>&#36;77.80</strong> |
| 80% | $28.00 | $20.80 | $17.60 | <strong>&#36;66.40</strong> |
| **85% (중앙 추정치)** | <strong>&#36;21.00</strong> | <strong>&#36;22.10</strong> | <strong>&#36;17.60</strong> | <strong>&#36;60.70</strong> |
| 90% | $14.00 | $23.40 | $17.60 | <strong>&#36;55.00</strong> |
| 95% (Jia.je 상한) | $7.00 | $24.70 | $17.60 | <strong>&#36;49.30</strong> |

공식: `합계 = 157.60 - 114 × r` (r = 캐시 히트율)

Z.ai의 컨텍스트 캐싱은 [자동/암묵적](https://docs.z.ai/guides/capabilities/cache.md)으로 작동한다. 별도 헤더 설정 없이 반복 컨텍스트(시스템 프롬프트, 대화 히스토리, 긴 문서)를 자동 감지하여 캐시 요금을 적용한다.

### 5.2 GLM Coding Plan 구독 티어

[Z.ai 공식 문서](https://docs.z.ai/devpack/overview) 기준:

| 플랜 | 기본 월정액 | 5시간 프롬프트 한도 | 주간 프롬프트 한도 | 권장 사용 패턴 |
|---|---|---|---|---|
| Lite | <strong>&#36;18/월</strong> | ≈80 프롬프트 | ≈400 프롬프트 | 단일 프로젝트 |
| Pro | <strong>&#36;72/월</strong> | ≈400 프롬프트 | ≈2,000 프롬프트 | 1–2개 프로젝트 병렬 |
| Max | <strong>&#36;160/월</strong> | ≈1,600 프롬프트 | ≈8,000 프롬프트 | 2개 이상 프로젝트 병렬 |

**연간 결제 할인 (30% 할인):** Lite $12.60, Pro $50.40, Max $112.00/월

**중요 규칙:**
- [초과 과금 없음](https://docs.z.ai/devpack/faq): 한도 소진 시 다음 5시간 윈도우까지 대기, 잔액 차감 없음
- [계정 공유 금지](https://docs.z.ai/devpack/usage-policy): 개발자 1인당 독립 구독 필요, 팀 쿼터 풀링 불가
- Coding Plan 쿼터는 API 종량제 엔드포인트로 이월 불가 (별도 엔드포인트)

### 5.3 GLM-5.2 쿼터 배율 분석

[Z.ai 공식 문서](https://docs.z.ai/devpack/overview)에 따르면 GLM-5.2 사용 시 쿼터가 추가 차감된다:

> "GLM-5.2 및 GLM-5-Turbo 사용 시 피크 시간대(UTC+8 14:00–18:00)에는 **3배**, 비피크 시간대에는 **2배** 차감됩니다."
> 한시적 혜택(2026년 9월 말까지): 비피크 시간대 **1배** 차감.

UTC+8 기준 9:00–18:00 근무 (비피크 5시간 + 피크 4시간) 가중 평균:

```
2026년 9월 이후 (2× 비피크, 3× 피크):
  (5 × 2 + 4 × 3) ÷ 9 = 22 ÷ 9 = 2.44× 평균 배율

2026년 9월 프로모션 (1× 비피크, 3× 피크):
  (5 × 1 + 4 × 3) ÷ 9 = 17 ÷ 9 = 1.89× 평균 배율
```

**실효 GLM-5.2 주간 프롬프트 예산 (2026년 9월 이후, 2.44× 기준):**

| 플랜 | 명목 주간 한도 | 실효 GLM-5.2 주간 한도 | Heavy User 수요 (58–77 프롬프트/주) 충족? |
|---|---|---|---|
| Lite | 400 | **164** | 예 (약 2배 여유) |
| Pro | 2,000 | **820** | 예 (약 10배 여유) |
| Max | 8,000 | **3,279** | 예 (약 40배 여유) |

**5시간 윈도우 실효 한도 (2.44× 기준):**

| 플랜 | 5시간 실효 GLM-5.2 한도 | Heavy User 버스트 수요 (13–17 프롬프트/5시간) 충족? |
|---|---|---|
| Lite | ≈33 | 예 (Scenario A 기준) |
| Pro | ≈164 | 예 (여유 충분) |
| Max | ≈656 | 예 (대규모 여유) |

**Lite의 운영 제약:** Z.ai는 Lite를 "단일 프로젝트" 전용으로 명시한다. 병렬 서브에이전트(Claude Code subagents, OpenCode task() 호출)를 사용하는 팀에는 운영상 부적합하다.

### 5.4 팀 규모별 Z.ai 비용 매트릭스

**기본 월정액 기준 (USD/월):**

| 옵션 | 인당/월 | 50명 | 100명 | 300명 | 500명 |
|---|---|---|---|---|---|
| API 종량제 (캐시 없음) | $157.60 | $7,880 | $15,760 | $47,280 | $78,800 |
| API 종량제 (85% 캐시) | $60.70 | $3,035 | $6,070 | $18,210 | $30,350 |
| Coding Plan Lite | $18.00 | $900 | $1,800 | $5,400 | $9,000 |
| **Coding Plan Pro** | <strong>&#36;72.00</strong> | <strong>&#36;3,600</strong> | <strong>&#36;7,200</strong> | <strong>&#36;21,600</strong> | <strong>&#36;36,000</strong> |
| Coding Plan Max | $160.00 | $8,000 | $16,000 | $48,000 | $80,000 |

**연간 결제 기준 (30% 할인, USD/월):**

| 옵션 | 인당/월 | 50명 | 100명 | 300명 | 500명 |
|---|---|---|---|---|---|
| Coding Plan Lite (연간) | $12.60 | $630 | $1,260 | $3,780 | $6,300 |
| **Coding Plan Pro (연간)** | <strong>&#36;50.40</strong> | <strong>&#36;2,520</strong> | <strong>&#36;5,040</strong> | <strong>&#36;15,120</strong> | <strong>&#36;25,200</strong> |
| Coding Plan Max (연간) | $112.00 | $5,600 | $11,200 | $33,600 | $56,000 |

---

## 6. GCP 자체 호스팅 TCO

### 6.1 GEAP 가격 구조

[GCP GEAP 공식 가격 페이지](https://cloud.google.com/products/gemini-enterprise-agent-platform/pricing) 기준:

| 구성 요소 | 시간당 가격 | 비율 |
|---|---|---|
| 원시 컴퓨팅 (Compute Engine) | [$64.44/hr](https://cloud.google.com/products/compute/pricing/accelerator-optimized) | 43.5% |
| GEAP 관리 수수료 | $83.772/hr | 56.5% |
| **GEAP Self-Deployed 총액** | **[$148.212/hr](https://cloud.google.com/products/gemini-enterprise-agent-platform/pricing)** | **100%** |

**GEAP 관리 수수료($83.772/hr)가 포함하는 것:**
- 관리형 Vertex AI 엔드포인트 (안정적 HTTPS DNS, TLS 종료)
- 자동 스케일링 컨트롤러
- 컨테이너 런타임 관리 (vLLM 배포, 헬스 체크, 롤링 업데이트)
- Cloud Monitoring/Logging 자동 통합
- VPC 격리 및 네트워크 관리

**핵심:** GEAP는 원시 컴퓨팅이 아닌 관리형 서비스다. 관리 수수료가 총 비용의 56.5%를 차지하며, 이 부분은 CUD 할인 대상이 아니다.

### 6.2 CUD 할인 분석

[GCP AI Hypercomputer 소비 모델 문서](https://cloud.google.com/ai-hypercomputer/docs/consumption-models)에 따르면:

- A4 인스턴스는 Sustained Use Discount(SUD) 및 일반 CUD 미적용
- [AI Hypercomputer Future Reservations](https://docs.cloud.google.com/ai-hypercomputer/docs/reserve-capacity)를 통해 1년/3년 약정 시 vCPU 및 GPU에 최대 53% 할인

**CUD가 관리 수수료에 적용되지 않는 이유:**

[GEAP 가격 페이지](https://cloud.google.com/products/gemini-enterprise-agent-platform/pricing)에 명시:
> "Compute Engine 예약 사용 시 CUD를 포함한 Compute Engine 요금이 적용됩니다. Agent Platform Inference 관리 수수료는 인프라 사용료에 추가됩니다."

즉, CUD는 원시 컴퓨팅($64.44/hr)에만 적용되고, 관리 수수료($83.772/hr)는 항상 전액 청구된다.

**시나리오별 실효 시간당 요금 (추정치):**

| 시나리오 | 컴퓨팅 ($/hr) | 관리 수수료 ($/hr) | 총액 ($/hr) | 총액 대비 실효 할인율 |
|---|---|---|---|---|
| 온디맨드 (CUD 없음) | $64.440 | $83.772 | <strong>&#36;148.212</strong> | 0.0% |
| 1년 CUD (컴퓨팅 35% 할인, 추정치) | $41.886 | $83.772 | <strong>&#36;125.658</strong> | 15.2% |
| 3년 CUD (컴퓨팅 53% 할인, 추정치) | $30.287 | $83.772 | <strong>&#36;114.059</strong> | 23.0% |
| 이론적 최선 (전체 53% 할인) | — | — | $69.660 | 53.0% (비현실적) |

**핵심 발견:** 명목 CUD 할인율은 53%이지만, 관리 수수료가 할인 대상이 아니기 때문에 실효 할인율은 <strong>21.5–23%</strong>에 불과하다. 이는 많은 팀이 기대하는 절감 효과와 크게 다르다.

### 6.3 전체 월 TCO 분석

**온디맨드 (730시간/월, GCP 표준 청구 월):**

| 항목 | 월 비용 | 비율 | 출처 |
|---|---|---|---|
| **컴퓨팅 (GEAP 온디맨드)** | <strong>&#36;108,194.76</strong> | **93.5%** | [$148.212/hr × 730hr](https://cloud.google.com/products/gemini-enterprise-agent-platform/pricing) |
| 스토리지 (Hyperdisk 500GB + GCS 744GB) | $64.89 | 0.1% | [GCP 디스크 가격](https://cloud.google.com/compute/disks-image-pricing), [GCS 가격](https://cloud.google.com/storage/pricing) |
| 네트워크 이그레스 (500명 기준) | ≈$2.00 | <0.1% | [GCP 네트워크 티어 가격](https://cloud.google.com/network-tiers/pricing) |
| Cloud Monitoring (추정치) | $25.00 | <0.1% | [Cloud Observability 가격](https://cloud.google.com/products/observability/pricing) |
| Cloud Logging (무료 티어 내) | $0.00 | 0.0% | [Cloud Observability 가격](https://cloud.google.com/products/observability/pricing) |
| 운영 오버헤드 (0.5 FTE @ $180K/년, 추정치) | $7,500.00 | 6.5% | 인건비 추정치 |
| **월 TCO 합계 (온디맨드)** | <strong>&#36;115,785</strong> | **100%** | |

**CUD 시나리오별 월 TCO:**

| 시나리오 | 컴퓨팅 비용 | 기타 비용 | **월 TCO** |
|---|---|---|---|
| 온디맨드 | $108,195 | $7,590 | <strong>&#36;115,785</strong> |
| 1년 CUD (추정치) | $91,730 | $7,590 | <strong>&#36;99,321</strong> |
| 3년 CUD (추정치) | $83,263 | $7,590 | <strong>&#36;90,853</strong> |

**비용 구성 시각화 (온디맨드):**

```
컴퓨팅 (GEAP)   ████████████████████████████████████████████  93.5%  $108,195
운영 오버헤드   ███                                            6.5%   $7,500
스토리지        ▏                                              0.1%   $65
모니터링        ▏                                              <0.1%  $25
네트워크        ▏                                              <0.1%  $1
로깅            ▏                                              0.0%   $0
                ─────────────────────────────────────────────────────
합계                                                           100%   $115,785/월
```

### 6.4 개발자 1인당 비용

단일 `a4-highgpu-8g` 노드가 모든 팀 규모를 서비스한다고 가정 (처리량 한계 분석은 섹션 7 참조):

| 팀 규모 | 온디맨드 | 1년 CUD (추정치) | 3년 CUD (추정치) | 비고 |
|---|---|---|---|---|
| 50명 | $2,316/인/월 | $1,986/인/월 | $1,817/인/월 | 단일 노드 |
| 100명 | $1,158/인/월 | $993/인/월 | $909/인/월 | 단일 노드 |
| 300명 | $386/인/월 | $331/인/월 | $303/인/월 | ⚠ 단일 노드 가정 — 실제 2노드 필요 (§7.2) |
| 500명 | $232/인/월 | $199/인/월 | $182/인/월 | ⚠ 단일 노드 가정 — 실제 4노드 필요 (§7.2) |

3년 CUD의 개발자 1인당 절감액은 온디맨드 대비 일정하게 <strong>21.5%</strong>다. 컴퓨팅이 지배적이고 할인율이 고정되어 있기 때문이다.

---

## 7. 손익분기점 및 규모의 경제

### 7.1 재무적 손익분기점 (10개 조합)

**공식:** `손익분기점 개발자 수 = 노드 월 비용 ÷ 개발자 1인당 SaaS 비용`

이 계산은 단일 노드가 모든 개발자를 서비스할 수 있다고 가정한다. 섹션 7.3에서 이 가정의 현실성을 검증한다.

**온디맨드 ($115,785/월) vs 5개 SaaS 옵션:**

| SaaS 옵션 | 인당/월 | 손익분기점 | 자체 호스팅이 저렴해지는 시점 |
|---|---|---|---|
| API 캐시 없음 | $157.60 | **735명** | >735명 |
| API 85% 캐시 | $60.70 | **1,908명** | >1,908명 |
| Lite | $18.00 | **6,433명** | >6,433명 |
| Pro | $72.00 | **1,608명** | >1,608명 |
| Max | $160.00 | **724명** | >724명 |

**3년 CUD ($90,853/월) vs 5개 SaaS 옵션:**

| SaaS 옵션 | 인당/월 | 손익분기점 | 자체 호스팅이 저렴해지는 시점 |
|---|---|---|---|
| API 캐시 없음 | $157.60 | **577명** | >577명 |
| API 85% 캐시 | $60.70 | **1,497명** | >1,497명 |
| Lite | $18.00 | **5,047명** | >5,047명 |
| Pro | $72.00 | **1,262명** | >1,262명 |
| **Max** | <strong>&#36;160.00</strong> | **568명** | **>568명** |

**산술 검증:**
```
온디맨드 vs Max:    115,785 ÷ 160.00 = 723.66 ✓
온디맨드 vs Pro:    115,785 ÷ 72.00  = 1,608.13 ✓
3년 CUD vs Max:      90,853 ÷ 160.00 = 567.83 ✓
3년 CUD vs Pro:      90,853 ÷ 72.00  = 1,261.85 ✓
```

**핵심 관찰:**
1. 가장 낮은 손익분기점은 **568명** (3년 CUD vs Z.ai Max)
2. Lite($18/인) 대비 손익분기점은 5,000–6,400명으로 현실적으로 불가능
3. Pro($72/인) 대비 손익분기점은 1,262–1,608명으로 분석 범위(50–500명)를 크게 초과

### 7.2 처리량 한계: 진짜 병목

VRAM이 아닌 **디코드 처리량**이 실제 병목이다.

**B200 FP8 디코드 처리량 추정 (추정치):**

GLM-5.2는 토큰당 40B 활성 파라미터(MoE). FP8(1 바이트/파라미터)에서 활성 가중치 읽기는 GPU당 5 GB 필요.

```
이론적 디코드 처리량:
  GPU당: 8 TBps ÷ 5 GB = 1,600 토큰/초
  집계 (TP=8, 모든 GPU가 동일 토큰 처리): 1,600 토큰/초

현실적 처리량 (MoE 라우팅, 어텐션, all-reduce 오버헤드 포함):
  보수적 (50% 효율): 800 토큰/초
  현실적 (60% 효율): 960 토큰/초
  낙관적 (70% 효율): 1,120 토큰/초
```

**RPM 용량 (요청당 출력 1,000 토큰 기준):**

| 처리량 | RPM 용량 | 최대 개발자 수 (100명당 40 RPM 기준) |
|---|---|---|
| 800 토큰/초 (보수적) | 48 RPM | **120명** |
| 960 토큰/초 (현실적) | 57 RPM | **142명** |
| 1,120 토큰/초 (낙관적) | 67 RPM | **168명** |

**단일 노드 처리 가능 개발자 수: 약 150명 (추정치)**

| 팀 규모 | 피크 RPM | 필요 노드 수 | 비고 |
|---|---|---|---|
| 50명 | 20 RPM | **1노드** | 2.85배 여유 |
| 100명 | 40 RPM | **1노드** | 1.4배 여유 (용량 한계 근접) |
| 300명 | 120 RPM | **2노드** | 1노드 용량 초과 |
| 500명 | 200 RPM | **4노드** | 1노드 용량 초과 |

**VRAM은 병목이 아니다:**
- **100K 평균 컨텍스트:** 단일 노드 120건 동시 → 20% 동시성 기준 600명 처리 가능
- **85% 프리픽스 캐시 히트:** ≈800건 동시 → 4,000명 이상 처리 가능
- **VRAM 병목 임계점:** 평균 컨텍스트 500K 초과 시 (코딩 에이전트 워크로드의 약 5%)

### 7.3 손익분기점의 신기루: 에스컬레이팅 스파이럴

재무적 손익분기점(568명)과 처리량 한계(150명/노드)를 교차 분석하면 자체 호스팅의 근본적 문제가 드러난다.

**에스컬레이팅 스파이럴:**

| 목표 손익분기점 | 필요 노드 수 | 수정된 월 비용 | 수정된 손익분기점 | 도달 가능? |
|---|---|---|---|---|
| 568명 (CUD vs Max) | 4노드 (⌈568÷150⌉) | $363,412 (4×$90,853) | **2,271명** | 아니오 |
| 2,271명 (수정) | 16노드 (⌈2,271÷150⌉) | $1,453,648 (16×$90,853) | **9,085명** | 아니오 |
| 9,085명 (재수정) | 61노드 (⌈9,085÷150⌉) | $5,542,033 (61×$90,853) | **34,638명** | 아니오 |

노드를 추가할수록 비용이 비례적으로 증가하여 손익분기점이 계속 후퇴한다. 이는 발산 수열이다.

### 7.4 수학적 불가능성 증명

다음 변수를 정의한다:
- N = 개발자 수
- C_node = 노드당 월 비용 ($90,853, 3년 CUD)
- D_node = 노드당 최대 개발자 수 (150명)
- S = 개발자 1인당 SaaS 비용 ($160, Max 플랜)

```
자체 호스팅 비용 = C_node × ⌈N ÷ D_node⌉
SaaS 비용 = S × N

⌈N ÷ D_node⌉ ≥ N ÷ D_node이므로 (천장 함수의 정의에 의해):

자체 호스팅 비용 ≥ (C_node ÷ D_node) × N
                 = ($90,853 ÷ 150) × N
                 = $605.69 × N
                 > $160 × N
                 = SaaS 비용

비율 ≥ $605.69 ÷ $160 = 3.79× (모든 유한한 N에 대해 성립)
```

**결론: 비율은 팀 규모와 무관하게 항상 3.79배 이상이다. 손익분기점은 어떤 유한한 팀 규모에서도 수학적으로 도달 불가능하다.**

이 증명에서 탈출하려면 다음 중 하나가 필요하다:
- 노드당 비용을 74% 감소 ($90,853 → $24,000 이하), 또는
- 노드당 처리량을 4배 증가 (150명 → 600명), 또는
- 두 조건의 조합

현재 GEAP 가격 구조와 B200 하드웨어로는 어느 것도 달성 불가능하다.

### 7.5 개발자 1인당 비용 하한선

노드 수와 용량이 모두 선형으로 확장되므로, 규모에 따른 개발자 1인당 비용은 상수로 수렴한다:

| 시나리오 | 노드당 비용 | 노드당 개발자 수 | **개발자 1인당 비용 하한선** |
|---|---|---|---|
| 온디맨드, 150명/노드 | $115,785 | 150 | <strong>&#36;772/인/월</strong> |
| 3년 CUD, 150명/노드 | $90,853 | 150 | <strong>&#36;606/인/월</strong> |
| 온디맨드, 200명/노드 (낙관적) | $115,785 | 200 | $579/인/월 |
| 3년 CUD, 200명/노드 (낙관적) | $90,853 | 200 | $454/인/월 |

가장 낙관적인 시나리오(200명/노드, 3년 CUD)에서도 개발자 1인당 비용은 <strong>&#36;454/월</strong>로, Z.ai Max($160)보다 2.8배, Z.ai Pro($72)보다 6.3배 비싸다.

**구조적 근본 원인:** GEAP 관리 수수료($83.772/hr = $61,154/월)가 CUD 할인 대상이 아니다. 150명/노드 기준으로 관리 수수료만 <strong>&#36;408/인/월</strong>이다. 이는 Z.ai Max($160/인)보다 2.5배 비싸다. 컴퓨팅이 무료라도 GEAP 관리 수수료만으로 자체 호스팅이 Z.ai SaaS보다 비싸다.

---

## 8. 시나리오별 비교 매트릭스

### 8.1 전체 비용 비교 (3년 CUD 기준, USD/월)

| 옵션 | 50명 | 100명 | 300명 | 500명 |
|---|---|---|---|---|
| Z.ai Lite ($18/인) | $900 | $1,800 | $5,400 | $9,000 |
| Z.ai API 85% 캐시 ($60.70/인) | $3,035 | $6,070 | $18,210 | $30,350 |
| **Z.ai Pro ($72/인)** | <strong>&#36;3,600</strong> | <strong>&#36;7,200</strong> | <strong>&#36;21,600</strong> | <strong>&#36;36,000</strong> |
| Z.ai Max ($160/인) | $8,000 | $16,000 | $48,000 | $80,000 |
| 자체 호스팅 3년 CUD (노드 수) | $90,853 (1노드) | $90,853 (1노드) | $181,706 (2노드) | $363,412 (4노드) |
| 자체 호스팅 온디맨드 (노드 수) | $115,785 (1노드) | $115,785 (1노드) | $231,570 (2노드) | $463,140 (4노드) |

### 8.2 자체 호스팅 vs Z.ai Pro 배율

| 팀 규모 | Z.ai Pro | 자체 호스팅 (3년 CUD) | 배율 |
|---|---|---|---|
| 50명 | $3,600 | $90,853 | **25.2배 비쌈** |
| 100명 | $7,200 | $90,853 | **12.6배 비쌈** |
| 300명 | $21,600 | $181,706 | **8.4배 비쌈** |
| 500명 | $36,000 | $363,412 | **10.1배 비쌈** |

### 8.3 자체 호스팅 vs Z.ai Max 배율

| 팀 규모 | Z.ai Max | 자체 호스팅 (3년 CUD) | 배율 |
|---|---|---|---|
| 50명 | $8,000 | $90,853 | **11.4배 비쌈** |
| 100명 | $16,000 | $90,853 | **5.7배 비쌈** |
| 300명 | $48,000 | $181,706 | **3.8배 비쌈** |
| 500명 | $80,000 | $363,412 | **4.5배 비쌈** |

### 8.4 팀 규모별 권고사항

| 팀 규모 | 권고 옵션 | 월 비용 | 이유 |
|---|---|---|---|
| **50명** | Z.ai Pro | $3,600 | 자체 호스팅 대비 25.2배 저렴. 단일 노드는 50명에게 과도한 용량 (150명 처리 가능). |
| **100명** | Z.ai Pro | $7,200 | 자체 호스팅 대비 12.6배 저렴. 단일 노드가 처리량 한계에 근접 (100/150명). |
| **300명** | Z.ai Pro | $21,600 | 자체 호스팅 대비 8.4배 저렴. 300명은 2노드 필요 ($181,706). |
| **500명** | Z.ai Pro | $36,000 | 자체 호스팅 대비 10.1배 저렴. 500명은 4노드 필요 ($363,412). |

**병렬 에이전트 없는 팀 (단일 프로젝트 워크플로우):** Lite($18/인)가 기술적으로 충분하며 가장 저렴하다. 단, Z.ai가 Lite를 "단일 프로젝트" 전용으로 명시하므로 병렬 서브에이전트 사용 시 Pro로 업그레이드 필요.

---

## 9. 최종 권고사항

### 9.1 핵심 권고

**Z.ai SaaS를 사용하라. 병렬 에이전트를 사용하는 Heavy User 팀에는 Pro($72/인/월)가 최적이다.**

자체 호스팅은 50–500명 규모의 어떤 팀에서도 경제적으로 타당하지 않다. 수학적 증명에 따르면 개발자 1인당 비용 하한선($606/월, 3년 CUD)이 Z.ai Max($160/월)보다 최소 3.79배 비싸며, 이 비율은 팀 규모와 무관하게 항상 3.79배 이상이다.

### 9.2 플랜 선택 가이드

| 팀 프로필 | 권고 플랜 | 인당/월 | 이유 |
|---|---|---|---|
| 단일 프로젝트, 병렬 에이전트 없음 | Lite | $18 | 주간 한도(164 실효 프롬프트)가 Heavy User 수요(58–77 프롬프트/주) 충족 |
| 병렬 서브에이전트, 1–2개 프로젝트 동시 | **Pro** | <strong>&#36;72</strong> | Z.ai 권장 티어, 5시간 한도 5배, 운영상 안전한 최저가 |
| 2개 이상 프로젝트 동시, 장시간 자율 세션 | Max | $160 | 5시간 한도 20배, 대규모 병렬 에이전트 워크로드 |
| 커스텀 스크립트/CI 파이프라인 | API 종량제 | $60.70 (85% 캐시) | Coding Plan은 공식 지원 도구만 허용 |

### 9.3 자체 호스팅이 타당한 5가지 조건

자체 호스팅은 Z.ai SaaS가 워크로드를 서비스할 수 없을 때만 고려한다:

1. **커스텀 파인튜닝 모델:** 도메인 특화 GLM-5.2 파인튜닝 버전이 필요한 경우. Z.ai SaaS는 기본 모델만 제공한다.

2. **엄격한 데이터 주권:** HIPAA, FedRAMP, EU GDPR 데이터 거주 요건 등 규제로 인해 코드를 제3자 SaaS API에 전송할 수 없는 경우. GEAP는 자체 VPC 내에서 실행된다.

3. **매우 긴 컨텍스트 워크로드 (평균 500K+ 토큰):** Z.ai Coding Plan의 5시간 한도나 API 속도 제한을 초과하는 장시간 컨텍스트 요청이 지배적인 경우. 자체 호스팅은 개발자당 프롬프트 한도가 없다.

4. **팀 규모 3,000명 초과:** 매우 대규모에서는 Google Cloud 계정 팀과의 맞춤형 GEAP 가격 협상이 가능하여 관리 수수료가 낮아질 수 있다. 이는 본 보고서의 50–500명 분석 범위를 벗어난다.

5. **비표준 추론 설정:** 커스텀 vLLM 플래그, 대안 서빙 런타임(SGLang, TensorRT-LLM), 또는 SaaS가 지원하지 않는 특수 디코딩 전략(투기적 디코딩, 빔 서치)이 필요한 경우.

표준 코딩 에이전트 사용 사례(기본 GLM-5.2, 병렬 에이전트, 50–500명, 일반적인 컨텍스트 길이)에서는 이 조건 중 어느 것도 해당되지 않으며, Z.ai SaaS가 명확한 선택이다.

### 9.4 구현 로드맵

**즉시 실행 (0–2주):**
1. Z.ai 계정 생성 및 GLM-5.2 API 키 발급
2. 팀 규모에 따라 Coding Plan 티어 선택 (병렬 에이전트 팀은 Pro)
3. Claude Code, OpenCode, Cursor 등 공식 지원 도구 설정
4. 첫 달 `usage.prompt_tokens_details.cached_tokens`로 실제 캐시 히트율 측정

**1개월 후 검토:**
- 실제 캐시 히트율 확인 (목표: 80–90%)
- 5시간 윈도우 한도 소진 빈도 모니터링
- 필요 시 Lite → Pro 또는 Pro → Max 업그레이드

---

## 10. 가정 및 한계점

### 10.1 검증된 사실 (GCP/Z.ai 공식 문서 인용)

| 사실 | 출처 |
|---|---|
| GEAP 온디맨드 가격 $148.212/hr | [GCP GEAP 가격 페이지](https://cloud.google.com/products/gemini-enterprise-agent-platform/pricing) |
| 원시 컴퓨팅 가격 $64.44/hr | [GCP 가속기 최적화 VM 가격](https://cloud.google.com/products/compute/pricing/accelerator-optimized) |
| A4 Persistent Disk 미지원 | [AI Hypercomputer 문서](https://docs.cloud.google.com/ai-hypercomputer/docs/create/create-vm) |
| A4 SUD/일반 CUD 미적용 | [AI Hypercomputer 문서](https://docs.cloud.google.com/ai-hypercomputer/docs/create/create-vm) |
| Future Reservations 최대 53% 할인 | [AI Hypercomputer 소비 모델](https://cloud.google.com/ai-hypercomputer/docs/consumption-models) |
| GEAP 관리 수수료 비할인 | [GEAP 가격 페이지](https://cloud.google.com/products/gemini-enterprise-agent-platform/pricing) |
| A4 Spot VM 미지원 | [AI Hypercomputer 문서](https://docs.cloud.google.com/ai-hypercomputer/docs/create/create-vm) |
| Z.ai API 가격 ($1.40/$0.26/$4.40) | [Z.ai 공식 가격 페이지](https://docs.z.ai/guides/overview/pricing) |
| Coding Plan 가격 ($18/$72/$160) | [Z.ai Coding Plan 개요](https://docs.z.ai/devpack/overview) |
| 초과 과금 없음 | [Z.ai Coding Plan FAQ](https://docs.z.ai/devpack/faq) |
| GLM-5.2 쿼터 배율 (2×/3×) | [Z.ai Coding Plan 개요](https://docs.z.ai/devpack/overview) |
| MLA KV 캐시 공식 | [DeepSeek-V3 논문](https://arxiv.org/abs/2412.19437) |
| GLM-5.2 FP8 가중치 744 GB | [HuggingFace GLM-5.2-FP8](https://huggingface.co/zai-org/GLM-5.2-FP8) |

### 10.2 추정치 (레이블 명시)

| 추정치 | 값 | 불확실성 |
|---|---|---|
| 1년 CUD 컴퓨팅 할인율 | ≈35% | GCP가 A4 CUD 요율을 공개 미게시. 표준 GPU CUD 패턴 기반 추정. |
| 3년 CUD 컴퓨팅 할인율 | ≈53% | GCP 문서의 "최대 53%" 상한값. 실제 협상 결과에 따라 낮을 수 있음. |
| 운영 오버헤드 | $7,500/월 (0.5 FTE) | 인건비 추정치. 300명 이상 팀에서 $11K≈$17K로 증가 가능. |
| 디코드 처리량 | ≈1,000 토큰/초 | B200에서 GLM-5.2 FP8 실제 벤치마크 미수행. ±30% 오차 가능. |
| 단일 노드 개발자 용량 | ≈150명 | 케이던스 모델(30초/요청, 20% 동시성) 기반. 실제 버스트 패턴에 따라 변동. |
| 캐시 히트율 | 85% (중앙 추정치) | Jia.je 90–95%, LeanOps 62% 사이의 중간값. 실제 레포 특성에 따라 변동. |
| Hyperdisk Balanced 가격 | ≈$0.10/GB-월 | GCP 가격 페이지에서 명시적 추출 불가. pd-balanced 후계 가격 기반 추정. |
| Cloud Monitoring 비용 | $25/월 | 커스텀 대시보드 및 알림 정책 추가 비용 추정. |

### 10.3 민감도 분석

**처리량이 2배 향상된다면 (2,000 토큰/초, 300명/노드):**
- 개발자 1인당 비용 하한선: $90,853 ÷ 300 = $303/인/월
- Z.ai Max($160) 대비 여전히 1.9배 비쌈
- 결론 변화 없음

**캐시 히트율이 70%로 낮다면:**
- API 종량제 비용: $77.80/인/월 (vs Pro $72/인/월)
- API 종량제가 Pro보다 약간 비싸지며, 예측 가능한 비용 상한 없음

**Z.ai가 가격을 2배 인상한다면:**
- Pro: $144/인/월, Max: $320/인/월
- 자체 호스팅 vs Pro 배율: 여전히 4.2배 (3년 CUD, 150명/노드)
- 결론 변화 없음

### 10.4 분석 범위 외 항목

- **멀티 리전 배포:** 이 분석은 us-central1 단일 리전 기준. 다른 리전은 GEAP 가격 및 네트워크 이그레스 요금이 다를 수 있음.
- **파인튜닝 비용:** GLM-5.2 파인튜닝 비용은 포함되지 않음.
- **개발자 생산성 향상:** AI 코딩 에이전트 도입으로 인한 개발자 생산성 향상 ROI는 이 TCO 분석의 범위 밖.
- **Z.ai 가격 변동 리스크:** Z.ai가 Coding Plan 가격을 인상하거나 GLM-5.2 쿼터 배율을 변경할 경우 분석 결과가 달라질 수 있음.
- **A4 Spot VM:** 현재 미지원. 향후 지원 시 컴퓨팅 비용 60–70% 절감 가능하여 경제성이 달라질 수 있음.

---

## 11. 참조 링크

### GCP 인프라 및 가격

| 문서 | URL |
|---|---|
| GCP GPU 사양 (`a4-highgpu-8g`) | https://cloud.google.com/compute/docs/gpus |
| GCP GPU 네트워크 대역폭 | https://cloud.google.com/compute/docs/gpus/gpu-network-bandwidth |
| GCP AI Hypercomputer 문서 | https://cloud.google.com/ai-hypercomputer/docs/gpu |
| A4 VM 생성 (Hyperdisk 전용, SUD 미적용) | https://docs.cloud.google.com/ai-hypercomputer/docs/create/create-vm |
| AI Hypercomputer 소비 모델 (Future Reservations) | https://cloud.google.com/ai-hypercomputer/docs/consumption-models |
| AI Hypercomputer 용량 예약 | https://docs.cloud.google.com/ai-hypercomputer/docs/reserve-capacity |
| GCP CUD 개요 | https://docs.cloud.google.com/compute/docs/instances/committed-use-discounts-overview |
| GCP CUD 등록 | https://docs.cloud.google.com/compute/docs/instances/signing-up-committed-use-discounts |
| GEAP Self-Deployed Models 개요 | https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/model-garden/self-deployed-models |
| GEAP Model Garden 배포 | https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/open-models/deploy-model-garden |
| GEAP 커스텀 vLLM 컨테이너 | https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/open-models/deploy-custom-vllm |
| **GEAP 가격 ($148.212/hr)** | https://cloud.google.com/products/gemini-enterprise-agent-platform/pricing |
| GCP 가속기 최적화 VM 가격 ($64.44/hr) | https://cloud.google.com/products/compute/pricing/accelerator-optimized |
| GCP 디스크 및 이미지 가격 | https://cloud.google.com/compute/disks-image-pricing |
| Hyperdisk 문서 | https://cloud.google.com/compute/docs/disks/hyperdisks |
| GCS 가격 ($0.020/GB-월) | https://cloud.google.com/storage/pricing |
| GCP 네트워크 서비스 티어 가격 | https://cloud.google.com/network-tiers/pricing |
| GCP VPC 가격 | https://cloud.google.com/vpc/pricing |
| Cloud Observability 가격 | https://cloud.google.com/products/observability/pricing |
| Cloud Logging 무료 티어 | https://cloud.google.com/blog/topics/cost-management/how-to-approach-cloud-logging-pricing-for-cloud-admins |
| Cloud Monitoring 비용 이해 | https://cloud.google.com/blog/products/management-tools/learn-to-understand-and-reduce-cloud-monitoring-costs |

### GLM-5.2 모델

| 문서 | URL |
|---|---|
| GLM-5.2 FP8 HuggingFace 카드 | https://huggingface.co/zai-org/GLM-5.2-FP8 |
| GLM-5.2 BF16 config.json | https://huggingface.co/zai-org/GLM-5.2/raw/main/config.json |
| GLM-5.2 FP8 config.json | https://huggingface.co/zai-org/GLM-5.2-FP8/raw/main/config.json |
| GLM-5.2 공식 GitHub README | https://github.com/zai-org/GLM-5/blob/main/README.md |
| GLM-5.2 Z.ai 블로그 (2026-06-16) | https://z.ai/blog/glm-5.2 |
| GLM-5.2 NVFP4 (753B/40B 파라미터 확인) | https://huggingface.co/nvidia/GLM-5.2-NVFP4 |

### MLA KV 캐시 기술 문서

| 문서 | URL |
|---|---|
| **DeepSeek-V3 논문 (MLA 공식)** | https://arxiv.org/abs/2412.19437 |
| DeepSeek-V3 추론 소스코드 (kv_cache + pe_cache) | https://github.com/deepseek-ai/DeepSeek-V3/blob/main/inference/model.py |
| HuggingFace LLM KV 캐시 계산기 (MLA 공식 구현) | https://huggingface.co/spaces/HathoraResearch/LLM-KV-cache-calculator/blob/main/app.py |
| MLA 설명 (576 요소/토큰) | https://www.tensoreconomics.com/p/deepseek-sparse-attention-from-first |

### vLLM 서빙

| 문서 | URL |
|---|---|
| vLLM 공식 문서 | https://docs.vllm.ai/en/stable/ |
| vLLM MLA 어텐션 구현 | https://docs.vllm.ai/en/stable/api/vllm/model_executor/layers/attention/mla_attention |
| vLLM FP8 MLA 프리필 커널 | https://docs.vllm.ai/en/stable/api/vllm/v1/attention/backends/mla/rocm_aiter_mla |

### Z.ai SaaS 가격 및 정책

| 문서 | URL |
|---|---|
| **Z.ai 공식 가격 페이지** | https://docs.z.ai/guides/overview/pricing |
| Z.ai 컨텍스트 캐싱 | https://docs.z.ai/guides/capabilities/cache.md |
| **Z.ai Coding Plan 개요** | https://docs.z.ai/devpack/overview |
| **Z.ai Coding Plan FAQ** | https://docs.z.ai/devpack/faq |
| Z.ai Coding Plan 사용 정책 | https://docs.z.ai/devpack/usage-policy |
| Z.ai Coding Plan 빠른 시작 | https://docs.z.ai/devpack/quick-start |
| Z.ai 구독 페이지 | https://z.ai/subscribe |
| AI//Cost GLM-5.2 트래커 | https://aicost.tools/llm-cost/zhipu/glm-5-2/ |
| Apidog GLM-5.2 가격 블로그 (2026-06-17) | https://apidog.com/blog/glm-5-2-pricing/ |
| Hyscaler GLM Coding Plan 리뷰 | https://hyscaler.com/insights/glm-coding-plan-review/ |
| AI Pricing Guru GLM Coding Pro | https://www.aipricing.guru/subscriptions/zai-glm-coding-pro/ |
| Datacamp GLM-5.2 가이드 | https://www.datacamp.com/blog/glm-5-2 |

### 코딩 에이전트 토큰 소비 연구

| 문서 | URL |
|---|---|
| Tokenade/Vantage — Claude Code 토큰 사용 통계 (2026) | https://tokenade.net/en/stats/claude-code-token-usage-statistics |
| Tokenade — AI 코딩 에이전트 컨텍스트 엔지니어링 (2026) | https://tokenade.net/en/articles/context-engineering |
| Break The Ordinary — AI 코딩 에이전트 API 비용 감사 (2026) | https://breaktheordinary.com/ai-coding-agent-api-cost-audit/ |
| WebProNews — AI 코더가 인간 비용을 초과할 때 (2026) | https://www.webpronews.com/when-ai-coders-outspend-their-human-overlords/ |
| Jia.je — AI Coding Plan 분석 | https://jia.je/kb/en/software/coding_plan.html |
| OpenReview — 코딩 에이전트의 토큰 소비 패턴 (학술) | https://openreview.net/pdf/969cd1d8b01a45caef5ad3d26e16e99986ec1ba9.pdf |

---

*보고서 생성: 2026-07-01*  
*데이터 기준일: 2026-07-01*  
*독립 검토: 계획 준수, 수학 품질, 수동 QA, 범위 충실도 — 4개 항목 승인 완료*  
*다음 검토 권장 시점: Z.ai 가격 변경 시 또는 2026-10-01 (GLM-5.2 쿼터 배율 프로모션 종료 후)*
