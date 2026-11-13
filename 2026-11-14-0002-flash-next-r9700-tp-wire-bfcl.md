# Report card — QualityBench BFCL A/B: radiance's lossy TP wire (`wht6`) vs exact, Qwen3.8-Flash-Next on 2× R9700

> **The question:** the previous card found Qwen3.8-Flash-Next iq4r on radiance **−7.5 pts on BFCL** against the
> same model as NVFP4 on vLLM, with every other task tied. radiance's own guide measured its lossy all-reduce wire
> (`--tp-wire wht6`) moving *tool-calling transcripts* the most of any content (mean KL 0.17–0.27, top-1 91–93%, vs
> 0.0004–0.007 for code). `wht6` is exact only for steps of ≤ 25 rows; BFCL is long tool-schema prefill at
> concurrency 8. Is the wire the cause?

**What we ran on:** 2× AMD Radeon AI PRO R9700 (210 W cap each) and NVIDIA GB10, as in
[the previous card](2026-11-14-0001-qwen38-27b-vs-flash-next-r9700-vs-gb10.md). **One variable** on the R9700 side:
`--tp-wire wht6` vs `--tp-wire exact` — every other flag identical (diffed). Confirmed from the startup log: the
`wht6` serve prints `tp wire: LOSSY`, the exact one does not.

**Provenance (locked):** radiance 1.17.0 `@sha256:c2acbaa373c8…`, `StillDeadcode/qwen3.8-next-flash-fp8-iq4r-moe@8267a673`;
GB10: eugr b12x nightly-20261015 `@sha256:03a956544e9f…`, `nvidia/Qwen3.8-Flash-Next-FP4-Local@b9ce18a6`. QualityBench
v0.4.0 (`2f4f337`), `--tasks bfcl --thinking off --concurrency 8`.

## Results — BFCL (400 items)

| arm | BFCL | gap to GB10 |
|---|---|---|
| R9700, `wht6` | 81.8 [77.7, 85.2] | −5.75 |
| R9700, **exact** | 83.2 [79.3, 86.6] | −4.25 |
| GB10 (NVFP4, vLLM) | 87.5 [83.9, 90.4] | — |

**Paired:**

| comparison | Δ pts [95% CI] | McNemar p |
|---|---|---|
| exact vs `wht6` | +1.50 [−1.75, +4.75] | 0.47 — **noise** |
| `wht6` vs GB10 | −5.75 [−9.25, −2.50] | 0.0014 |
| exact vs GB10 | −4.25 [−7.50, −1.00] | 0.019 |

## Read in 20 seconds
- **The wire is not the cause.** Exact vs lossy: +1.5 pts, p 0.47. Both R9700 serves stay significantly behind GB10.
- What remains is the **iq4r quantisation** (or radiance's numerics) — most plausibly the 4-bit codebook experts; the
  int8 trunk is identical in both serves. The model card reports quality on *text turns* only; a calibration set
  without tool-call transcripts would hurt exactly this.
- **Determinism:** radiance at temperature 0 repeated its BFCL **exactly** (81.8, both runs); vLLM at concurrency 8
  did not (89.2 → 87.5). The GB10 side carries ±2 pts of run-to-run spread; the gap survives it.

## Reproduce
As the previous card, with `--tp-wire exact` in place of `--tp-wire wht6` for the second R9700 serve, and
`qualitybench run … --tasks bfcl --thinking off --concurrency 8` against each.

## What this card omits / caveats
- **Thinking off** only. Tool use with thinking on is a separate card.
- Exact-wire *prefill* speed was not measured here (radiance's guide says the lossy wire exists for faster prefill);
  single-stream decode is exact on both, and measured the same (219.8 vs 218.8 tok/s greedy).
