# Roadmap — when to attack each project

The projects follow CS:APP's chapter order, but you should not wait until you've finished the whole book, nor start a project before the reading its first assignment needs. This file gives three levels of guidance:

1. **Project gates** — the chapters to finish before starting a project.
2. **Assignment gates** — the extra reading each later assignment needs, so you can start building early and read the rest in parallel.
3. **A month-by-month schedule** that interleaves reading, labs and building.

Section numbers refer to CS:APP 3rd edition. "Pn" = project n, "An.m" = assignment m of project n (as numbered in the spec).

---

## 1. Project gates at a glance

| # | Project | Finish before starting | Read while building | Do this lab first |
|---|---------|-----------------------|---------------------|-------------------|
| 0 | *(before anything)* | Ch. 1 (tour); set up an x86-64 Linux environment | — | — |
| 1 | `qnum` | **Ch. 2 complete** (2.1 bits/bytes, 2.2–2.3 integers, 2.4 floating point) | Ch. 3.1–3.6 and 3.11 before **A1.6** | **Data Lab** |
| 2 | `cforward` | **Ch. 5 complete**, **Ch. 6.1–6.7** (memory mountain) | rest of Ch. 6; skim Ch. 4.1–4.5 before Ch. 5 | **Cache Lab** |
| 3 | `ltrace-ml` | **Ch. 7 complete**, **Ch. 8.1–8.4** | 8.5 (signals) before **A3.3**; skim 12.3 before **A3.4** | **Shell Lab** |
| 4 | `pagedkv` | **Ch. 9 complete** | — | none — A4.1 *is* Malloc Lab, done properly |
| 5 | `tinyjit` | **P4 done**; re-read **3.2–3.7, 3.11** | 7.4–7.7 again before **A5.6** | Bomb / Attack Lab if you skipped them |
| 6 | `servlet` | **Ch. 10 and Ch. 11** | Ch. 12 before **A6.3** | Proxy Lab (optional warm-up) |

Chapters 1 and 4 have no project. Read Ch. 1 first; skim Ch. 4.1–4.5 (Y86 pipelining) before Ch. 5 because §5.7's latency/throughput argument assumes it.

### Why these gates

- **P1 after Ch. 2** — every converter is integer bit manipulation plus IEEE 754 rounding (§2.4). Ch. 3 is only needed for the assembly study in A1.6, so read it while you implement A1.2–A1.5.
- **P2 after Ch. 5 and 6.1–6.7** — A2.0, the very first assignment, is CS:APP's memory mountain; the roofline's two ceilings come from §6.6 and §5.7. The reference forward pass (A2.1) needs only C, so you *may* start it right after P1 while you finish Ch. 5–6.
- **P3 after Ch. 7 and 8.1–8.4** — `LD_PRELOAD`, PLT/GOT and `dlsym(RTLD_NEXT)` are §7.10–7.13; the launcher is `fork/execve/waitpid` (§8.4). The sampling profiler (A3.3) needs signals and async-signal safety (§8.5).
- **P4 after Ch. 9** — the allocator is §9.9 end to end; the block pool, huge pages and copy-on-write are §9.6–9.8. Nothing here works if you only read half of Ch. 9.
- **P5 after P4** — by now you have read everything it uses; what it needs is a *second pass* through Ch. 3 (instruction encoding, calling convention, SIMD) with an encoder in mind, plus `mmap`/`mprotect` (§9.8) for W⊕X code pages.
- **P6 after Ch. 10–11** — A6.1 is a non-blocking `epoll` HTTP server: Unix I/O, short counts, RIO (§10.1–10.5) and sockets/HTTP (§11.1–11.6). Threads and the scheduler start at A6.3, which is when you need Ch. 12.

---

## 2. Assignment gates (fine-grained)

Read the listed sections **before** starting the assignment. "—" means nothing beyond the project gate.

### P1 `qnum` — gate: Ch. 2

| Asgn | Title | Budget | Read first |
|------|-------|--------|------------|
| A1.1 | Repository, harness, build | 6 h | spec Ch. 1 (toolchain, Make, Git fundamentals) |
| A1.2 | fp16 & bf16, three rounding modes | 12 h | §2.4.2–2.4.4 (IEEE format, rounding) |
| A1.3 | fp8 E4M3 / E5M2 | 8 h | §2.4 (subnormals, special values) |
| A1.4 | int8/int4 group quantization + BFP | 10 h | §2.2–2.3 (two's complement, overflow, shifts) |
| A1.5 | Measurement suite on a real checkpoint | 10 h | §2.1.3 (byte order — safetensors is little-endian) |
| A1.6 | Assembly study + hardware oracles | 8 h | **§3.1–3.6, §3.11** |

### P2 `cforward` — gate: Ch. 5, Ch. 6.1–6.7

| Asgn | Title | Budget | Read first |
|------|-------|--------|------------|
| A2.0 | Memory mountain, peak FLOP/s & bandwidth, roofline | 8 h | §6.6 (memory mountain), §5.7 |
| A2.1 | Reference forward pass | 14 h | — (can start right after P1) |
| A2.2 | The matvec ladder R1–R4 | 10 h | §5.7–5.11 (multiple accumulators, unrolling, SIMD) |
| A2.3 | Quantized weights via `qnum` | 10 h | P1 tagged `v0.1.0`+ |
| A2.4 | Attention (online softmax) and prefill | 12 h | §6.5 (cache-friendly code, blocking) |
| A2.5 | Hardware counters | 6 h | §5.12–5.14 (memory performance, limiting factors) |

### P3 `ltrace-ml` — gate: Ch. 7, Ch. 8.1–8.4

| Asgn | Title | Budget | Read first |
|------|-------|--------|------------|
| A3.1 | Allocation hooks + launcher | 10 h | §7.10–7.13, §8.4 |
| A3.2 | Stack walking; `LD_PRELOAD` vs `--wrap` | 8 h | §3.7 (stack frames), §7.13 |
| A3.3 | Sampling profiler, single-threaded | 10 h | **§8.5** (signals, async-signal safety) |
| A3.4 | Threads | 8 h | skim **§12.3–12.4** (look-ahead) |
| A3.5 | CUDA runtime interposition | 10 h | — |
| A3.6 | Overhead study | 5 h | — |
| A3.7 | Stretch: profile a WRF binary | 4 h | — |

### P4 `pagedkv` — gate: Ch. 9

| Asgn | Title | Budget | Read first |
|------|-------|--------|------------|
| A4.1 | The allocator (segregated fits, coalescing) | 14 h | §9.9 (all) |
| A4.2 | Block pool, pages and the TLB | 10 h | §9.6–9.8 |
| A4.3 | Block tables, fork, copy-on-write | 8 h | §9.8.2 (private COW objects) |
| A4.4 | Radix tree + online eviction policies | 10 h | — |
| A4.5 | Reuse distance, Belady's OPT, working set | 12 h | spec §"Reuse distance and optimal replacement" (an OS textbook's page-replacement chapter helps) |
| A4.6 | Cold tier and paged attention | 10 h | P2 A2.4 done |

### P5 `tinyjit` — gate: P4 done, Ch. 3 second pass

| Asgn | Title | Budget | Read first |
|------|-------|--------|------------|
| A5.1 | Encoder, golden-tested against `as` | 10 h | §3.2–3.6 (re-read) + spec's encoding table |
| A5.2 | IR, interpreter, first kernel | 8 h | §3.7 (calling convention), §9.8 (`mmap`, protections) |
| A5.3 | Register allocation, masked tail, unrolling, fuzzing | 12 h | §3.11 (SIMD registers) |
| A5.4 | Transcendentals + fusion benchmark | 12 h | Ch. 5 (re-skim) |
| A5.5 | Integer ops + quantizer kernels | 8 h | P1 complete |
| A5.6 | Stretch: relocatable ELF object | 8 h | **§7.4–7.7** (re-read: sections, symbols, relocations) |

### P6 `servlet` — gate: Ch. 10, Ch. 11

| Asgn | Title | Budget | Read first |
|------|-------|--------|------------|
| A6.1 | Reactor, buffers, parser, static endpoint | 14 h | §10.1–10.5, §11.1–11.6, §12.2 (I/O multiplexing) |
| A6.2 | Tokenizer + single-sequence completion | 8 h | — |
| A6.3 | Engine thread, queues, SSE streaming | 12 h | **§12.3–12.5** (threads, sharing, semaphores) |
| A6.4 | Continuous batching with chunked prefill | 14 h | §12.6 (prethreading) |
| A6.5 | Preemption + prefix-affinity scheduling | 10 h | P4 A4.4 |
| A6.6 | Load generator + latency–load study | 12 h | — |
| A6.7 | CI, releases, submodules, back-porting | 10 h | §12.7 (races, deadlocks) for the TSan campaign |

---

## 3. Month-by-month schedule

Reading runs roughly one chapter ahead of building, so each project's gate is already cleared when you reach it.

| Month | Read (weekday sessions) | Build (weekend sessions) |
|-------|-------------------------|--------------------------|
| 0 (2–3 wk) | Ch. 1, Ch. 2; **Data Lab** | Environment + A1.1 |
| 1 | Ch. 3.1–3.6, 3.11; **Bomb Lab** | P1: A1.2–A1.4 |
| 2 | Ch. 3 rest (Attack Lab optional), skim Ch. 4.1–4.5 | P1: A1.5–A1.6 → tag `v1.0.0`; start A2.1 |
| 3 | Ch. 5, Ch. 6.1–6.7; **Cache Lab** | P2: A2.1 (cont.), A2.0, A2.2 |
| 4 | Ch. 6 rest; Ch. 7 begins | P2: A2.3–A2.5 → blog posts 2/2b |
| 5 | Ch. 7, Ch. 8.1–8.4 | P3: A3.1–A3.2 |
| 6 | Ch. 8.5–8.6; **Shell Lab**; skim 12.3 | P3: A3.3–A3.7 → blog posts 3/3b |
| 7 | Ch. 9 (all) | P4: A4.1 (allocator) |
| 8 | Ch. 3 & 7.4–7.7 second pass | P4: A4.2–A4.6 → blog posts 4/4b |
| 9 | Ch. 10 | P5: A5.1–A5.3 |
| 10 | Ch. 11 | P5: A5.4–A5.6 → blog posts 5/5b |
| 11 | Ch. 12; Proxy Lab (optional) | P6: A6.1–A6.3 |
| 12 | Kerrisk (TLPI) as reference | P6: A6.4–A6.7 → blog posts 6/6b, all six repos released |

**Deadline weeks:** drop to 30-minute reading, no building. Don't skip the gate to "catch up" — a project started without its reading tends to need a rewrite.

## 4. Companion reading (optional, by project)

- **P1:** Knuth TAOCP Vol. 2 §4.2 (floating-point arithmetic) for rounding test cases; §3 for the stochastic-rounding RNG.
- **P2, P4:** Drepper, *What Every Programmer Should Know About Memory*.
- **P3, P4, P6:** Kerrisk, *The Linux Programming Interface* — the reference for every syscall used.
- **P4:** Knuth Vol. 1 §2.5 (dynamic storage allocation) — the source of boundary tags and the fifty-percent rule.
- **P5:** Hennessy & Patterson Ch. 2–4 for architecture vocabulary.
