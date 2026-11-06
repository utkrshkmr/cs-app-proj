# P1 · `qnum` — a low-precision numerics library

> **Start after:** CS:APP Ch. 1–2 (and Data Lab) · **Read while building:** §3.1–3.6, §3.11 before A1.6 · **Language:** C · **Budget:** ≈54 h (months 1–2) · **Spec:** `spec/02-p1-qnum.tex`

## What you build

A C library for every number format that matters in LLM inference, with conversions written **only with integer bit manipulation**:

- **Floats:** fp16, bf16, fp8 **E4M3** and **E5M2** (OCP), to and from fp32.
- **Rounding:** round-to-nearest-even (RNE), round-toward-zero, and stochastic rounding (PCG32-driven).
- **Edge cases done right:** subnormals, ±0, infinities, NaN canonicalization, saturating vs non-saturating overflow (E4M3 has no infinity).
- **Integer quantization:** int8 / int4, symmetric and asymmetric, per-group scales stored as bf16, packed int4 with correct sign extension.
- **Block floating point** `BFP(m, n)`: a block of `n` mantissas sharing one exponent (the Tenstorrent-style format).
- **Measurement:** a safetensors reader and a suite that reports SQNR / ULP error of every format on real model weights.

## Why it matters

KV-cache compression, weight quantization, and the accelerator formats you'll meet on Tenstorrent (BFP) or Hopper/Blackwell (fp8, micro-scaled fp4) all reduce to these bit manipulations. It's also the first place compiler and ISA differences show up in numerical output — the same question as a compiler-divergence study, at the scale of one instruction.

## The core ideas (derived in the spec)

- **IEEE encoding as integers.** A float is `(-1)^s · 1.f · 2^(e−bias)`; the bit patterns of non-negative floats are *monotone as integers*, which is what makes ULP distance and exhaustive testing easy.
- **RNE with integer arithmetic.** Dropping `d` bits from `X`: add `2^(d−1) − 1 + (lsb of the kept part)` then shift. The spec proves the three cases (below half, above half, exact tie). Carry into the exponent is a *feature*: it rounds up to the next binade correctly.
- **Narrowing the exponent.** Values below the target's normal range shift into subnormals before rounding; overflow either saturates or becomes Inf/NaN depending on the format.
- **Integer quantization error.** Uniform step `Δ` gives error ≈ `Δ²/12`; storage is `B + 16/g` bits per weight for group size `g` — the trade-off you'll plot.
- **BFP is integer quantization with power-of-two scales.** An element `k` octaves below the block max loses `k` bits; MLP matrices with wide dynamic range suffer more.

## Repository layout

```
qnum/
  include/qnum/  fp.h intq.h bfp.h stats.h rng.h safetensors.h ulp.h
  src/           fp.c intq.c bfp.c stats.c rng.c safetensors.c json.c
  tests/         test_fp16.c test_bf16.c test_fp8.c test_intq.c test_bfp.c test_stats.c test_safetensors.c
  bench/         bench_convert.c  (scalar vs intrinsics throughput)
  scripts/       gen_golden.py plot_sqnr.py export_weights.py
  docs/          machine.md design.md blog.md
```

## Assignments

| Id | Title | Budget | Read first | Done when |
|----|-------|--------|------------|-----------|
| A1.1 | Repository, harness (`ck.h`), Makefile | 6 h | spec Ch. 1 | builds with gcc+clang in all 4 `BUILD` variants; `v0.0.1` pushed |
| A1.2 | fp16 & bf16, both directions, 3 rounding modes | 12 h | §2.4 | all 2¹⁶ round trips; goldens; ties-to-even; stochastic mean within 4σ |
| A1.3 | fp8 E4M3 & E5M2 | 8 h | §2.4 | goldens vs `torch.float8_*`; saturating and NaN overflow both correct |
| A1.4 | int8/int4 group quantization + BFP | 10 h | §2.2–2.3 | error ≤ Δ/2; int4 pack identity; BFP idempotent |
| A1.5 | Measurement suite on `stories110M` | 10 h | §2.1.3 | CSV per (tensor, format, group) + 3 figures regenerate in one command |
| A1.6 | Assembly study + hardware oracles | 8 h | §3.1–3.6, §3.11 | zero mismatches over all 2³² fp32 inputs vs `vcvtps2ph` |

## How it's tested

Exhaustive round trips (2¹⁶ / 2⁸ patterns), an exhaustive 2³² sweep against the F16C instruction, golden tables from NumPy/PyTorch, monotonicity over 10⁶ pairs, constructed ties at binade boundaries, statistical tests for stochastic rounding, and `ASan` + `-Wconversion` clean builds.

## Blog posts

- **Post 1 — "Every inference number format, from the bits up."** Central figure: SQNR vs bits-per-weight for every format and group size.
- **Post 1b — "Verifying a float converter against every one of the four billion inputs."** Exhaustive testing as proof, with planted bugs the sweep catches.

## Git lesson

Object model and the three trees, `.gitignore`, SSH keys, first remote, annotated tags, commit messages.

**Feeds:** P2 (quantized weights), P5 (JIT quantizer kernels must be bit-identical to `qnum`), P6 (submodule).
