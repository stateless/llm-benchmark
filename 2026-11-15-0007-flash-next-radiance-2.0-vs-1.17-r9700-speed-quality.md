# Report card — BetterBench + QualityBench: radiance 2.0.0 + IQR-V4 vs radiance 1.17.0 + iq4r, Qwen3.8-Flash-Next on 2× R9700 (speed + quality)

> **BetterBench** — per-category decode speed, a prefill sweep and a concurrency sweep. — [BetterBench](https://github.com/GGZ14/BetterBench)
>
> **QualityBench** — a fixed, pinned suite, greedy, seed 608, paired item by item. — [QualityBench](https://github.com/launch80/QualityBench)

**Why this card:** [card 0006](2026-11-15-0006-flash-next-radiance-2.0-vs-1.17-r9700-speed-energy.md) measured the speed of the radiance 2.0.0 upgrade. Does the new engine with its new Flash-Next container (IQR-V4) also keep the answers
the same? This card adds QualityBench, thinking off and on, against [card 0004](2026-11-15-0004-flash-next-r9700-think-vs-nothink-energy-tokens.md)'s 1.17 run on the same box, with time and wall energy for each.

**What we ran on:** 2× AMD Radeon AI PRO R9700 (gfx1201, 32 GB each, **210 W cap** per card, PCIe 5.0 x8 each, peer-to-peer on), Ryzen 9 9950X, 64 GB DDR5-6000, Linux 7.2.6. One arm at a time, alone on the cards; the shared gateway's route to the host withdrawn during each run.

| arm | engine | container | KV | config |
|---|---|---|---|---|
| **1.17** | radiance **1.17.0** | `qwen3.8-next-flash-fp8-iq4r-moe` | fp8 | CLI flags (card 0001) |
| **2.0** | radiance **2.0.0** | `qwen3.8-next-flash-iqr-v4` | **i8** | YAML, the same settings |

**Provenance (locked):**
- 1.17: `docker.io/stilldeadcode/radiance:1.17.0@sha256:c2acbaa373c8…` · `StillDeadcode/qwen3.8-next-flash-fp8-iq4r-moe@8267a673`.
- 2.0: `docker.io/stilldeadcode/radiance:2.0.0@sha256:5ee52dc14c77…` (tag v2.0.0 = `ddcb9c6`) ·
  `StillDeadcode/qwen3.8-next-flash-iqr-v4@679c9432` (113.3 GiB).
- Both: TP2, `tp-wire wht6`, 8 sequences, 200000 context, `placement expert_tiered`, 24 GiB pinned host pool in all,
  4 GiB host prefix tier, MTP 3, 2048-token prefill chunk, `reasoning-effort medium`, `default-max-tokens 32768`.
- BetterBench **v0.6.1** (`f72bb90`), `--runs 5 --seed 1234`, corpus v1.0, temperature 0.7, cold prefix cache.
- QualityBench **v0.4.0** (`2f4f337`), `--concurrency 8`, tokenizer `Qwen/Qwen3.8-Flash-Next@812d96e1`; thinking on =
  `--thinking-max-tokens 32768`, `reasoning_effort=medium`. The same tasks, items and flags for both arms.

> ⚠ **Engine AND quant changed together** — this measures the upgrade as shipped, not either part alone. 
> Runs a day apart, not interleaved: QualityBench deltas are paired item by item; BetterBench's are not paired in time.

## Results — quality (QualityBench, paired 2.0 − 1.17)

| task (n) | off: 1.17 → 2.0 | Δ off [95% CI] | on: 1.17 → 2.0 | Δ on [95% CI] |
|---|---|---|---|---|
| gsm8k (1319) | 96.7 → 97.0 | +0.2 [−0.3, +0.8] | 97.0 → 97.2 | +0.2 [−0.4, +0.9] |
| ifeval (541) | 85.8 → 86.0 | +0.2 [−2.0, +2.4] | 89.6 → 90.0 | +0.4 [−1.9, +2.6] |
| mmlu_pro (1000) | 77.6 → 77.8 | +0.2 [−1.3, +1.8] | 86.9 → 86.4 | −0.5 [−1.8, +0.8] |
| humaneval (164) | 94.5 → 92.7 | −1.8 [−4.3, 0.0] | 94.5 → 95.7 | +1.2 [−1.2, +3.7] |
| mbpp (378) | 78.6 → 79.6 | +1.1 [−0.5, +2.9] | 82.3 → 82.5 | +0.3 [−1.3, +1.9] |
| bfcl (400) | 82.8 → 82.5 | −0.3 [−3.8, +3.3] | 90.8 → 90.5 | −0.3 [−2.0, +1.8] |
| lcb (175) | 45.7 → 46.3 | +0.6 [−4.0, +5.1] | 64.6 → 65.1 | +0.6 [−3.4, +4.0] |
| needle 32k / 128k | 100 / 100 → 100 / 100 | — | 100 / 100 → 100 / 100 | — |

**No task moved in either mode:** every McNemar p ≥ 0.25, and QualityBench's per-task no-regression test passes 7/7
in both modes. The largest move, humaneval −1.8 thinking off, is 3 of 164 problems and reverses with thinking on.

## Results — speed

Single stream (BetterBench, per-category median decode tok/s; all eight categories significant):

| | 1.17 | 2.0 | Δ |
|---|---|---|---|
| weighted decode | 189.5 | **238.9** | **+26%** (per category +19% to +31%) |
| aggregate, 1 / 8 streams | 174.0 / 495.1 | 216.0 / 533.9 | +24% / +8% |
| prefill, 2k / 64k cold | 4,728 / 6,424 | 5,256 / 6,449 | +11% / 0% |

The quality suite (8 requests at a time; the engine's own token count — the same work in each pair):

| | off: 1.17 → 2.0 | Δ | on: 1.17 → 2.0 | Δ |
|---|---|---|---|---|
| wall | 65.7 → 62.0 min | −6% | 160.3 → 145.0 min | −10% |
| generated tokens | 2.19 M → 2.19 M | — | 5.35 M → 5.24 M | −2% |
| generation rate | 555 → 588 tok/s | +6% | 556 → 602 tok/s | +8% |
| energy at the wall | 546.3 → 508.8 Wh | −7% | 1356.7 → 1216.5 Wh | −10% |
| **Wh per correct answer** | 0.162 → **0.150** | **−7%** | 0.38 → **0.34** | **−10%** |

## Read in 20 seconds
- **Quality unchanged:** no QualityBench task moved, thinking off or on.
- **Faster, most at one stream:** +26% single-stream decode; +6–8% on the quality suite at 8 requests at a time —
  what the concurrency sweep (+8% at 8 streams) predicts.
- **Less energy per correct answer:** −7% thinking off, −10% thinking on, at the same mean power (~500 W).
- **Thinking on costs the same on both engines:** ~2.3× the time and ~2.4× the energy, for lcb +19 pts,
  mmlu_pro +9, bfcl +8, ifeval +4.

## Reproduce
Serve 2.0 with the YAML entry in card 0006. Then:
```bash
uv tool install git+https://github.com/GGZ14/BetterBench@f72bb9011a830fd98e82ab89d01e549efa76e9c5
betterbench run --endpoint http://<server>:8000/v1 --model flashnext --runs 5 --seed 1234 --out v200.json
betterbench compare v117.json v200.json
qualitybench run --endpoint http://<server>:8000/v1 --model flashnext --tokenizer Qwen/Qwen3.8-Flash-Next \
  --tasks gsm8k,ifeval,mmlu_pro,humaneval,mbpp,bfcl,lcb,needle --concurrency 8 --thinking off --out v200.off.json
qualitybench run --endpoint http://<server>:8000/v1 --model flashnext --tokenizer Qwen/Qwen3.8-Flash-Next \
  --tasks gsm8k,ifeval,mmlu_pro,humaneval,mbpp,bfcl,lcb,needle --concurrency 8 \
  --thinking on --thinking-max-tokens 32768 --request-param reasoning_effort=medium --out v200.on.json
qualitybench compare v117.off.json v200.off.json
qualitybench compare v117.on.json v200.on.json
```
Energy: any metering plug on the host, averaged over each run's window. Tokens: radiance logs one line per finished
request (`done … prompt= (cache ) gen= reason=`); sum them over each run's window.

## What this card omits / caveats
- **Engine and quant moved together.** The IQR-V4 container needs a build newer than 1.17.0; splitting the gain
  would need 2.0 with the old container, which was not tried.
- **One QualityBench run per arm and mode** (greedy; not repeated here).
- The author's card reports KL vs bf16 on model-written text 0.0242 for IQR-V4 (iq4r 0.0279) with this wire; at
  QualityBench's resolution the two score the same.
- **Room cooling, a side factor, is not in the energy figures.** A hand-set room aircon ran during the 2.0 quality runs
  (off since the hour before): 559 Wh during thinking off, 981 Wh during thinking on — about as much again as the
  box itself. It follows the room and the operator's setting, not the engine, and was not read for the 1.17 runs, so
  it is not compared.
- 210 W cap per card; the CPU governor was `powersave` (not tuned here).
