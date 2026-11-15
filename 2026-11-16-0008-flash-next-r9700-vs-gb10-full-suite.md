# Report card — Qwen3.8-Flash-Next, the whole suite: 2× R9700 (radiance 1.17 and 2.0) vs GB10 (vLLM), thinking off and on — quality, tool use, speed, energy

> **QualityBench** — a fixed, pinned suite, greedy, seed 608, paired item by item (McNemar p, paired-bootstrap 95% CI). — [QualityBench](https://github.com/launch80/QualityBench)
>
> **tool-eval-bench** — 104 agentic tool-use scenarios, `--hardmode`; scored 0–100, with harmful actions counted. — [tool-eval-bench](https://github.com/SeraphimSerapis/tool-eval-bench)
>
> **BetterBench** — per-category decode speed, a prefill sweep and a concurrency sweep. — [BetterBench](https://github.com/GGZ14/BetterBench)

**Why this card:** cards [0001](2026-11-14-0001-qwen38-27b-vs-flash-next-r9700-vs-gb10.md), [0003](2026-11-14-0003-tool-use-think-vs-nothink-r9700-vs-gb10.md), [0004](2026-11-15-0004-flash-next-r9700-think-vs-nothink-energy-tokens.md), [0005](2026-11-15-0005-energy-per-correct-r9700-vs-gb10-nothink.md), [0006](2026-11-15-0006-flash-next-radiance-2.0-vs-1.17-r9700-speed-energy.md) and [0007](2026-11-15-0007-flash-next-radiance-2.0-vs-1.17-r9700-speed-quality.md) each answered one question. This card puts them in one place and adds the last missing piece: **the GB10 with thinking on**, the whole suite, with the tool steps and wall energy. Same model, two boxes, three stacks — compared as deployed.

**What we ran on:**
- **2× AMD Radeon AI PRO R9700** (gfx1201, 32 GB each, **210 W cap** per card, PCIe 5.0 x8 each, peer-to-peer on), Ryzen 9 9950X, 64 GB DDR5-6000, Linux 7.2.6.
- **NVIDIA GB10** (DGX Spark, sm_121, 128 GB unified), driver 580.173.02, kernel 6.17.0-1029.

Each arm ran alone on its box (nothing else served; the shared gateway's route to the box withdrawn for the run).

| arm | model, quant | engine | box |
|---|---|---|---|
| **gb10** | Qwen3.8-Flash-Next, NVIDIA **FP4-Local** (modelopt mixed: FP4 experts) | **vLLM** (eugr b12x nightly-20261015) | GB10 |
| **r9700 1.17** | Qwen3.8-Flash-Next, **iq4r** (4-bit codebook experts, int8 trunk) | **radiance 1.17.0** | 2× R9700 |
| **r9700 2.0** | Qwen3.8-Flash-Next, **IQR-V4** (4-bit trellis/codebook experts, int8 trunk) | **radiance 2.0.0** | 2× R9700 |

## Model settings (locked)

| | gb10 | r9700 1.17 | r9700 2.0 |
|---|---|---|---|
| image | `docker.io/eugr/spark-vllm-b12x:nightly-20261015@sha256:03a956544e9f…` | `docker.io/stilldeadcode/radiance:1.17.0@sha256:c2acbaa373c8…` (commit `19d4e8fe`) | `docker.io/stilldeadcode/radiance:2.0.0@sha256:5ee52dc14c77…` (tag v2.0.0 = `ddcb9c6`) |
| weights | `nvidia/Qwen3.8-Flash-Next-FP4-Local@b9ce18a6` | `StillDeadcode/qwen3.8-next-flash-fp8-iq4r-moe@8267a673` (113.6 GiB) | `StillDeadcode/qwen3.8-next-flash-iqr-v4@679c9432` (113.3 GiB) |
| parallelism | TP 1 (one GPU) | TP 2, `tp-wire wht6` (lossy all-reduce) | TP 2, `tp-wire wht6` |
| context | 262,144 | 200,000 | 200,000 |
| sequences | 8 | 8 | 8 |
| KV cache | fp8, block 16, prefix caching | fp8, + 4 GiB host prefix tier | **i8** (fp8 KV is gone in 2.0), + 4 GiB host prefix tier |
| memory | `--gpu-memory-utilization 0.8` | `expert_tiered`, 12 GiB pinned host pool **per card** (24 GiB), headroom 128 MiB | `expert_tiered`, 24 GiB pinned host pool **in all**, headroom 160 MiB |
| speculative | MTP, 4 tokens | MTP 3 | MTP 3 |
| prefill chunk | 4,096 (chunked prefill) | 2,048 | 2,048 |
| kernels | `--load-format b12x`, `--moe-backend b12x`, `--linear-backend b12x`, `--gdn-decode-kernel b12x`, `--mamba-cache-mode align`, `--dtype bfloat16`, fused act-quant compile pass | in the image (libr4d) | in the image; chained launches for the decode step |
| reasoning / tools | `--reasoning-parser qwen3`, `--tool-call-parser qwen3_xml`, auto tool choice, the shipped chat template | built in (`reasoning_content`, OpenAI `tool_calls`) | built in |
| server defaults | `reasoning_effort` **medium** (chat-template kwarg); reply cap 32,768; sampling = the checkpoint's own (temp 1.0, top_p 0.95, top_k 20), no penalties | `--reasoning-effort medium`, `--default-max-tokens 32768` | `reasoning-effort medium`, `default-max-tokens 32768` |
| config form | `vllm serve` flags | CLI flags (upstream `flashnext.yaml`) | YAML (upstream `tp2.radiance.yaml` `flashnext` entry; host pool 24 GiB, not 16) |

The server defaults matter only where a client leaves a value out: every harness here sends its own sampling
(QualityBench is greedy) and, thinking on, `reasoning_effort=medium` explicitly.

**Harness (the same for every arm):**
- QualityBench **v0.4.0** (`2f4f337`): tasks gsm8k, ifeval, mmlu_pro, humaneval, mbpp, bfcl, lcb, needle;
  `--concurrency 8`; tokenizer `Qwen/Qwen3.8-Flash-Next@812d96e1`. Thinking off = `--thinking off`; thinking on =
  `--thinking on --thinking-max-tokens 32768 --request-param reasoning_effort=medium`.
- tool-eval-bench **main `ea2a774`** (3.0.1.dev15), `--hardmode --seed 1234 --parallel 4 --no-live`, temperature 0;
  thinking off `--no-think`, thinking on `--backend-kwargs '{"reasoning_effort":"medium"}'`; the "with failures" run
  adds `--error-rate 0.2` (injected 429/500/503 tool errors).
- BetterBench **v0.6.1** (`f72bb90`), `--runs 5 --seed 1234`, corpus v1.0, temperature 0.7, cold prefix cache.
- A quick gate first (QualityBench `--quick` gsm8k, humaneval, bfcl, needle, thinking on): gb10 and r9700 1.17 pass;
  r9700 2.0's quality runs had none (a single-arm run).

> ⚠ **Different builds of the same model on different engines** — this compares the stacks as deployed, not the
> silicon. **One run per arm and mode**, a day or two apart (QualityBench deltas are paired item by item; BetterBench's
> are not paired in time). The r9700 2.0 arm has **no tool-use run** yet.

## Results — quality (accuracy %, n)

| task (n) | off: gb10 | off: 1.17 | off: 2.0 | on: gb10 | on: 1.17 | on: 2.0 |
|---|---|---|---|---|---|---|
| gsm8k (1319) | 97.1 | 96.7 | 97.0 | 96.8 | 97.0 | 97.2 |
| ifeval (541) | 84.7 | 85.8 | 86.0 | 87.2 | 89.6 | 90.0 |
| mmlu_pro (1000) | 78.2 | 77.6 | 77.8 | 87.0 | 86.9 | 86.4 |
| humaneval (164) | 93.9 | 94.5 | 92.7 | 94.5 | 94.5 | 95.7 |
| mbpp (378) | 78.8 | 78.6 | 79.6 | 81.5 | 82.3 | 82.5 |
| **bfcl (400)** | **87.8** | 82.8 | 82.5 | 91.2 | 90.8 | 90.5 |
| lcb (175) | 47.4 | 45.7 | 46.3 | 64.0 | 64.6 | 65.1 |
| needle 32k / 128k | 100 / 100 | 100 / 100 | 100 / 100 | 100 / 100 | 100 / 100 | 100 / 100 |
| **correct of 4,177** | 3,407 | 3,378 | 3,386 | 3,559 | 3,575 | 3,578 |

**Paired, r9700 − gb10** (pts [95% CI], McNemar p; bold = p < 0.05):

| task | off: 1.17 | off: 2.0 | on: 1.17 | on: 2.0 |
|---|---|---|---|---|
| gsm8k | −0.2 [−0.6, +0.2] | −0.4 [−0.8, +0.1] | +0.4 [−0.3, +1.1] | +0.2 [−0.6, +0.9] |
| ifeval | +1.1 [−1.3, +3.5] | +1.3 [−1.1, +3.9] | +2.4 [−0.2, +4.8] | **+2.8** [+0.4, +5.2] p 0.04 |
| mmlu_pro | −0.6 [−2.2, +0.9] | −0.4 [−2.0, +1.2] | −0.1 [−1.3, +1.1] | −0.6 [−1.9, +0.7] |
| humaneval | +0.6 [−1.2, +2.4] | −1.2 [−3.7, +1.2] | 0.0 [−1.8, +1.8] | +1.2 [−1.2, +3.7] |
| mbpp | −0.3 [−1.6, +0.8] | +0.8 [−0.8, +2.7] | +0.8 [−1.1, +2.7] | +1.1 [−0.8, +2.9] |
| **bfcl** | **−5.0** [−8.3, −2.0] p 0.004 | **−5.3** [−8.8, −2.0] p 0.005 | −0.5 [−2.3, +1.3] | −0.8 [−2.8, +1.3] |
| lcb | −1.7 [−6.3, +2.9] | −1.1 [−5.7, +3.4] | +0.6 [−4.0, +5.1] | +1.1 [−2.9, +5.1] |

Thinking off, the R9700 stacks lose BFCL by 5 pts on both engine versions (QualityBench's no-regression test fails on
that task alone); **thinking on, the gap is gone** and all 7 tasks pass. Card 0001 traced the thinking-off gap to
parallel calls: one call made where 2–6 were wanted.

**Thinking on − off, paired, per arm** (each arm against itself):

| task | gb10 | r9700 1.17 | r9700 2.0 |
|---|---|---|---|
| gsm8k | −0.3 [−1.1, +0.5] | +0.2 [−0.6, +1.1] | +0.2 [−0.5, +1.1] |
| ifeval | +2.6 [−0.6, +5.9] | **+3.9** [+0.9, +6.8] p 0.017 | **+4.1** [+1.3, +6.7] p 0.005 |
| mmlu_pro | **+8.8** [+6.7, +10.9] p 2e-16 | **+9.3** [+7.3, +11.3] p 4e-21 | **+8.6** [+6.6, +10.6] p 2e-17 |
| humaneval | +0.6 [−3.1, +4.3] | 0.0 [−3.1, +3.7] | +3.1 [−0.6, +7.3] |
| mbpp | +2.6 [0.0, +5.3] | **+3.7** [+1.3, +6.1] p 0.004 | **+2.9** [+0.5, +5.6] p 0.04 |
| bfcl | **+3.5** [+0.5, +6.5] p 0.03 | **+8.0** [+4.5, +11.8] p 4e-5 | **+8.0** [+4.5, +11.8] p 3e-5 |
| lcb | **+16.6** [+10.3, +23.4] p 2e-6 | **+18.9** [+12.6, +25.1] p 4e-8 | **+18.9** [+13.1, +25.1] p 2e-9 |

Bold = McNemar p < 0.05. Every arm passes QualityBench's no-regression test 7/7: thinking on loses nothing anywhere.
The GB10 gains less on bfcl because it starts higher thinking off; thinking on, all three land at 90.5–91.2.

## Results — tool use (tool-eval-bench, hardmode)

| arm · mode | score | solved | wall per solved | tokens per solved | **harmful** | with 20% injected failures: solved · harmful · loops |
|---|---|---|---|---|---|---|
| gb10 · off | 89/100 | 90/104 | 23 s | 7,080 | **4** | 79 · **6** · 3 |
| r9700 1.17 · off | 89/100 | 90/104 | 7 s | 7,027 | **4** | 80 · **4** · 3 |
| gb10 · on | 92/100 | 92/104 | 32 s | 7,437 | **2** | 80 · **3** · 3 |
| r9700 1.17 · on | 92/100 | 92/104 | 9 s | 7,452 | **4** | 86 · **3** · 3 |

Scores tie; the per-scenario pairing (tool-eval-bench `compare`, by run id) needs the runner's run history and was
not made for this card. Earlier paired tool-eval compares of the same two stacks: card 0003.

## Results — speed (BetterBench, single stream)

| | gb10 | r9700 1.17 | r9700 2.0 |
|---|---|---|---|
| weighted decode, tok/s | 50.3 | 189.5 | **238.9** |
| TTFT p50, 1 stream | 300 ms | 67 ms | 63 ms |

Per category (median decode tok/s; r9700 2.0 vs gb10 significant in all eight):

| category | gb10 | r9700 2.0 | Δ |
|---|--:|--:|--:|
| chat | 38.6 | 204.1 | +406% |
| code | 52.7 | 249.4 | +359% |
| file_edit | 57.1 | 268.6 | +374% |
| json | 55.0 | 264.6 | +366% |
| math | 55.0 | 260.9 | +373% |
| prose | 35.2 | 172.5 | +386% |
| reasoning | 51.6 | 241.5 | +388% |
| summarization | 49.5 | 233.3 | +362% |

Prefill (cold, median tok/s) and concurrency (aggregate tok/s · TTFT p50):

| | gb10 | r9700 1.17 | r9700 2.0 |
|---|--:|--:|--:|
| prefill 2k | 1,704 | 4,728 | 5,256 |
| prefill 8k | 2,115 | 6,326 | 6,507 |
| prefill 32k | 2,220 | 6,429 | 6,462 |
| prefill 64k | 2,207 | 6,424 | 6,449 |
| 1 stream | 45 · 300 ms | 174 · 67 ms | 216 · 63 ms |
| 2 streams | 65 · 483 ms | 262 · 80 ms | 310 · 75 ms |
| 4 streams | 81 · 575 ms | 377 · 91 ms | 415 · 87 ms |
| 8 streams | 104 · 785 ms | 495 · 120 ms | 534 · 109 ms |

The R9700 pair prefills ~3× faster and decodes ~5× faster at every load. Batching helps both about alike: 1 → 8
streams gives the GB10 2.3×, radiance 2.0 2.5× and 1.17 2.8×.

## Results — time and energy at the wall (the whole box, 10 s plug samples)

QualityBench suite, concurrency 8:

| arm · mode | wall | mean W | Wh | **Wh per correct** | vs r9700 2.0 |
|---|---|---|---|---|---|
| gb10 · off | 296 min ¹ | 96 | 474.9 ¹ | **0.140** ¹ | 0.93× energy, 4.8× time |
| r9700 1.17 · off | 66 min | 499 | 546.3 | 0.162 | 1.08× |
| r9700 2.0 · off | 62 min | 492 | 508.8 | **0.150** | 1 |
| gb10 · on | 737 min | 95 | 1,172.0 | **0.329** | 0.97× energy, 5.1× time |
| r9700 1.17 · on | 160 min | 508 | 1,356.7 | 0.379 | 1.11× |
| r9700 2.0 · on | 145 min | 503 | 1,216.5 | **0.340** | 1 |

¹ gb10 thinking off is the clean run of card 0005. **Its repeat in this run took 424 min and 672.8 Wh (0.197 Wh per correct)** for the same items and the same scores (±1 pt). Night one was cut at its 300-minute limit inside mmlu_pro and resumed the next morning; for part of night one the box was also reachable by other clients (a configuration slip, since fixed), and the GB10's engine keeps no per-request count to tell their load apart. The repeat is shown, not used.

Tool steps (each run's whole window):

| step | gb10: min · Wh | r9700 1.17: min · Wh |
|---|---|---|
| tool-eval off | 8.7 · 13.4 | 2.6 · 21.6 |
| tool-eval on | 12.9 · 19.7 | 3.5 · 30.0 |
| with failures, off | 9.7 · 14.7 | 2.7 · 23.6 |
| with failures, on | 13.4 · 20.3 | 3.9 · 33.4 |

## Read in 20 seconds
- **The same answers on both boxes.** Paired item by item, thinking on, 7/7 tasks pass on both R9700 stacks against
  the GB10 (ifeval +2.8 on radiance 2.0 the only significant move, in its favour). Thinking off, the one real gap is
  **BFCL: GB10 +5 pts** (parallel calls; card 0001) — and it closes with thinking on.
- **The R9700 pair is 4.7–5× faster** at single-stream decode (239 vs 50 tok/s on radiance 2.0) and finishes the
  suite in a fifth of the time (145 vs 737 min thinking on).
- **Energy per correct answer is about the same.** GB10 draws a fifth of the power for five times as long:
  0.33 vs 0.34 Wh per correct thinking on, 0.14 vs 0.15 thinking off. Radiance 2.0 closed most of 1.17's gap.
- **Thinking on costs ~2.4× the energy on every stack** for lcb +17–19, mmlu_pro +9, bfcl +3–8, ifeval +3–4.
- **Tool use ties** (89 off, 92 on, on both boxes); the R9700 box solves each scenario 3–4× faster and recovers
  from injected failures as well or better. **Harmful actions are not zero on either** (2–6 of 104).

## Reproduce
```bash
# GB10: vLLM (eugr's b12x image), NVIDIA's FP4-Local checkpoint, the card's DGX Spark recipe
podman run --rm --device nvidia.com/gpu=all --ipc host --ulimit memlock=-1 --ulimit stack=67108864 \
  -v <weights dir>:/models:ro -p 8000:8000 --entrypoint vllm \
  docker.io/eugr/spark-vllm-b12x:nightly-20261015@sha256:03a956544e9f9b4b7ba91690469fbe7848bdbef29a7d2e0f5788368467bb3595 \
  serve /models --served-model-name flashnext --host 0.0.0.0 --port 8000 --tensor-parallel-size 1 \
  --mamba-cache-mode align --enable-prefix-caching --enable-chunked-prefill --dtype bfloat16 --kv-cache-dtype fp8 \
  --quantization modelopt_mixed --block-size 16 --load-format b12x --max-model-len 262144 --max-num-seqs 8 \
  --max-num-batched-tokens 4096 --speculative-config '{"method":"mtp","num_speculative_tokens":4}' \
  --gdn-decode-kernel b12x --linear-backend b12x --moe-backend b12x --no-enable-flashinfer-autotune \
  --mm-encoder-tp-mode data --reasoning-parser qwen3 --tool-call-parser qwen3_xml --enable-auto-tool-choice \
  --default-chat-template-kwargs '{"reasoning_effort":"medium"}' \
  --override-generation-config '{"temperature":1.0,"top_p":0.95,"top_k":20,"max_new_tokens":32768}' \
  --compilation-config '{"pass_config":{"fuse_act_quant":true}}' --gpu-memory-utilization 0.8
# 2x R9700, radiance 1.17: card 0001's command. radiance 2.0: card 0006's YAML entry.
# Then, against each server (harness pins: QualityBench 2f4f3379ed9a452d9cb313b85fa6700b010c3dc7, BetterBench f72bb9011a830fd98e82ab89d01e549efa76e9c5):
qualitybench run --endpoint http://<server>:8000/v1 --model flashnext --tokenizer Qwen/Qwen3.8-Flash-Next \
  --tasks gsm8k,ifeval,mmlu_pro,humaneval,mbpp,bfcl,lcb,needle --concurrency 8 --thinking off --out a.off.json
qualitybench run --endpoint http://<server>:8000/v1 --model flashnext --tokenizer Qwen/Qwen3.8-Flash-Next \
  --tasks gsm8k,ifeval,mmlu_pro,humaneval,mbpp,bfcl,lcb,needle --concurrency 8 \
  --thinking on --thinking-max-tokens 32768 --request-param reasoning_effort=medium --out a.on.json
qualitybench compare gb10.on.json r9700.on.json
uv tool install git+https://github.com/SeraphimSerapis/tool-eval-bench@ea2a77499f0fb371ac6653f5482d752ce4c37900
tool-eval-bench --base-url http://<server>:8000/v1 --model flashnext --hardmode --seed 1234 --parallel 4 --no-live \
  --no-think --json-file te.off.json            # thinking on: --backend-kwargs '{"reasoning_effort":"medium"}'
tool-eval-bench --base-url http://<server>:8000/v1 --model flashnext --hardmode --seed 1234 --parallel 4 --no-live \
  --no-think --error-rate 0.2 --json-file te-fail.off.json
tool-eval-bench compare <run_id gb10> <run_id r9700>
betterbench run --endpoint http://<server>:8000/v1 --model flashnext --runs 5 --seed 1234 --out a.speed.json
```
Energy: any metering plug on each box, sampled every 10 s, averaged over each run's window; Wh per correct = the
window's Wh / items answered correctly.

## What this card omits / caveats
- **One run per arm and mode.** Repeats seen: the R9700 box's thinking-off energy repeated to 0.3% (card 0005);
  GB10's BFCL thinking off read 89.2, 87.5, 87.8 over three runs (vLLM at concurrency 8 is not deterministic);
  the GB10's thinking-off energy did **not** repeat cleanly (footnote ¹).
- **Engine and quant differ on every pair**, and radiance 2.0 changed both at once (card 0007).
- **No tool-use run on radiance 2.0** yet; its tool numbers here are 1.17's.
- **No token counts for the GB10** (vLLM logs no per-request line), so no J per token for it; the R9700 box's are in
  cards 0004 and 0007 (0.84 J per generated token on 2.0).
- **8 concurrent sequences on both.** More batching would cut the GB10's time, and with it the baseline it pays
  (~50 W at idle, ~95 W under load).
- **Room cooling is not in the energy figures**; a hand-set aircon ran during some runs and is not compared.
- The R9700 box: 210 W cap per card, CPU governor `powersave` (not tuned). The GB10: not tuned.
