# Experiments 

This page documents architectural explorations, design pattern evaluations, and optimization candidates benchmarked during the development.

---

## Exp-1 — Parser Dispatch: Function Pointer Table (`fptr`) vs. Switch-Case

### Overview

Message-type dispatch in the parser layer was evaluated: an indirect function pointer table (`fptr`, production) against a `switch-case` implementation. Code and pipeline are otherwise identical, both variants use the custom SPSC queue (`CustomQueue`). Both were measured back-to-back on 2026-10-31 (switch 21:54–22:07, `fptr` 22:11–22:19) with the `performance` governor. Benchmark values are medians; cycles are TSC ticks; CV is the run-to-run noise of the measurement.

---

### Part A: Micro-Benchmark Comparisons

#### 1. Mixed Traffic Dispatch (`parser_dispatch_bench`)

| Metric          | `fptr` (production) | Switch-Case | Delta | CV (fptr / switch) |
|-----------------|---------------------|-------------|-------|--------------------|
| **Median (ns)** | 1381                | 1405        | +1.7% | 4.9% / 3.0%        |
| **p50 cycles**  | 3731                | 3778        | +1.3% | 3.6% / 2.2%        |
| **p99 cycles**  | 5047                | 5297        | +5.0% | 5.2% / 4.1%        |
| **p999 cycles** | 13386               | 13593       | +1.5% | 1.1% / 0.9%        |

#### 2. Per-Protocol Parsers

| Parser / Scenario     | Metric       | `fptr` | Switch | Delta   | Note                       |
|-----------------------|--------------|--------|--------|---------|----------------------------|
| **OUCH BIST** (normal)| Median (ns)  | 2188   | 2244   | +2.6%   |                            |
|                       | p50 cycles   | 6026   | 6152   | +2.1%   |                            |
|                       | p99 cycles   | 8378   | 8710   | +4.0%   |                            |
| **ITCH BIST** (best)  | Median (ns)  | 140    | 140    | 0%      |                            |
|                       | p50 cycles   | 295    | 254    | (−13.9%)| noisy: p50 CV 8–16%        |
| **ITCH BIST** (worst) | Median (ns)  | 195    | 197    | +1.0%   |                            |
|                       | p50 cycles   | 417    | 434    | +4.1%   |                            |
|                       | p99 cycles   | 754    | 738    | −2.1%   |                            |
| **ITCH NASDAQ** (best)| Median (ns)  | 141    | 140    | −0.7%   |                            |
|                       | p50 cycles   | 300    | 260    | (−13.3%)| noisy: p50 CV 9–11%        |
| **ITCH NASDAQ** (worst)| Median (ns) | 205    | 210    | +2.4%   |                            |
|                       | p50 cycles   | 450    | 454    | +0.9%   |                            |
|                       | p99 cycles   | 820    | 784    | −4.4%   |                            |

> Values in parentheses are within the measurement noise of the scenario and are not treated as differences. p999 of the per-message parsers varies by 8–25% between repetitions and is not used for comparison.

---

### Part B: System Profiling & `perf` Metrics (`parser_dispatch_perf`)

One `perf` run per variant, same binary otherwise (switch 21:55, `fptr` 22:19). Values are P-core counters; E-core counters were sampled below 0.01% and are excluded.

| Perf Metric              | `fptr` (production) | Switch-Case     | Delta          |
|--------------------------|---------------------|-----------------|----------------|
| **Cycles**               | 122.12B             | 118.59B         | −2.9%          |
| **Instructions**         | 66.81B              | 66.30B          | −0.8%          |
| **IPC**                  | 0.547               | 0.559           | +2.2%          |
| **Branches**             | 14.043B             | 14.033B         | −0.1%          |
| **Branch miss rate**     | 0.320% (44.88M)     | 0.308% (43.25M) | −3.6%          |
| **L1-D load misses**     | 654.0M – 656.7M     | 654.6M – 662.7M | +0.1% – +0.9%  |
| **L1-I load misses**     | 8.27M               | 8.21M           | −0.8%          |
| **Local HITM (`c2c`)**   | 19,761              | 20,232          | +2.4%          |
| **Shared cache lines**   | 2,610               | 2,708           | +3.8%          |
| **User time**            | 27.5 s              | 26.6 s          | −3.1%          |

> Run-to-run noise, estimated from repeated measurements inside each run: L1-D load misses vary by 0.4–1.2% and elapsed time by 3–4% between `perf stat` invocations of the same binary. All differences above fall within this range.

In the `fptr` profile the message handlers appear as separate frames (called through pointers, not inlined); in the switch build they are inlined into the parser functions. The self time of `Parser_Dispatch::dispatch()` is the same in both variants (2.6% vs 2.7%).

---

### Decision

- **Result:** No measurable difference between `fptr` and switch-case dispatch. Benchmarks differ by 0–5% on stable metrics and `perf` counters by 0–4%, all within run-to-run noise. No scenario shows a clear improvement for switch-case.
- **Outcome:** **`fptr` remains in production.** Dispatch is a small share of the work (`dispatch()` ≈ 2.6% self time), so no dispatch mechanism can move end-to-end latency noticeably. The function pointer table keeps the handlers separate from the dispatch block and is easier to extend per message type.

---

## Exp-2 — MarketBook Data Structure: `absl::btree` vs. `std::vector`

### Overview
The core order book data structure within the standalone MarketBook component was evaluated to compare performance trade-offs between cache-friendly B-Tree node indexing (`absl::btree`) and contiguous flat-array storage (`std::vector`). 

Tests were conducted across realistic market replay workloads (64 symbols, up to 500 depth per side) under identical pipeline environments.

Tested layout: ascending sorted std::vector (best bid at the back, best ask at the front); a descending-ask layout was not evaluated, so the ask side shifts the whole array on top-of-book updates.

---

### Part A: Micro-Benchmark Comparisons (Baseline vs. Alternative)

#### MarketBook Standalone Performance (`MB_Mixed` Workload)
| Metric | `absl::btree` (Baseline / Win) | `std::vector` (Alternative) | Delta / Decision |
|---|---|---|---|
| **Median Latency (ns)** | 57.0 ns | 108.0 ns | **+89.5% (vector slower)** |
| **p50 cycles** | 160 cycles | 244 cycles | **+52.5%** |
| **p99 cycles** | 253 cycles | 1993 cycles | **+687.7%** |
| **p999 cycles** | 842 cycles | 2047 cycles | **+143.1%** |

---

### Part B: System Profiling & `perf` Metrics (`mb_standalone` Stress)

| Perf Metric | `absl::btree` (Baseline / Win) | `std::vector` (Alternative) | Impact / Observation |
|---|---|---|---|
| **CPU Cycles** | 82.99B | 80.73B | Equivalent execution runtime under benchmark loop bounds. |
| **Instructions Retired** | 81.97B | 229.65B | **~2.8x instruction bloat** in `std::vector` due to scalar/vector `memmove` loops. |
| **P-core IPC** | 0.99 | 2.84 | High vector IPC is deceptive: driven purely by tight hardware-accelerated memory-copy loops. |
| **Branches** | 19.19B | 82.61B | **~4.3x more branch instructions** in `std::vector` controlling shift loop bounds. |
| **Branch Miss Rate** | 5.61% (1.07B misses) | 0.41% (344.19M misses) | Tree traversal branch prediction is harder, but total executed instructions are vastly lower. |
| **L1-D Load Misses** | 2.35B misses | 3.95B misses | **+67.7% more L1 Data Cache misses** in `std::vector` due to array eviction. |
| **LLC Load Misses** | 432,530 | 267,079 | Minimal Last-Level Cache footprint for both implementations. |
| **dTLB Load Misses** | 3.01M | 62,501 | Node allocations in B-Tree add minor page crossings, negligible (<0.01%) on total runtime. |

---

### Decision

- **Result:** `std::vector` causes severe performance degradation in mixed order book operations (median latency **+89.5% slower**, p99 tail latency **~7x higher**) and inflates order book heap memory footprint by likely driven by the reserved capacity.

- **Why `std::vector` was rejected:** Maintaining sorted array invariants forces $O(N)$ linear memory shifts (`memmove`) on every order book update. Under deep book depths, this produces massive instruction bloat (229.6B vs 81.9B instructions retired) and pollutes L1 Data Caches. High hardware IPC in `vector` is an artifact of raw memory copying rather than productive business logic.

- **Outcome:** `std::vector` is rejected for production MarketBook usage; **`absl::btree` remains in production**.
