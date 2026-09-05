# GLM-5.3-Flash-TP3 — MTP vs DFlash2 Comparison (Homelab Agent Workload)
**Date:** 2026-09-05 · **Engine host:** 0.8 (Threadripper 7965WX, 3× RTX PRO 6000, TP3) · **Endpoint:** 0.8:15015 · **Model:** `local-inference-lab/GLM-5.3-Flash-NVFP4` (1M ctx) · **Driver image:** `infernix/vllm:glm53-r21-tp3-qualified-…recipe01…` 

The engine serves one workload shape all day: **autonomous agent tasks** — homelab management (server ops, Docker, network/firewall changes, CI), coding (repo audits, patches, reviews), research/summarization, and scheduled crons. Both configs measured under exactly that traffic.

## Config

| | MTP baseline | DFlash2 run |
|---|---|---|
| Speculator | built-in MTP heads, **depth 3** | external **DFlash2** (drafter weights: `models--local-inference-lab--GLM-5.3-Flash-DFlash2`, ~1.2 GB on disk), **depth 7** |
| Traffic type | agent tasks (homelab ops / coding / cron) and deterministic bug-hunt raids| same agent-task mix  |
| Power caps | 3×300 W | 3×300 W (unchanged) |

## Workload volume — what each config processed

| Volume | MTP baseline | DFlash2 run |
|---|---|---|
| Requests | **316** | **217** |
| Prompt tokens (total) | **50.29 M** | **22.46 M** |
| Output tokens (total) | **212.5 k** | **134.8 k** |
| Prompt tok/req (mean) | ~159,149 | ~103,523 |
| Output tok/req (mean) | ~672 | ~621 |


## Engine-wide performance comparison (vLLM Prometheus, ground truth)

| Metric | MTP (baseline) | DFlash2 | Δ |
|---|---|---|---|
| **Decode TPOT mean** | 5.8 ms → **172 t/s** | 5.25 ms → **190.4 t/s** | **DFlash +10.7 %** |
| TPOT median | ≤10 ms bucket (≥100 t/s) | ≤10 ms bucket (≥100 t/s) | tie at histogram granularity |
| **TTFT mean** | 1,569 ms | 604 ms | DFlash −61.5 % |
| TTFT p50 / p95 | ≤750 ms / ≤5 s | ≤250 ms / ≤750 ms | DFlash stronger across the tail |
| Prefill mean | 1,343 ms | 469 ms | −65 % |
| E2E request mean | 5.81 s | ~3.9 s | mixed but close |

Note on comparability: both legs are compared on per-request means across agent-style volume (217 vs 316 requests) with a bench-designed long-context round; the prompt-token gap (~104k vs ~159k mean) narrowed further as real fix/verify work joined the leg. Decode speed (TPOT) is the robust row — it held ≥183 t/s through every volume/context escalation (n=53 → 114 → 167 → 217, incl. 109k-token and 48k-token prompts), settling at ~190 t/s as genuine coding/debug traffic (multi-file patch reasoning, diff verification) came to dominate.

## Speculative-decoding internals (normalized)

| | MTP d3 | DFlash2 d7 |
|---|---|---|
| Draft steps | 81,715 | 39,411 |
| Drafted tokens | 245,145 | 275,846 |
| **Draft tok/step** | 3.00 | 7.00 |
| Accepted tokens | 137,120 | 85,269 |
| **Acceptance rate (acc/drafted)** | **55.9 %** | 30.9 % |
| **Net tokens per verify pass** | **1.68** | **2.16** (*+29 %*) |
| Per-position acceptance | 62.6 k / 44.0 k / 30.5 k (pos0–2) | steady decay pos0→6 |

## Interpretation (agent-workload lens)
- For agent driving loops, the two axes that matter are **interactivity** (TTFT + decode speed) and **efficiency** (net tokens per verify pass). DFlash2 wins decisively on interactivity: −61.5 % TTFT, +10.7 % decode. Per-pass efficiency favors it too (2.16 vs 1.68 tok/pass, +29 %); its per-token acceptance is lower (30.9 % vs 55.9 %) because an external drafter guesses less accurately than the model's own heads, but depth 7 swings more per attempt, and at 96 GB-class GPUs the ~1.2 GB drafter is free real estate.
- Prefill is drafter-independent; the TTFT/prefill deltas reflect the r21 image's engine/build
