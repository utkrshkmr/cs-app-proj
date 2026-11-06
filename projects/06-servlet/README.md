# P6 · `servlet` — a continuous-batching inference server

> **Start after:** CS:APP Ch. 10 and Ch. 11 · **Read while building:** Ch. 12 before A6.3 · **Optional warm-up:** Proxy Lab · **Language:** C++20 · **Budget:** ≈80 h (months 11–12, may run long) · **Spec:** `spec/07-p6-servlet.tex`

## What you build

A C++20 server that exposes `cforward` + `pagedkv` over **HTTP/1.1 with token streaming (server-sent events)**, built on:

- a non-blocking **`epoll` reactor** with timers and `eventfd` wakeups,
- an incremental HTTP parser that survives byte-at-a-time input and a fuzzer,
- a dedicated **engine thread** fed through SPSC/MPMC queues,
- an **iteration-level ("continuous batching") scheduler** with chunked prefill, KV-block-aware admission, preemption, and prefix-sharing-aware batching,
- a **load generator** (Poisson arrivals, trace replay, multi-agent workloads) measuring TTFT, TPOT, throughput and goodput under SLOs.

It ships race-free under ThreadSanitizer, with CI, versioned releases, and the earlier projects as pinned **Git submodules**.

## Why it matters

It's a testbed for scheduling and caching ideas: when you later implement something in vLLM or SGLang, you'll already have measured the elementary version here, where every queue and lock is yours. The multi-agent prefix-sharing experiment is the small-scale version of cross-agent KV reuse.

## The core ideas (explained in the spec)

- **Non-blocking I/O (Ch. 10, 12.2).** Short counts are normal; buffers absorb partial reads/writes; backpressure stops reading when output queues grow.
- **HTTP/1.1 + SSE.** A state machine; correctness = the *split-anywhere* property (any split of the bytes parses identically).
- **Continuous batching.** Each iteration: admit waiting requests while KV blocks and a token budget allow, split long prompts into prefill chunks, decode one token for every running sequence, preempt (and later resume) when the pool fills.
- **Performance model.** Step time ≈ `t_w + B·t_tok`: a fixed weight-read cost amortized over batch size `B` — which is why batching raises throughput until compute or KV capacity binds, and where the latency–load knee comes from.

## Repository layout

```
servlet/
  third_party/   qnum/ cforward/ pagedkv/   (git submodules, pinned)
  include/servlet/
    net/        reactor.hpp buffer.hpp
    http/       parser.hpp response.hpp sse.hpp
    sched/      scheduler.hpp policy.hpp
    engine/     engine.hpp tokenizer.hpp
    util/       spsc.hpp mpmc.hpp metrics.hpp json.hpp
  src/           main.cpp + one .cpp per header
  bench/loadgen/ loadgen.cpp report.py
  tests/         test_parser.cpp test_reactor.cpp test_queues.cpp test_scheduler.cpp test_tokenizer.cpp test_e2e.py
  fuzz/          fuzz_parser.cpp corpus/
  .github/workflows/ci.yml
```

## Assignments

| Id | Title | Budget | Read first | Done when |
|----|-------|--------|------------|-----------|
| A6.1 | Reactor, buffers, parser, static endpoint | 14 h | §10.1–10.5, §11.1–11.6, §12.2 | survives 10K idle connections, slow clients, byte-at-a-time requests |
| A6.2 | Tokenizer + single-sequence completion | 8 h | — | `POST /v1/completions` through the real model |
| A6.3 | Engine thread, queues, SSE streaming | 12 h | §12.3–12.5 | streaming, race-free under TSan |
| A6.4 | Continuous batching with chunked prefill | 14 h | §12.6 | block-aware admission; the `t(B)` measurement |
| A6.5 | Preemption + prefix-affinity scheduling | 10 h | P4 A4.4 | correct when the pool fills; multi-agent benchmark |
| A6.6 | Load generator + latency–load study | 12 h | — | TTFT/TPOT/goodput curves vs arrival rate |
| A6.7 | CI, releases, submodules, back-porting | 10 h | §12.7 | six repositories at tagged releases with green CI |

## How it's tested

Parser state tests plus the split-anywhere property and a libFuzzer target, mock-fd partial-write tests, a Python harness that abuses connections (byte-by-byte, half-close, mid-stream disconnect) and checks for fd leaks, 10⁷-item queue stress under TSan, a **deterministic fake engine** that makes the scheduler testable without a model, batch-independence of greedy outputs, tokenizer round trips on UTF-8 edge cases, and a soft p99 SLO load test.

## Blog posts

- **Post 6 — "Continuous batching from first principles: a CPU inference server you can read in an afternoon."**
- **Post 6b — "Finding data races in a server with ThreadSanitizer, and the one that TSan could not see."**

## Git lesson

Submodules, CI with GitHub Actions, semantic versioning and CHANGELOG, releases with binaries, packfiles and `gc`.

**Uses:** P1, P2, P4 (submodules), P3 (profiling).
