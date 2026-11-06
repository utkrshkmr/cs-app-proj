# cs-app-proj — six systems projects to build while reading CS:APP

Six C/C++ projects designed to be built alongside *Computer Systems: A Programmer's Perspective* (Bryant & O'Hallaron, 3rd ed.). Each one lands on the chapters you are reading at the time, and they compose: by the end you have a small CPU-side LLM inference stack where every layer was written and measured by you.

| # | Project | What it is | Start after (gate) | Lang | Months |
|---|---------|-----------|--------------------|------|--------|
| 1 | [`qnum`](projects/01-qnum/) | Low-precision numerics: fp16/bf16/fp8, int8/int4 group quant, block floating point | Ch. 1–2 | C | 1–2 |
| 2 | [`cforward`](projects/02-cforward/) | Llama-style forward pass, optimized step by step onto a measured roofline | Ch. 5, 6.1–6.7 | C | 3–4 |
| 3 | [`ltrace-ml`](projects/03-ltrace-ml/) | `LD_PRELOAD` tracer (malloc → CUDA runtime) + `SIGPROF` sampling profiler | Ch. 7, 8.1–8.4 | C | 5–6 |
| 4 | [`pagedkv`](projects/04-pagedkv/) | Allocator + paged KV cache + radix prefix cache + Belady/OPT analysis | Ch. 9 | C | 7–8 |
| 5 | [`tinyjit`](projects/05-tinyjit/) | AVX2 JIT compiler for fused tensor ops, optional ELF `.o` writer | P4 done; re-read Ch. 3, 7.4–7.7 | C | 9–10 |
| 6 | [`servlet`](projects/06-servlet/) | C++20 continuous-batching inference server over `epoll`, SSE streaming | Ch. 10–11 | C++20 | 11–12 |

**Start here:** [`ROADMAP.md`](ROADMAP.md) — exactly which chapters to finish before each project *and* before each assignment, which CS:APP lab to do first, and a month-by-month reading/building schedule.

**Web version:** [`index.html`](index.html) — a light/dark page explaining every project (served via GitHub Pages once enabled: *Settings → Pages → Deploy from branch → `main` / root*).

## How the projects compose

```
qnum (P1) ──► cforward (P2) ──► ltrace-ml (P3) traces it
                  │
                  ├──► pagedkv (P4): attention reads K/V through block tables
                  ├──► tinyjit (P5): replaces elementwise kernels, emits P1's quantizers
                  └──► servlet (P6): cforward + pagedkv + qnum as submodules, profiled by P3
```

## What's in this repo

| Path | Contents |
|------|----------|
| `ROADMAP.md` | When to attack each project; per-assignment chapter gates; lab order; 12-month schedule |
| `projects/NN-name/README.md` | One explainer per project: goal, why it matters, reading map, layout, assignments, tests, blog posts, Git lesson |
| `spec/` | Full LaTeX specification (89 pages): math derivations, normative headers, pseudocode, test plans, blog briefs, Git curriculum |
| `spec/csapp_six_projects.pdf` | The compiled spec |
| `index.html` | Single-file explainer page with light/dark theme |

Build the spec yourself:

```bash
cd spec && latexmk -pdf main.tex
```

## What's left to you

The spec does everything except the four things that are the point of doing it:

1. **Implementation** of the specified headers (C for P1–P5, C++20 for P6).
2. **Testing** — the tests listed in each project's test plan, green under ASan/UBSan/TSan.
3. **Documentation** — README, `docs/`, and the two blog posts per project.
4. **Pushing** — using the Git workflow taught in that project's Git lesson (it escalates from first commit to submodules + CI + releases).

## Pacing

~9 h/week (four 60-min weekday reading sessions + two 2–2.5 h weekend build sessions), ≈55–70 h per project, two months each, with P6 allowed to run long. Every assignment is closed by a commit whose message starts with its id (`A2.3: quantized matvec via qnum`).

Each project yields one main blog post and one short companion post, written measurement-first around a single central figure — see spec Chapter 1, *How to write the blog posts*.
