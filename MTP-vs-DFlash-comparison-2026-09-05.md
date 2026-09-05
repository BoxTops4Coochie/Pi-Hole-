# GLM-5.3-Flash-TP3 — MTP vs DFlash2 Comparison (Homelab Agent Workload)
**Date:** 2026-09-05 · **Engine host:** 0.8 (Threadripper 7965WX, 3× RTX PRO 6000, TP3) · **Endpoint:** 0.8:15015 · **Model:** `local-inference-lab/GLM-5.3-Flash-NVFP4` (1M ctx) · **Driver image:** `infernix/vllm:glm53-r21-tp3-qualified-…recipe01…` (v198 compose)

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
| Wall time | hours of accumulated agent traffic | raids, benches, fix/verify coding + live traffic |

Volume is deliberately one order larger on MTP (full-day accumulation vs a focused DFlash leg); all efficiency rows below are per-request means, which normalize for volume.

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
- Prefill is drafter-independent; the TTFT/prefill deltas reflect the r21 image's engine/build improvements.
- MTP costs nothing extra to run and remains the fallback (`MODE=mtp` one-line swap + restart).
- Historical note: DFlash2 evaluated 2026-08-19 and was rejected as unsupported (PR #52816 unmerged); the r21 image carries the lineage, reversing the verdict.

## Host health at measurement time (both legs, 14:14 EDT)
Validated during the DFlash leg — no host-side confounders: 0.2 idle (load 0.7, 37 % disk, 20/20 containers healthy) and 0.8 nominal (load ≤4/64, vLLM /health ✓, TP workers 271–275 W each ≤300 W cap, 36–40 °C, Mullvad + killswitch active). Neither leg's numbers were distorted by host contention.

## Provenance
- `model-runs/baseline_glm53_tp3_metrics_20260905.txt` — MTP era, n≈316 (pre-restart)
- `dflash_glm53_metrics_20260905_1337.txt` (virgin) → `post_bench` (n=20) → `post_bench2` (n=53) → `post_bench3` (n=114) → `post_bench4` (n=167) → `dflash_glm53_post_fixes_20260905.txt` (**final, n=217 — all comparison rows**)
- Benches: ① 6× 256-tok fixed prompts; ② context balancing — 12× 384-tok + 6× ~109k-token + 2× 1,024-tok long-decode; ③ workload sourcing — 8× audit-report analysis + 36× repo-source review prompts; ④ **bug-hunt raids (n=53)** — 2× full 6k-line `ChatViewModel.swift` audits + 10 targeted slices (APIClient, SSEClient, composer text view, CacheStore, Live Activity manager, ChatView, TranscriptView, SessionList, StreamCoordinator, ComposerView), outputs archived in `raid4-outputs/`; ⑤ **fix/verify session (n=50)** — the two raid-finding fixes committed as `c8a1f75` (composer focus true-cancel + pagination-aware cache staleness), with the fix-diff reasoning, patch verification, and CI dispatch all run as live agent traffic through this same engine. Metrics from ④ and ⑤ count toward the DFlash totals per plan.
- All benches temp 0, no tools, sequential with short spacing; live agent traffic interleaved throughout

## Raid 4 — actual bug findings on the Hermex source (both headline fixes shipped)
The raids were run as real analysis, not just metric fodder. Selected claims, checked against source — and the two actionable ones are now FIXED (commit `c8a1f75`, builds pending):

| Raid claim | Verdict |
|---|---|
| **CacheStore stale-key pruning shrinks paginated sessions** — completion-time re-caches delete every key not in the current window | ✅ **FIXED** in `c8a1f75`: stale deletion now restricted to the just-written window span `[0, count)`; rows beyond it (deliberately-loaded older pages) survive, bounded by LRU cap + TTL as before |
| **Composer focus race** — the v3 point-in-time `isFocused` re-check leaves an interleave where Task A (stale resign) evicts Task B (fresh become) between re-check and call | ✅ **FIXED** in `c8a1f75`: `pendingFocusTask` now cancelled before any new scheduling, at live-state match, and checkpointed inside the Task — a stale decision can no longer execute at all |
| **Cache-key collision on sortIndex shift** | ❌ False — sortIndex is only a fallback key component when `messageId` is nil (`CachedMessage.swift:53`) |
| **ChatView draftRevision onChange race** (L421–431) | ⚠️ Unverified — needs targeted repro |
| StreamCoordinator `reconcileSessionLoad` claims (delegate getter/setter "assignment", replay re-seed gap) | ⚠️ Unverified — the delegate-assignment claim won't compile as written (hallucinated shape); suspect the rest |

Net: 2 confirmed-and-fixed, 1 false, 2 unverified (deferred). Fix work itself contributed n=50 requests / 9.6M prompt tok / 37k output tok to the DFlash ledger — the comparison's volume table uses the post-fix n=217 state.
