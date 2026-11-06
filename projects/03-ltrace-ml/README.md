# P3 · `ltrace-ml` — an interposition tracer and sampling profiler

> **Start after:** CS:APP Ch. 7 and §8.1–8.4 (and Shell Lab) · **Read while building:** §8.5 before A3.3, skim §12.3–12.4 before A3.4 · **Language:** C (+ a Python symbolizer) · **Budget:** ≈55 h (months 5–6) · **Spec:** `spec/04-p3-ltrace.tex`

## What you build

A shared library loaded with `LD_PRELOAD` that **intercepts calls into other libraries** — first `malloc`/`free`/`realloc`, then the CUDA runtime (`cudaMalloc`, `cudaMemcpyAsync`, `cudaLaunchKernel`, …) — logging sizes, timings and call stacks with overhead low enough to run under a real PyTorch or vLLM workload. Plus:

- a **`SIGPROF` sampling profiler** that emits flame-graph input,
- a **launcher** (`ltrace-ml run -- prog`) built on `fork`/`execve`/`waitpid`,
- an **offline symbolizer** (Python) that turns raw addresses + `/proc/self/maps` into names, folded stacks and summary tables.

## Why it matters

It becomes the measurement tool for serving and compiler work: what does PyTorch actually ask the CUDA runtime for, how often, and from where? Which host-side path burns time between kernel launches? CUPTI and Nsight answer parts of this; building the poor man's version teaches the linker and signal machinery they stand on — and you can point it at a WRF binary just as easily.

## The core ideas (explained in the spec)

- **Why preloading works.** The dynamic linker searches preloaded objects first, so your `malloc` wins symbol resolution; `dlsym(RTLD_NEXT, "malloc")` finds the real one. Calls bound at static link time are invisible — hence the `LD_PRELOAD` vs `--wrap` interposition matrix.
- **Reentrancy.** `dlsym` itself may call `malloc`; a bump arena and a per-thread recursion guard prevent infinite recursion and deadlock.
- **Capturing the caller.** A frame-pointer walk (guarded against bad frames), optionally libunwind.
- **Sampling is statistics.** With `n` samples, a function's share `p̂` has a confidence interval ≈ `p̂ ± 1.96·√(p̂(1−p̂)/n)`; choose the rate (e.g. 997 Hz, prime to avoid aliasing) from the accuracy you need.
- **Async-signal safety.** The handler only writes to a lock-free per-thread ring buffer; overflow is counted, never silently dropped.

## Repository layout

```
ltrace-ml/
  include/ltrace/  hooks.h log.h stack.h prof.h maps.h
  src/             hooks_malloc.c hooks_pthread.c hooks_cuda.c log.c stack.c prof.c maps.c init.c launcher/main.c
  tools/           symbolize.py
  tests/           targets/ (alloc_counter.c cpu_shares.c threads.c) test_hooks.c test_stack.c test_prof.c test_launcher.c
  bench/           overhead.c
```

## Assignments

| Id | Title | Budget | Read first | Done when |
|----|-------|--------|------------|-----------|
| A3.1 | Allocation hooks + launcher | 10 h | §7.10–7.13, §8.4 | by-call-site allocation table for any dynamic binary |
| A3.2 | Stack walking; `LD_PRELOAD` vs `--wrap` | 8 h | §3.7, §7.13 | stacks on every event; interposition matrix measured |
| A3.3 | Sampling profiler (single thread) | 10 h | §8.5 | flame graph + confidence intervals that hold 19/20 runs |
| A3.4 | Threads | 8 h | §12.3–12.4 (skim) | per-thread attribution; TSan clean |
| A3.5 | CUDA runtime interposition | 10 h | — | a trace of what one PyTorch step asks of `libcudart` |
| A3.6 | Overhead study | 5 h | — | cost of the tool, in numbers |
| A3.7 | Stretch: a WRF binary | 4 h | — | host-side profiles compared across two compiler builds |

## How it's tested

Exact event counts on a known target, reentrancy (a hook that calls `printf` must not recurse or deadlock), stack-walker correctness and safe truncation, profiler accuracy against a 70/20/10 % CPU split, a 10 kHz / 8-thread signal-safety soak under TSan, launcher exit-status and `SIGINT` propagation, symbolizer stability across ASLR layouts, and the 2×2 interposition matrix as tests.

## Blog posts

- **Post 3 — "What does PyTorch actually ask the CUDA runtime for? 400 lines of `LD_PRELOAD` later."**
- **Post 3b — "A sampling profiler is a statistics problem."**

## Git lesson

History archaeology (`log -S`, `log -G`, `blame`), the reflog as a safety net, worktrees, versioned hooks, `.gitattributes`.

**Uses:** P2 as its first target. **Feeds:** P6 (profiling the server).
