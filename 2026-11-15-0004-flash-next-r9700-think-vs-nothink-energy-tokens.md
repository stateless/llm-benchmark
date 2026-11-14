# Report card — the whole suite, thinking on vs off, with wall energy and engine tokens: Qwen3.8-Flash-Next iq4r on radiance (2× R9700)

> **QualityBench** — *"how good a model's answers are and how much quality a quantized build or a different
> inference engine loses"*: a fixed, pinned suite, greedy, seed 608, paired item by item. — [QualityBench](https://github.com/launch80/QualityBench)
> **tool-eval-bench `--hardmode`** — 104 scenario-scored multi-turn tool-use scenarios; `--error-rate 0.2` injects
> 429/500/503 tool failures (the retry loop every agent runs). — [tool-eval-bench](https://github.com/SeraphimSerapis/tool-eval-bench)
> **BetterBench** — per-category decode speed. — [BetterBench](https://github.com/GGZ14/BetterBench)

**Why this card:** [card 0001](2026-11-14-0001-qwen38-27b-vs-flash-next-r9700-vs-gb10.md) ran the suite thinking
**off**; [card 0003](2026-11-14-0003-tool-use-think-vs-nothink-r9700-vs-gb10.md) ran only the tool tasks both ways.
This is the whole suite in both modes on one arm, and — new — **what each answer costs**: the energy at the wall
plug per step, and the tokens the engine itself counted per request.

**What we ran on:** 2× AMD Radeon AI PRO R9700 (gfx1201, 32 GB each, **210 W cap** per card, PCIe 5.0 x8 each),
Ryzen 9 9950X, 64 GB DDR5-6000. One arm, alone on the cards; the shared gateway's route to the model was withdrawn for the run.

| arm | model, quant | engine | hardware |
|---|---|---|---|
| **qfn-r9700** | Qwen3.8-Flash-Next, **iq4r** (4-bit codebook experts, int8 trunk), MTP 3, TP2, `--tp-wire wht6` | **radiance 1.17.0** | 2× R9700 |

**Provenance (locked):** as card 0001 — `docker.io/stilldeadcode/radiance:1.17.0@sha256:c2acbaa373c8…` (commit
`19d4e8fe`) · `StillDeadcode/qwen3.8-next-flash-fp8-iq4r-moe@8267a673` · upstream `flashnext.yaml` flags plus
`--gpu-headroom-mib 128 --reasoning-effort medium --default-max-tokens 32768`.
Harness: QualityBench **v0.4.0** (`2f4f337`), `--concurrency 8`, tokenizer `Qwen/Qwen3.8-Flash-Next@812d96e1`;
tool-eval-bench **main `ea2a774`** `--hardmode --seed 1234 --parallel 4 --no-live`, temperature 0;
BetterBench **v0.6.1** (`f72bb90`) `--runs 5 --seed 1234`.
**Thinking off:** QualityBench `--thinking off`, tool-eval-bench `--no-think`. **Thinking on:** QualityBench
`--thinking on --thinking-max-tokens 32768 --request-param reasoning_effort=medium`; tool-eval-bench
`--backend-kwargs '{"reasoning_effort":"medium"}'`.

> ⚠ **Single run per mode.** Run-to-run spread on the thinking-off column (this run vs card 0001's, same build):
> bfcl 82.8 / 81.8, lcb 45.7 / 44.0, humaneval 94.5 / 93.9, ifeval 85.8 / 85.0, mmlu_pro 77.6 / 78.0 — read moves of
> 1–2 pts as noise. A quick-tier config gate ran first (gsm8k, humaneval, bfcl, needle, thinking on): 4/4 pass.

## Results — quality (accuracy %, [95% CI], n)

| task | off | on |
|---|---|---|
| gsm8k | 96.7 [95.6, 97.6] (1319) | 97.0 [95.9, 97.8] (1319) |
| humaneval (+) | 94.5 [89.9, 97.1] (164) | 94.5 [89.9, 97.1] (164) |
| ifeval | 85.8 [82.6, 88.5] (541) | 89.6 [86.8, 91.9] (541) |
| mbpp (+) | 78.6 [74.2, 82.4] (378) | 82.3 [78.1, 85.8] (378) |
| mmlu_pro | 77.6 [74.9, 80.1] (1000) | **86.9** [84.7, 88.9] (1000) |
| lcb (v6 slice) | 45.7 [38.5, 53.1] (175) | **64.6** [57.2, 71.3] (175) |
| bfcl (v4 non-live) | 82.8 [78.7, 86.1] (400) | **90.8** [87.5, 93.2] (400) |
| needle 32k / 128k | 100% / 100% | 100% / 100% |
| tool-eval hardmode | 89 (186/208) | 92 (192/208) |

**Thinking on vs off, paired item by item** (Δ pts [95% CI], McNemar p):

| task | Δ on − off |
|---|---|
| **lcb** | **+18.86 [+12.57, +25.14], p 3.6e-8** |
| **mmlu_pro** | **+9.30 [+7.30, +11.30], p 4.3e-21** |
| **bfcl** | **+8.00 [+4.50, +11.75], p 4.2e-5** |
| **ifeval** | **+3.88 [+0.92, +6.84], p 0.017** |
| **mbpp** | **+3.70 [+1.32, +6.08], p 0.0043** |
| gsm8k | +0.23 [−0.61, +1.06], p 0.72 |
| humaneval | +0.00 [−3.05, +3.66], p 1 |
| tool-eval hardmode | +2.88 [−3.85, +9.62], p 0.81 (11 improved, 8 regressed) |
| tool-eval, injected failures | **+5.77 [+0.48, +11.54]**, McNemar p 0.18 (11 improved, 4 regressed) |

No task got worse with thinking on.

## Results — agent cost & risk (tool-eval-bench)

| mode | run | solved | wall per solved | median turns | tokens per solved | harmful | loops |
|---|---|---|---|---|---|---|---|
| off | tool-eval | 90/104 | 7 s | 3 | 7,027 | **4** | 0 |
| off | with injected failures | 80/104 | 8 s | 3 | 9,042 | **4** | 3 |
| on | tool-eval | 92/104 | 9 s | 3 | 7,452 | **4** | 0 |
| on | with injected failures | 86/104 | 11 s | 3 | 9,256 | **3** | 3 |

*Harmful* = a safety violation or a forbidden action; it should be 0, and is not in either mode.

## Results — energy and tokens

Energy is the **whole host at the wall plug** (both GPUs, CPU, PSU loss), 10 s samples, Wh = mean W × step length.
Tokens are the **engine's own per-request count**; generated includes reasoning.

| step | mode | wall | mean W | Wh | prompt tok (cached) | gen tok | of which reasoning | gen tok/s | J per gen tok | Wh per correct | gen tok per correct |
|---|---|---|---|---|---|---|---|---|---|---|---|
| quality (7 tasks + needle) | off | 65 min | 499 | 546 | 2,437,641 (21%) | 2,187,879 | 1,063 | 555 | 0.90 | **0.16** | 647 |
| quality (7 tasks + needle) | on | 160 min | 508 | 1,357 | 2,431,490 (22%) | 5,347,889 | 4,299,433 | 556 | 0.91 | **0.38** | 1,496 |
| tool-eval | off | 2.6 min | 495 | 21.6 | 599,809 (58%) | 41,238 | 227 | 263 | 1.89 | 0.24 per solved | |
| tool-eval | on | 3.5 min | 519 | 30.0 | 629,349 (61%) | 63,133 | 23,479 | 304 | 1.71 | 0.33 per solved | |
| with injected failures | off | 2.7 min | 522 | 23.6 | 673,751 (60%) | 41,545 | 201 | 255 | 2.05 | 0.30 per solved | |
| with injected failures | on | 3.9 min | 516 | 33.4 | 714,201 (59%) | 69,046 | 26,878 | 296 | 1.74 | 0.39 per solved | |
| speed (BetterBench, 1 stream) | — | 9.7 min | 508 | 82.0 | 924,037 (0%) | 107,360 | 72,025 | 185 | 2.75 | | |

Decode, single stream (BetterBench weighted median): **189.5 tok/s, TTFT 61 ms**. The host draws **~500 W whether
it serves one stream or eight**, so J per generated token is set by the batch: 0.90 at 8 concurrent requests,
2.75 at one. The tool and speed rows also carry 9–15 prompt tokens per generated token; their J per gen tok
charges the prefill to the generated tokens. The prefix cache served ~60% of the tool steps' prompt tokens (each
turn resends the conversation) and ~21% of the quality steps'; the speed step's prompts were never cached.

## Read in 20 seconds
- **Thinking buys accuracy where reasoning matters, and costs 2.4×.** lcb +19 pts, mmlu_pro +9, bfcl +8, ifeval +4,
  mbpp +4; nothing on gsm8k or humaneval (already at the ceiling). Generated tokens ×2.44, wall time ×2.46,
  energy ×2.48 — the three move together because throughput does not change (556 vs 555 tok/s).
- **Per correct answer: 0.16 Wh thinking off, 0.38 Wh thinking on** — on 2× R9700 at a 210 W cap, the whole box.
- **No task got worse; five got significantly better.** Tool-eval's +2.9 is within noise; with injected tool
  failures thinking helps more (+5.8, CI above 0).
- **For agents, thinking is cheap.** tool-eval tokens per solved +6% (7,027 → 7,452): prompts dominate each turn,
  and at medium effort the model reasons ~70 tokens per tool turn. With injected failures it solves 86 vs 80. **4 harmful actions in either mode** (3 on, with failures).
- **Energy per token is a batching number:** ~0.9 J per generated token at concurrency 8, ~2.75 J single stream.

## Reproduce
Serve as in card 0001. Then, per mode:
```bash
qualitybench run --endpoint http://<server>:8000/v1 --model <served name> --tokenizer Qwen/Qwen3.8-Flash-Next \
  --tasks gsm8k,ifeval,mmlu_pro,humaneval,mbpp,bfcl,lcb,needle --concurrency 8 --thinking off --out off.json
qualitybench run --endpoint http://<server>:8000/v1 --model <served name> --tokenizer Qwen/Qwen3.8-Flash-Next \
  --tasks gsm8k,ifeval,mmlu_pro,humaneval,mbpp,bfcl,lcb,needle --concurrency 8 \
  --thinking on --thinking-max-tokens 32768 --request-param reasoning_effort=medium --out on.json
uv tool install git+https://github.com/SeraphimSerapis/tool-eval-bench@ea2a77499f0fb371ac6653f5482d752ce4c37900
tool-eval-bench --base-url http://<server>:8000/v1 --model <served name> --hardmode --seed 1234 --parallel 4 --no-live --no-think --json-file te-off.json
tool-eval-bench --base-url http://<server>:8000/v1 --model <served name> --hardmode --seed 1234 --parallel 4 --no-live \
  --backend-kwargs '{"reasoning_effort":"medium"}' --json-file te-on.json
# the same two with --error-rate 0.2 for the injected-failure rows
qualitybench compare off.json on.json
```
Energy: any metering plug on the host, averaged over each step's window. Tokens: radiance logs one line per finished
request (`done … prompt= (cache ) gen= reason= ttft=`); sum them over each step's window.

## What this card omits / caveats
- **One arm.** The same table for Flash-Next NVFP4 on GB10 and the dense 27B needs a token source for vLLM
  (it writes no per-request lines).
- **The energy is per step, not per phase.** 10 s plug samples cannot split prefill from generation inside a
  concurrency-8 run; the J per gen token column charges everything (prefill, idle gaps between requests, the CPU)
  to generated tokens.
- Thinking off is not zero thinking: 1,063 reasoning tokens over 4,008 requests.
- Power: the R9700s run at a **210 W cap** (stock 300 W); the host's ~500 W is the whole box.
- tool-eval-bench at an unreleased main commit; scores are not comparable with v2.x or v3.0.0 runs.
- The think/no-think compares also change `reasoning_effort` (unset → medium), the setting agents use; both tools
  flag the configurations as different, which is the point of the comparison.
