# Report card — energy per correct answer at the wall: Qwen3.8-Flash-Next on 2× R9700 vs GB10, and Qwen3.8-27B (thinking off)

> **QualityBench** — *"how good a model's answers are and how much quality a quantized build or a different
> inference engine loses"*: a fixed, pinned suite, greedy, seed 608, paired item by item. — [QualityBench](https://github.com/launch80/QualityBench)
> **BetterBench** — per-category decode speed. — [BetterBench](https://github.com/GGZ14/BetterBench)

**Why this card:** [card 0001](2026-11-14-0001-qwen38-27b-vs-flash-next-r9700-vs-gb10.md) compared quality and speed.
This is **the same run, costed**: each host's wall plug was sampled every 10 s during it, and the samples were read
back afterwards for each step's window. Nothing was re-run. **Preliminary:** thinking off only. A GB10 run with
thinking on is in progress; this card will be updated with it (the R9700 box's thinking-on numbers are in
[card 0004](2026-11-15-0004-flash-next-r9700-think-vs-nothink-energy-tokens.md)).

**What we ran on, arms, provenance:** exactly as card 0001.
- **2× AMD Radeon AI PRO R9700** (210 W cap per card), Ryzen 9 9950X, 64 GB DDR5-6000: arms **27b** (vLLM, PARO int5 +
  DFlash2 drafter) and **qfn-r9700** (radiance 1.17.0, iq4r, MTP 3).
- **NVIDIA GB10** (DGX Spark, 128 GB unified): arm **qfn-gb10** (vLLM, NVIDIA FP4-Local, MTP 4).
- QualityBench v0.4.0, `--thinking off --concurrency 8`, the 7 scored tasks + needle; BetterBench v0.6.1 `--runs 5 --seed 1234`.

> ⚠ **The whole box at the wall plug** (GPUs, CPU, memory, fans, PSU loss), not GPU power. **Single run per arm.**
> Thinking off. Both hosts capped at 8 concurrent sequences.

## Results — energy per step

| arm | step | wall | mean W | Wh | samples (10 s) |
|---|---|---|---|---|---|
| qfn-gb10 | quality (7 tasks + needle) | 296 min | **96** | **474.9** | 1781 (full) |
| qfn-r9700 | quality | 66 min | **497** | **547.3** | 397 (full) |
| 27b | quality | 148 min | 592 | 1,466.1 | 891 (full) |
| qfn-gb10 | speed (BetterBench) | 36 min | 95 | 57.4 | 218 |
| qfn-r9700 | speed | 9 min | 511 | 82.6 | 58 |
| 27b | speed | 15 min | 581 | 152.2 | 94 |

## Results — per correct answer (quality, 3,977 scored items)

| arm | correct | **Wh per correct** | J per correct | wall | relative energy | relative time |
|---|---|---|---|---|---|---|
| qfn-gb10 | 3,397 | **0.140** | 503 | 296 min | **0.87×** | 4.5× |
| qfn-r9700 | 3,376 | **0.162** | 584 | 66 min | 1 | 1 |
| 27b (R9700) | 3,337 | 0.439 | 1,582 | 148 min | 2.7× | 2.2× |

Correct = card 0001's per-task accuracy × n (each ±1 item from rounding); the two QFN arms are within 21 answers of
each other (almost all of it bfcl), so the energy gap is the energy, not the score.

**Repeatability:** qfn-r9700's thinking-off quality step, run again the next night (card 0004), used **546 Wh**
(here 547.3) — the R9700 box repeats to within 0.3%. The GB10's repeat is the run in progress.

## Read in 20 seconds
- **GB10 answers for ~13% less energy, at 4.5× the time.** 0.140 vs 0.162 Wh per correct answer. The R9700 box draws
  ~500 W and finishes in 66 min; the GB10 draws 96 W for 296 min.
- **A baseline is half of the GB10's draw.** It drew 47–51 W while only loading a model and 96 W under load, so being
  slower costs it that baseline the whole time. The R9700 box's draw is nearly flat with load (~500 W), so its lever is batching; the
  GB10's is finishing sooner.
- **The same model, two boxes, about the same energy per answer**; the dense 27B on the R9700s costs 2.7× QFN's.
- **Speed test, same work:** GB10 57 Wh, R9700 83 Wh, 27b 152 Wh.
- In money, the gap is small: 0.07 kWh over the whole 4,000-question suite.

## Reproduce
Run card 0001's commands. Energy: any metering plug on each host, sampled during the run; per step, Wh = mean W ×
the step's length (from the step's start and end times); per correct = Wh / items answered correctly.

## What this card omits / caveats
- **One run per arm.** The R9700 figure repeated; the GB10's has not yet.
- **Thinking off only.** With thinking on the R9700 box spent 2.5× the energy per correct answer (card 0004); the
  GB10's thinking-on number comes with the update.
- **Different builds of the same model** (iq4r on radiance vs NVFP4 on vLLM): this compares the two stacks as
  deployed, not the silicon alone.
- **8 concurrent sequences on both.** The GB10 was not tried at higher concurrency; more batching would lower its
  idle share.
- 10 s plug samples: per step, not per phase — prefill, decode and idle gaps are all in one figure.
- No token counts for the vLLM arms (vLLM writes no per-request lines), so no J per token for them here.
