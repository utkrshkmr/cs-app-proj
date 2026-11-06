# P5 · `tinyjit` — an AVX2 JIT compiler for fused tensor operations

> **Start after:** P4 done, plus a second pass through CS:APP §3.2–3.7 and §3.11 · **Read while building:** §7.4–7.7 again before A5.6 · **Language:** C · **Budget:** ≈58 h (months 9–10) · **Spec:** `spec/06-p5-tinyjit.tex`

## What you build

A small compiler from a **tensor-expression IR** (elementwise arithmetic, transcendental approximations, reductions over fp32 arrays, and `qnum`'s quantize/dequantize ops) to **x86-64 AVX2 machine code generated at run time** into executable memory and called through a function pointer. It does:

- **operator fusion** — one loop, no intermediate arrays,
- **linear-scan register allocation** over the 16 `ymm` registers, with spilling,
- a **vectorized main loop with a masked tail**,
- a **constant pool addressed RIP-relatively** — so you compute a relocation by hand,
- (stretch) the same code generator writes a **relocatable ELF `.o`** that `gcc` links, closing the loop with CS:APP Ch. 7.

## Why it matters

This is the last mile of every DL compiler (XLA, TVM, Triton's CPU backend, TT-MLIR lowering to kernels): an expression DAG becomes a loop nest becomes instruction bytes, and register pressure and memory traffic decide performance. Doing it for AVX2 removes every abstraction between you and the numbers; the fused-vs-unfused measurement is the elementary case of the fusion decisions those compilers make.

## The core ideas (explained in the spec)

- **Instruction encoding.** VEX prefixes, ModR/M, SIB, displacements — every encoder is golden-tested byte-for-byte against GNU `as`.
- **System V calling convention (§3.7).** Arguments in `rdi`, `rsi`, …; callee-saved registers; `vzeroupper` before returning.
- **Executable memory.** W⊕X: write into an `mmap`'d buffer, then `mprotect` to `PROT_READ|PROT_EXEC`.
- **Transcendentals in SIMD.** `exp` by range reduction (`x = k·ln2 + r`) plus a fitted polynomial and exponent-field construction; `rsqrt` with a Newton step.
- **Why fusion pays.** An unfused chain of `k` elementwise ops moves ≈ `k×` the bytes of the fused kernel — on a bandwidth-bound op, that's the speedup.

## Repository layout

```
tinyjit/
  include/tinyjit/  ir.h asm.h regalloc.h codegen.h jit.h elfout.h interp.h
  src/              ir.c asm.c regalloc.c codegen.c jit.c elfout.c interp.c mathlib.c
  tests/            test_asm.c test_regalloc.c test_codegen.c test_math.c test_fuzz.c test_elf.c
  bench/            fused_vs_unfused.c  (rmsnorm, silu-gate, quantize vs -O3 C vs numpy)
  scripts/          asm_golden.py fit_exp_poly.py
```

## Assignments

| Id | Title | Budget | Read first | Done when |
|----|-------|--------|------------|-----------|
| A5.1 | Encoder, golden-tested against the assembler | 10 h | §3.2–3.6 | byte-exact for every form and register combination |
| A5.2 | IR, interpreter, first kernel | 8 h | §3.7, §9.8 | `y = a*b + c` runs from JIT memory |
| A5.3 | Register allocation, masked tail, unrolling, fuzzing | 12 h | §3.11 | arbitrary DAGs compile correctly under register pressure |
| A5.4 | Transcendentals + fusion benchmark | 12 h | Ch. 5 (re-skim) | JIT RMSNorm/SiLU-gate replace `cforward`'s; fused-vs-unfused figure |
| A5.5 | Integer ops + quantizer kernels | 8 h | P1 | JIT conversions bit-identical to `qnum` |
| A5.6 | Stretch: relocatable ELF object | 8 h | §7.4–7.7 | `gcc` links your `.o`; `readelf -r` shows your relocations |

## How it's tested

Encoder goldens over every register triple and addressing form, label/patch displacement tests, codegen vs a double-precision interpreter across awkward lengths (0, 1, 7, 9, 1025 …) within 1–2 ULP, 10⁴ fuzzed random DAGs (including forced spills) under ASan, crash isolation in forked children, masked-tail checks against a `PROT_NONE` guard page, deterministic output bytes, and bit-identity with `qnum`.

## Blog posts

- **Post 5 — "An AVX2 JIT for fused tensor ops in 2000 lines of C."**
- **Post 5b — "How I test a code generator."**

## Git lesson

Automated bisection (`git bisect run`), `range-diff`, `rerere`, `describe`, signed commits.

**Uses:** P1, P2. **Feeds:** P2/P6 (faster elementwise kernels).
