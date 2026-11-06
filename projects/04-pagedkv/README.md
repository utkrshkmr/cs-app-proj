# P4 · `pagedkv` — a memory allocator and a paged KV cache with prefix caching

> **Start after:** CS:APP Ch. 9 complete (A4.1 *is* Malloc Lab, done properly) · **Also needs:** P2 A2.4 before A4.6 · **Language:** C · **Budget:** ≈64 h (months 7–8) · **Spec:** `spec/05-p4-pagedkv.tex`

## What you build

Two layers.

1. **A general-purpose allocator** — Malloc Lab taken to completion: segregated explicit free lists, boundary-tag coalescing, in-place `realloc`, utilization and throughput measured against glibc.
2. **A vLLM-style paged KV-cache manager** on `mmap`'d memory: fixed-size KV blocks, per-sequence block tables, reference counts with **copy-on-write** for forked sequences (beam search, parallel sampling), a **radix tree over token prefixes** for prefix caching, pluggable eviction (LRU, LFU, depth-aware LRU), a cold tier on disk, and `cforward`'s attention adapted to read K/V through block tables.

Plus an **offline replacement analysis**: record the block-reference trace, compute reuse distances, and compare every online policy against **Belady's optimal (OPT)**.

## Why it matters

This is the data structure behind PagedAttention and SGLang's RadixAttention. Attaching a replacement-policy analysis lets you ask the operating-systems question — *how far is a practical eviction policy from the offline optimum?* — for KV blocks instead of pages. The capacity-limited block pool is the same object as a memory bank in memory-slot experiments.

## The core ideas (derived in the spec)

- **KV-cache geometry.** Bytes per token = `2 · layers · kv_heads · head_dim · bytes_per_elem`; block size `B` tokens trades internal fragmentation (≤ `B−1` tokens per sequence) against table size.
- **Sharing invariants.** A block's refcount equals the number of block tables plus radix nodes pointing at it; a write to a shared block copies first; nothing with refcount > 1 is evicted.
- **Reuse distance & Mattson's stack algorithm.** One pass computing the reuse-distance histogram `H(d)` gives the LRU miss-ratio curve for *every* capacity at once: `miss_LRU(C) = (n_∞ + Σ_{d≥C} H(d)) / n`. A Fenwick tree makes it `O(n log n)`.
- **Belady's MIN.** Evict the block used furthest in the future — offline only, but a lower bound for every policy. KV-specific twists: the tree-constrained OPT must evict leaves first, and pinned blocks shrink effective capacity. FIFO's Belady anomaly is a test case.
- **Pages and the TLB.** TLB reach = entries × page size; 4 KB vs 2 MB pages changes `dTLB-load-misses` measurably on attention gathers.

## Repository layout

```
pagedkv/
  include/pagedkv/  malloc.h pool.h blocktable.h radix.h tier.h kvcache.h reuse.h
  src/              malloc.c pool.c blocktable.c radix.c tier.c kvcache.c reuse.c
  tests/            test_malloc.c test_pool.c test_blocktable.c test_radix.c test_tier.c test_kvcache.c test_props.c
  bench/            mdriver.c tlb.c tier_bw.c mrc.c
  scripts/          gen_traces.py model_check.py plots
```

## Assignments

| Id | Title | Budget | Read first | Done when |
|----|-------|--------|------------|-----------|
| A4.1 | The allocator | 14 h | §9.9 | utilization ≥ 0.85 on classic traces; throughput within 3× glibc |
| A4.2 | Block pool, pages and the TLB | 10 h | §9.6–9.8 | huge-page support + the TLB experiment |
| A4.3 | Block tables, fork, copy-on-write | 8 h | §9.8.2 | sharing semantics proven by a model-based test |
| A4.4 | Radix tree + online eviction policies | 10 h | — | hit-rate benchmark on agent-style workloads |
| A4.5 | Reuse distance, OPT, working set | 12 h | spec §Reuse distance | the policies-vs-OPT miss-ratio figure |
| A4.6 | Cold tier + paged attention | 10 h | P2 A2.4 | paged attention bit-identical to contiguous |

## How it's tested

Allocator trace replay with payload patterns and heap checks after every op, coalescing cases, pool exhaustion/refill, COW semantics, radix `match` vs brute-force LCP, a 10⁵-operation **model-based test** against a Python reference, reuse-distance vs brute force, histogram-derived LRU = simulated LRU for every `C`, the Belady-anomaly trace (FIFO: 9 misses at C=3, 10 at C=4), OPT ≤ every policy, tier integrity, and paged = contiguous attention bit-for-bit.

## Blog posts

- **Post 4 — "Paging a KV cache on a CPU: what the TLB thinks of PagedAttention."**
- **Post 4b — "Belady's MIN for KV blocks: how far is LRU from optimal?"** Design output: the smallest pool at which LRU is within 10 % of OPT.

## Git lesson

Pull requests against yourself, branch protection, conventional commits, `revert` vs `reset` on public history, `cherry-pick`, `rebase --onto`, GitHub Releases.

**Uses:** P2. **Feeds:** P6 (KV-block-aware admission and prefix-affinity scheduling).
