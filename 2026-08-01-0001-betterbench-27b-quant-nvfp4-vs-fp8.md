
# Report card — BetterBench paired A/B: 27b NVFP4 vs FP8 (vLLM 0.25.1, GB10)

> **BetterBench A/B** — *"paired, drift-cancelled A/B measurement … engineered to resolve ~1% differences"*
> (`betterbench ab`: two endpoints, **interleaved one request at a time**, so drift cancels).
> — [BetterBench README](https://github.com/GGZ14/BetterBench)

**What we ran on:** **GB10** (sm_121), driver 580.159.03, kernel `6.17.0-1021-nvidia`. **One variable:** the weight
quant. Same model (Qwen3.6-27B), same engine (vLLM 0.25.1), same KV(fp8)/attn(flashinfer)/MAXLEN — only NVFP4 (4-bit)
vs FP8 (8-bit) differs. Both quants co-tenant on the one GB10, interleaved.

**Provenance (locked):** **A** = `nvidia/Qwen3.6-27B-NVFP4@0893e16` (`--quantization modelopt`, 21 GB weights);
**B** = `Qwen/Qwen3.6-27B-FP8` (`--quantization fp8`, blockwise e4m3, 29 GB weights). Engine vLLM **0.25.1**, both sides.
BetterBench git `3655947`. Greedy.

> ⚠ **Speed only, and a RELATIVE delta** — co-tenant at reduced GMU, so the absolutes are lower than a dedicated serve
> (nvfp4 12.3 here vs the sweep's 12.2 — they agree). The A/B isolates *only* the quant effect.

## Results — B (FP8) vs A (NVFP4)

| metric | mean A (nvfp4) | mean B (fp8) | Δ (B vs A) | 95% CI | verdict |
|---|--:|--:|--:|---|---|
| **decode throughput** (t/s) | 12.26 | 7.85 | **−35.95%** | [−36.24%, −35.66%] | FP8 **35.95% slower** — significant |
| **ITL median smoothness** | — | — | **−56.24%** | [−56.58%, −55.91%] | FP8 56% slower — significant |

Converged in **12 pairs** (paired CV 0.46%, CI half-width 0.29% ≤ 1.0% MDE → auto-stop).
**⇒ NVFP4 decodes ≈ 1.56× faster than FP8** (12.26 / 7.85).

## Read in 20 seconds
- **NVFP4 is ~1.56× faster than FP8 on 27b decode** — a large, unambiguous, operationally material gap.
- **This confirms the ~1.5× quant rule** (FP8 = ~2× the weight bytes → ~2× the bytes moved per token; decode is
  memory-bandwidth-bound, so throughput scales inversely with weight size). Disk footprint 29 vs 21 GB is the tell.
- **Contrast with the engine-version A/B** (0.23 vs 0.25 = <1%): **quant is a real decode lever; engine version is not.**
  For a decode-bound 24/7 serve, NVFP4 is the clear default; FP8 costs ~36% throughput for no quality mandate here.

## Reproduce (externally repeatable)
```bash
# same model + engine both sides; only the weight quant differs. Serve A on :8000, B on :8001 (sequenced).
#   A: nvidia/Qwen3.6-27B-NVFP4  --quantization modelopt    B: Qwen/Qwen3.6-27B-FP8  --quantization fp8
VLLM_NVFP4_GEMM_BACKEND=marlin VLLM_MXFP4_USE_MARLIN=1 VLLM_MARLIN_USE_ATOMIC_ADD=1 \
vllm serve nvidia/Qwen3.6-27B-NVFP4 --revision 0893e16 --served-model-name 27b \
  --quantization modelopt --kv-cache-dtype fp8 --attention-backend flashinfer \
  --max-model-len 8192 --max-num-seqs 8 --gpu-memory-utilization 0.30 --port 8000
vllm serve Qwen/Qwen3.6-27B-FP8 --served-model-name 27b \
  --quantization fp8 --kv-cache-dtype fp8 --attention-backend flashinfer \
  --max-model-len 8192 --max-num-seqs 8 --gpu-memory-utilization 0.40 --port 8001
betterbench ab --endpoint-a http://localhost:8000/v1 --endpoint-b http://localhost:8001/v1 \
  --model 27b --mde 1.0 --out ab.json
```

## What this card omits
- **Per-category / concurrency / prefill** — `ab` mode measures only the paired decode + ITL deltas.
- **Quality** — a perf-only comparison. NVFP4 vs FP8 *accuracy* (does the 4-bit lose anything?) is a separate,
  quality-axis test; this card only says NVFP4 serves ~1.56× faster.
- **Absolute throughput** — co-tenant at reduced GMU; only the ratio is the deliverable.

## Caveats
- **27b only.** The ratio should hold for any bandwidth-bound dense decode, but MoE models (fewer active params)
  may show a different absolute picture.
- **Co-tenant, GMU 0.30/0.40** — the fp8 side needs more reserve for its 29 GB weights; sequenced startup avoids
  the co-tenant init-OOM. This is a relative delta, not an absolute-throughput number.

# Detailed Results

The 12 interleaved decode-t/s pairs (inline, so this is self-contained). Every pair has NVFP4 ~1.56× above FP8 — the
spread is tiny (ratio 1.541–1.568×), which is why a large mean gap clears 95% significance in just 12 pairs.

| pair | A (NVFP4) t/s | B (FP8) t/s | A/B |
|--:|--:|--:|--:|
| 1 | 12.29 | 7.86 | 1.564× |
| 2 | 12.30 | 7.86 | 1.564× |
| 3 | 12.29 | 7.84 | 1.568× |
| 4 | 12.25 | 7.83 | 1.563× |
| 5 | 12.29 | 7.86 | 1.564× |
| 6 | 12.11 | 7.86 | 1.541× |
| 7 | 12.18 | 7.83 | 1.555× |
| 8 | 12.29 | 7.86 | 1.564× |
| 9 | 12.28 | 7.86 | 1.562× |
| 10 | 12.29 | 7.86 | 1.564× |
| 11 | 12.29 | 7.86 | 1.564× |
| 12 | 12.30 | 7.86 | 1.564× |
| **mean** | **12.26** | **7.85** | **1.56×** |

# Glossary
| term | meaning |
|---|---|
| **NVFP4 / FP8** | 4-bit vs 8-bit weight quantization. FP8 weights are ~2× the bytes → ~2× bandwidth per token. |
| **decode t/s** | steady-state output tokens/second; memory-bandwidth-bound at batch 1, so it scales inversely with weight bytes. |
| **A/B (paired)** | two endpoints measured interleaved, request-by-request, so shared drift cancels in the difference. |
| **Δ / 95% CI** | percent difference B-vs-A and its confidence interval; "significant" = CI excludes 0. |
| **MDE** | minimum detectable effect — the loop samples until CI half-width ≤ MDE (1.0%), then auto-stops. |
