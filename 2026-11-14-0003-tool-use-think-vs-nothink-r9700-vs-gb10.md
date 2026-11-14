# Report card — tool use, thinking on vs off: Qwen3.8-Flash-Next iq4r on radiance (2× R9700) vs NVFP4 on vLLM (GB10), Qwen3.8-27B as control

> **BFCL v4, non-live subset, via QualityBench** — 400 function-calling items, 100 each from `simple_python`,
> `multiple`, `parallel` and `parallel_multiple` (no live, irrelevance or multi-turn categories), scored by BFCL's own
> AST checker through the server's tool parser; greedy, seed 608, paired item by item.
> — [QualityBench](https://github.com/launch80/QualityBench)
> **tool-eval-bench `--hardmode`** — 104 scenario-scored multi-turn tool-use scenarios in 16 categories (selection,
> parameter precision, chains, restraint, error recovery, safety, state, structured output, Hard Mode), each
> pass 2 / partial 1 / fail 0. — [tool-eval-bench](https://github.com/SeraphimSerapis/tool-eval-bench)

**Why this card:** [card 0001](2026-11-14-0001-qwen38-27b-vs-flash-next-r9700-vs-gb10.md) found Flash-Next iq4r on
radiance tied with NVFP4 on vLLM on every task **except BFCL, −7.5 pts** — all of it in *parallel* calls, a decision
shift (one call where 2–6 were wanted), not a parser fault; [card 0002](2026-11-14-0002-flash-next-r9700-tp-wire-bfcl.md)
ruled out the TP wire. Those runs were **thinking off**. Agents run thinking **on**.

**What we ran on:** 2× AMD Radeon AI PRO R9700 (210 W cap each, PCIe 5.0 x8 each, Ryzen 9 9950X, 64 GB DDR5) ·
NVIDIA GB10 (DGX Spark). The R9700 arms ran one after another on the same cards, the GB10 arm in parallel on its own.

| arm | model, quant | engine | hardware |
|---|---|---|---|
| **27b** | Qwen3.8-27B, PARO int5 + DFlash2-FP8 drafter, TP2 | vLLM (tcclaviger 29.05.1) | 2× R9700 |
| **qfn-r9700** | Qwen3.8-Flash-Next, **iq4r**, MTP 3, TP2, `--tp-wire wht6` | **radiance 1.17.0** | 2× R9700 |
| **qfn-gb10** | Qwen3.8-Flash-Next, NVIDIA FP4-Local (NVFP4), MTP 4 | vLLM (eugr b12x nightly-20261015) | GB10 |

**Provenance (locked):** as card 0001 — radiance `1.17.0@sha256:c2acbaa373c8…` +
`StillDeadcode/qwen3.8-next-flash-fp8-iq4r-moe@8267a673`; `tcclaviger/vllm:29.05.1@sha256:cabd0947e156…` +
`Launch80/Qwen3.8-27B-PARO-int5@668cb0d7` + `tcclaviger/Qwen3.8-27B-DFlash2-FP8@ee0cb26a`;
`eugr/spark-vllm-b12x:nightly-20261015@sha256:03a956544e9f…` + `nvidia/Qwen3.8-Flash-Next-FP4-Local@b9ce18a6`.
Harness: QualityBench **v0.4.0** (`2f4f337`) `--tasks bfcl --concurrency 8`; tool-eval-bench **main `ea2a774`**
(2026-11-11, past v3.0.0) `--hardmode --seed 1234 --parallel 4`, temperature 0.
**Thinking off:** QualityBench `--thinking off`, tool-eval-bench `--no-think`. **Thinking on:** QualityBench
`--thinking on --thinking-max-tokens 32768 --request-param reasoning_effort=medium`; tool-eval-bench
`--backend-kwargs '{"reasoning_effort":"medium"}'` — the same effort on both engines.

> ⚠ **Single run per cell.** tool-eval-bench's 104 scenarios give wide intervals (±5 pts); read its paired verdicts,
> not the headline. Run-to-run spread seen on BFCL thinking-off: radiance 81.8 / 81.8 / 83.0, vLLM 89.2 / 87.5 / 87.5.

## Results

| | 27b off | 27b on | qfn-r9700 off | qfn-r9700 on | qfn-gb10 off | qfn-gb10 on |
|---|---|---|---|---|---|---|
| **BFCL v4 non-live** (n 400) | 92.0 [88.9, 94.3] | 91.8 [88.6, 94.1] | **83.0** [79.0, 86.4] | **90.8** [87.5, 93.2] | 87.5 [83.9, 90.4] | 91.2 [88.1, 93.6] |
| **tool-eval hardmode** | 87 (181/208) | **96** (199/208) | 89 (185/208) | 90 (188/208) | 89 (186/208) | 92 (191/208) |
| wall, BFCL + tool-eval | 8 min | 12 min | 4 min | 6 min | 16 min | 25 min |

**Thinking on vs off, paired within each arm:**

| arm | BFCL Δ [95% CI], McNemar p | tool-eval Δ [95% CI], McNemar p |
|---|---|---|
| 27b | −0.25 [−2.50, +2.00], p 1 | **+8.65 [+2.40, +15.38], p 0.0026** (17 improved, 4 regressed) |
| qfn-r9700 | **+7.75 [+4.25, +11.25], p 4.7e-5** | +1.44 [−4.33, +7.21], p 0.61 |
| qfn-gb10 | **+3.75 [+1.00, +6.50], p 0.011** | +2.40 [−3.37, +8.17], p 0.79 |

**qfn-r9700 vs qfn-gb10, paired (same model, different quant + engine):**

| | BFCL | tool-eval |
|---|---|---|
| thinking off | **−4.50 [−7.75, −1.50], p 0.0064** | −0.48 [−5.29, +4.33], p 0.75 |
| thinking on | −0.50 [−2.50, +1.50], p 0.82 | −1.44 [−6.25, +3.37], p 1.0 |

**qfn-r9700 vs 27b, paired:** off — BFCL **−9.00 [−12.50, −5.75], p 2.1e-7**, tool-eval +1.92 (p 0.45);
on — BFCL −1.00 [−3.50, +1.50] (p 0.54), tool-eval −5.29 [−10.58, 0.00] (p 0.12).

## Read in 20 seconds
- **Thinking closes the iq4r function-calling gap.** BFCL on radiance: 83.0 → **90.8** (+7.75, p 5e-5); against
  NVFP4 on GB10 the paired gap goes from **−4.5 (p 0.006)** to **−0.5 (p 0.82)**. The no-think parallel-call
  deficit is real but it is a *no-think* deficit.
- **On multi-turn tool use (tool-eval hardmode) the two Flash-Next builds are tied in both modes** (paired CIs
  through 0) — and radiance finished the work in **a quarter of the time** (6 vs 25 min thinking on).
- **The dense 27B gains most from thinking on multi-turn tool use** (+8.65, p 0.003 → 96/100, the best cell), and
  needs no thinking for BFCL (92 either way).
- For a tool-heavy agent at thinking-on: Flash-Next on 2× R9700 ≈ Flash-Next on GB10 in accuracy, ~4× faster.

## Reproduce
Serves as in card 0001. Then, per arm and mode:
```bash
qualitybench run --endpoint http://<server>:8000/v1 --model <served name> --tokenizer Qwen/Qwen3.8-Flash-Next \
  --tasks bfcl --concurrency 8 --thinking off --out off.json
qualitybench run --endpoint http://<server>:8000/v1 --model <served name> --tokenizer Qwen/Qwen3.8-Flash-Next \
  --tasks bfcl --concurrency 8 --thinking on --thinking-max-tokens 32768 --request-param reasoning_effort=medium --out on.json
uv tool install git+https://github.com/SeraphimSerapis/tool-eval-bench@ea2a77499f0fb371ac6653f5482d752ce4c37900
tool-eval-bench --base-url http://<server>:8000/v1 --model <served name> --hardmode --seed 1234 --parallel 4 --no-think --json-file te-off.json
tool-eval-bench --base-url http://<server>:8000/v1 --model <served name> --hardmode --seed 1234 --parallel 4 \
  --backend-kwargs '{"reasoning_effort":"medium"}' --json-file te-on.json
qualitybench compare off.json on.json          # think vs no-think, paired
tool-eval-bench compare <run_id_off> <run_id_on>
```

## What this card omits / caveats
- **iq4r only.** [`StillDeadcode/qwen3.8-next-flash-iqr-v4`](https://huggingface.co/StillDeadcode/qwen3.8-next-flash-iqr-v4)
  (2026-11-13) adds **tool-use transcripts to its calibration** — the obvious candidate for the no-think BFCL gap —
  but needs a radiance build newer than 1.17.0. The next card, when it ships.
- Power: the R9700s ran at a **210 W cap** (stock 300 W); speed here is not the hardware's ceiling.
- tool-eval-bench at an unreleased main commit (for its unparsed-tool-call diagnostics); scores are not comparable
  with v2.x or v3.0.0 runs (TC-21/89 regraded, 104 vs 84 scenarios).
- The think/no-think compares also change `reasoning_effort` (unset → medium); that is the setting agents use.
