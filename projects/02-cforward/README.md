# P2 · `cforward` — an optimized single-core transformer forward pass

> **Start after:** CS:APP Ch. 5 and §6.1–6.7 (and Cache Lab) · **Early start allowed:** A2.1 right after P1 · **Language:** C · **Budget:** ≈60 h (months 3–4) · **Spec:** `spec/03-p2-cforward.tex`

## What you build

A plain-C, llama2.c-style decoder-only transformer (Llama architecture: RMSNorm, RoPE, grouped-query attention, SwiGLU, sampling) that loads real weights, **matches the reference logits**, and is then optimized one technique at a time — loop order, cache blocking, AVX2 intrinsics, multiple accumulators, quantized weights from `qnum` — with every step placed on a **roofline you measured yourself**.

It starts with CS:APP's *memory mountain*, so the roofline's two ceilings (peak FLOP/s and peak bandwidth) are numbers from your machine, not a datasheet.

## Why it matters

Prefill is compute-bound and decode is bandwidth-bound; every serving paper assumes this. Here you derive it, measure it on a CPU, and see what 8-bit and 4-bit weights do to the decode ceiling — the same analysis you'll later do for GPUs, Tenstorrent cores, and bandwidth-rich but FLOPs-poor accelerators.

## The core ideas (derived in the spec)

- **Cost model.** Decode does ~2 FLOPs per weight byte-read at fp32 → arithmetic intensity far below the ridge point → bandwidth-bound. Prefill batches `T` tokens per weight read → intensity grows with `T` → compute-bound.
- **Why one accumulator is slow (§5.7).** A single FMA chain is bounded by FMA *latency*; `k` independent accumulators approach FMA *throughput*. The CPE table makes this visible.
- **Cache blocking (§6.6).** Choose tile sizes so the working set fits L1/L2; derive miss counts.
- **Online softmax.** Attention computed in one pass with a running max and rescaled sum — one invariant, proved in the spec (and the subject of post 2b).

## Repository layout

```
cforward/
  include/cforward/  model.h kernels.h forward.h tokenizer.h roofline.h
  src/               one .c per header; kernels/ matvec_naive.c matvec_blocked.c matvec_avx2.c ...
  tests/             test_rmsnorm.c test_rope.c test_softmax.c test_matvec.c test_attention.c test_forward.c
  bench/             mountain.c peak_flops.c peak_bw.c bench_matvec.c bench_decode.c
  scripts/           export_safetensors.py golden_logits.py plot_roofline.py plot_mountain.py
  data/              (ignored) stories15M.safetensors tokenizer.bin golden/*.bin
```

## Assignments

| Id | Title | Budget | Read first | Done when |
|----|-------|--------|------------|-----------|
| A2.0 | Memory mountain, peak FLOP/s, peak BW, roofline | 8 h | §6.6, §5.7 | ceilings and cache plateaus in `docs/machine.md` |
| A2.1 | Reference forward pass | 14 h | — | 200 greedy tokens equal the Python reference |
| A2.2 | The matvec ladder R1–R4 | 10 h | §5.7–5.11 | CPE table + roofline points for four variants |
| A2.3 | Quantized weights via `qnum` | 10 h | P1 | bf16/int8/int4 paths; tokens/s vs bytes figure |
| A2.4 | Online-softmax attention + blocked GEMM prefill | 12 h | §6.5 | prefill-vs-decode curves |
| A2.5 | Hardware counters | 6 h | §5.12–5.14 | bandwidth-bound diagnosis confirmed by `perf` counters |

## How it's tested

Kernel unit tests vs NumPy goldens, every matvec/GEMM variant vs `naive` across awkward shapes (1, 7, 9, 65, …), online = offline softmax within 8 ULP, causality (prefill logits = token-by-token decode), end-to-end golden tokens, GQA and odd head-dim edge cases, CPE regression thresholds, ASan/UBSan, and one cachegrind comparison.

## Blog posts

- **Post 2 — "A roofline for LLM decode, measured from scratch on one CPU core."**
- **Post 2b — "Online softmax, explained with one invariant."**

## Git lesson

Branches as pointers, fast-forward vs three-way merge, rebase / interactive rebase, `--force-with-lease`, stash, conflict resolution, manual `git bisect`.

**Uses:** P1. **Feeds:** P3 (first tracing target), P4 (paged attention), P5 (JIT replaces elementwise kernels), P6 (the engine).
