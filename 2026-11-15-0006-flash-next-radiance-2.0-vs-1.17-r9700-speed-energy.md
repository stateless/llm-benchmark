# Report card — BetterBench: radiance 2.0.0 + IQR-V4 vs radiance 1.17.0 + iq4r, Qwen3.8-Flash-Next on 2× R9700 (speed + wall energy)

> **BetterBench** — per-category decode speed, a prefill sweep and a concurrency sweep. — [BetterBench](https://github.com/GGZ14/BetterBench)

**Why this card:** radiance 2.0.0 (released 2026-11-14) adds chained kernel launches ("up to 40%"), replaces
the FP8 KV cache with i8, and needs a new Flash-Next container (IQR-V4). This is the first speed and energy
check of the upgrade on the same box that produced [card 0004](2026-11-15-0004-flash-next-r9700-think-vs-nothink-energy-tokens.md).
**Speed and energy only — quality on the new quant is not measured here** (a QualityBench A/B follows).

**What we ran on:** 2× AMD Radeon AI PRO R9700 (gfx1201, 32 GB each, **210 W cap** per card, PCIe 5.0 x8 each,
peer-to-peer on), Ryzen 9 9950X, 64 GB DDR5-6000, Linux 7.2.6. One arm at a time, alone on the cards; the
shared gateway's route to the host withdrawn during each run.

| arm | engine | container | KV | config |
|---|---|---|---|---|
| **qfn-r9700 (1.17)** | radiance **1.17.0** | `qwen3.8-next-flash-fp8-iq4r-moe` | fp8 | CLI flags (card 0001) |
| **qfn-r9700 (2.0)** | radiance **2.0.0** | `qwen3.8-next-flash-iqr-v4` | **i8** | YAML, the same settings |

**Provenance (locked):**
- 1.17: `docker.io/stilldeadcode/radiance:1.17.0@sha256:c2acbaa373c8…` · `StillDeadcode/qwen3.8-next-flash-fp8-iq4r-moe@8267a673`.
- 2.0: `docker.io/stilldeadcode/radiance:2.0.0@sha256:5ee52dc14c77…` (tag v2.0.0 = `ddcb9c6`) ·
  `StillDeadcode/qwen3.8-next-flash-iqr-v4@679c9432` (113.3 GiB).
- Both: TP2, `tp-wire wht6`, 8 sequences, 200000 context, `placement expert_tiered`, **24 GiB pinned host pool in
  all**, 4 GiB host prefix tier, MTP 3, 2048-token prefill chunk, `reasoning-effort medium`, `default-max-tokens 32768`.
  Headroom 128 MiB (1.17) / 160 MiB (2.0, upstream's).
- BetterBench **v0.6.1** (`f72bb90`), `--runs 5 --seed 1234`, corpus v1.0, temperature 0.7, cold prefix cache.

> ⚠ **Two runs, a day apart, not interleaved** (BetterBench: cross-file compares are unpaired in time).
> **Engine AND quant changed together** — this measures the upgrade as shipped, not either part alone.

## Results — decode, single stream (per-category median tok/s)

| category | 1.17 | 2.0 | Δ [95% CI] |
|---|---|---|---|
| reasoning | 179.3 | 241.5 | **+31.1%** [+19.4, +42.8] |
| math | 209.2 | 260.9 | **+25.4%** [+20.3, +30.5] |
| code | 194.5 | 249.4 | **+25.1%** [+21.1, +29.2] |
| chat | 156.6 | 204.1 | **+24.9%** [+14.1, +35.6] |
| file_edit | 215.7 | 268.6 | **+24.4%** [+17.6, +31.3] |
| json | 219.5 | 264.6 | **+20.5%** [+12.9, +28.0] |
| summarization | 190.8 | 233.3 | **+19.7%** [+7.0, +32.4] |
| prose | 145.1 | 172.5 | **+19.4%** [+12.8, +26.0] |
| **weighted** | **189.5** | **238.9** | **+26%** |

All eight significant.

## Results — concurrency and prefill

| streams | aggregate tok/s 1.17 → 2.0 | Δ | TTFT p50 (ms) | per-request decode |
|---|---|---|---|---|
| 1 | 174.0 → 216.0 | **+24%** | 66.7 → 63.1 | 194.9 → 239.0 |
| 2 | 261.8 → 310.4 | +19% | 79.6 → 74.9 | 146.8 → 174.8 |
| 4 | 377.3 → 414.6 | +10% | 90.6 → 87.5 | 111.2 → 120.9 |
| 8 | 495.1 → 533.9 | **+8%** | 119.5 → 109.1 | 77.0 → 81.7 |

| prefill depth (cold) | 1.17 | 2.0 | Δ |
|---|---|---|---|
| 2k | 4,728 | 5,256 | +11% |
| 8k | 6,326 | 6,507 | +3% |
| 16k | 6,375 | 6,517 | +2% |
| 32k | 6,429 | 6,463 | +1% |
| 64k | 6,424 | 6,449 | 0% |

## Results — energy at the wall (the whole box, 10 s plug samples)

| | 1.17 | 2.0 | Δ |
|---|---|---|---|
| BetterBench run, wall | 9.7 min | 8 min | −18% |
| mean power | 508 W | 507 W | — |
| energy | 82.0 Wh | **71.3 Wh** | **−13%** |
| generated tokens (engine's count) | 107,360 | 105,820 | same work |
| **J per generated token** | 2.75 | **2.43** | **−12%** |

## Read in 20 seconds
- **+26% single-stream decode, every category faster, all significant** — the gain an interactive agent feels.
- **The gain shrinks with batching:** +24% at 1 stream, +8% at 8 — what removing per-step launch overhead
  predicts. Prefill at depth is unchanged (~6,450 tok/s); short prompts +11%.
- **Same power, less time: −13% energy for the same benchmark work**, 2.43 J per generated token.
- **Loads in 12 s** (1.17: 49 s): 65.6 GiB read at 7.3 GB/s; the 47.7 GiB n-gram table read in place from disk.
- The KV cache holds more: 912k tokens (4.6 sessions at 200k) vs 813k (4.1).
- Quality of the IQR-V4 container: not measured here. The author's card reports KL vs bf16 on model-written text
  0.0242 (iq4r 0.0279) with this wire.

## Reproduce
2.0 takes only a YAML config (`radiance --config FILE`). The model entry used here, as one line of YAML flow:
```
{"server":{"host":"0.0.0.0","port":8000},"memory":{"gpu-headroom-mib":160},
 "models":{"flashnext":{"model":"/models/qwen3.8-next-flash-iqr-v4.rad","tp":2,"tp-wire":"wht6","max-num-seqs":8,
 "max-model-len":200000,"placement":"expert_tiered","host-pool-mib":24576,"kv-cache-dtype":"i8","prefix-cache-mib":4096,
 "num-speculative-tokens":3,"max-num-batched-tokens":2048,"reasoning-effort":"medium","default-max-tokens":32768}}}
```
```bash
uv tool install git+https://github.com/GGZ14/BetterBench@f72bb9011a830fd98e82ab89d01e549efa76e9c5
betterbench run --endpoint http://<server>:8000/v1 --model flashnext --runs 5 --seed 1234 --out v200.json
betterbench compare v117.json v200.json
```
Energy: any metering plug on the host, averaged over the run's window.

## What this card omits / caveats
- **Not interleaved:** `betterbench ab` alternating the two would cancel drift between days.
- **Engine and quant moved together.** The IQR-V4 container needs a build newer than 1.17.0; splitting the gain
  would need 2.0 with the old container, which was not tried.
- **Quality unmeasured** — a QualityBench A/B (thinking off and on) is the next card.
- Another 2× R9700 rig reported 213.5 tok/s weighted, 446 at 8 streams and 5,641 prefill at 64k on 2.0 with
  peer-to-peer **off** and 20 passes; this rig, peer-to-peer on, saw no prefill or batch regression.
- 210 W cap per card; the CPU governor was `powersave` (radiance flags it as a host-gap risk; not tuned here).
