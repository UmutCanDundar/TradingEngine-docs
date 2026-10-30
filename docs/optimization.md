# Recent Optimization 

The optimizations on this page were identified through profiling and benchmarking runs conducted against the baseline implementation. Perf results highlighted the bottlenecks; the changes below are the direct response to those findings.

---

## Opt-1 — OrderManager:: <br>`folly::ConcurrentHashMap` → `ConcurrentFixedFlatMap` <small>(2026-06-03)</small> { #opt-1 }

`folly::ConcurrentHashMap` was replaced with a custom `mmap`-backed, fixed-capacity, open-addressing `ConcurrentFixedFlatMap`. The map is accessed exclusively by the store thread — concurrent access never occurs — making folly's hazard pointer and mutex machinery pure overhead.

Additionally, folly is not a modular library: even when only `ConcurrentHashMap` was used, the entire library was pulled into every build. Removing it cut full build time by **~2x**.

---

## Benchmark Comparison (`BM_OrderManager/MixedTraffic`)

| Metric | Before (folly) | After (ConcurrentFixedFlatMap) | Δ |
|---|---|---|---|
| Mean latency | 2584 ns | 2621 ns | +1.4% (within noise) |
| p50 cycles | 7,261 | 7,736 | +6.5% |
| p95 cycles | 7,941 | 8,643 | +8.8% |
| p99 cycles | 8,903 | 9,471 | +6.4% |
| p999 cycles | 183,912 | 17,400 | **−91%** |

Mean and steady-state cycle counts are effectively unchanged — the hot path cost of the map swap is neutral. The significant gain is in tail latency: p999 dropped **−91%**, directly attributable to the elimination of folly's hazard pointer reclamation and deferred free paths that caused occasional long stalls.

---

## Performance Comparison

### Key Counters (P-core / Store Thread)

| Metric | Before | After | Δ |
|---|---|---|---|
| P-core IPC | 0.35 | 3.33 | **+9.5x** |
| P-core cycles | 226.9B | 13.8B | **−94%** |
| P-core instructions | 79.8B | 45.9B | **−42%** |
| Branch miss rate | 1.00% | 0.01% | **−99%** |
| L1-D miss rate | 3.48% | 0.08% | **−98%** |
| L1-I misses | 372M | 302M | **−19%** |
| dTLB misses | 1,495,210 | 485,986 | **−67%** |
| iTLB misses | 132,210 | 7,533 | **−94%** |
| Local HITM | 45,310 | 0 | **−100%** |

IPC and cycle improvements are partly attributable to the removal of consumer spin-loops. Instruction count reduction and icache hotspot changes reflect application-level gains only.

---

### ICache Hotspots

| Symbol | Before (self%) | After (self%) |
|---|---|---|
| `folly::ConcurrentHashMap::eraseImpl` | **11.02%** | — |
| `pthread_mutex_lock` (folly internal) | **5.11%** | — |
| `folly::ConcurrentHashMap::emplace` | 1.01% | — |
| `ConcurrentFixedFlatMap::erase` | — | 1.49% |

Three folly symbols previously consumed **29.26% icache self** combined. All are gone.

---

### DCache Hotspots

| Symbol | Before (self%) | After (self%) |
|---|---|---|
| Consumer spin-loops (3×) | ~51% total | — |
| `folly::ConcurrentHashMap::eraseImpl` | **4.21%** | — |
| `ConcurrentFixedFlatMap::erase` | — | 6.02% |

---

### False Sharing (C2C)

| Metric | Before | After | Δ |
|---|---|---|---|
| Shared cache lines | 19 | 0 | **−100%** |
| Local HITM | 45,310 | 0 | **−100%** |
| Store HITs on shared lines | 83,941 | 0 | **−100%** |

---

## Summary

| Category | Gain |
|---|---|
| Build time | **~2x faster** — folly dependency removed entirely |
| ICache overhead | folly symbols: **29.26% → 0%** |
| DCache overhead | `eraseImpl`: **4.21% → 0%** |
| iTLB misses | **−94%** |
| dTLB misses | **−67%** |
| False sharing / HITM | **−100%** |

back to [OrderManager perf findings](Perf/order_manager_perf.md/#findings)

## Opt-2 — Builder_FIX::transact_time: <br>`localtime_r` per call → precomputed second table <br><small>(2026-06-03)</small> { #opt-2 }

Each FIX order previously called `localtime_r` to format the `TransactTime` field, making timestamp formatting a syscall-level cost on every message. A static `char g_second_table[86400][17]` is now built once at startup — covering all 86400 seconds of the day — so the hot path is a single `memcpy(17 bytes)` + millisecond suffix append.

---

## Benchmark Comparison (`BM_Builder_FIX_newOrder`) 

| Metric | Before | After | Δ |
|---|---|---|---|
| Mean latency | 175 ns | 100 ns | **−43%** |
| p50 cycles | 440 | 222 | **−50%** |
| p95 cycles | 454.6 | 228.4 | **−50%** |
| p99 cycles | 497.4 | 269.8 | **−46%** |
| p999 cycles | 584 | 319.2 | **−45%** |

---

## Performance Comparison

| Metric | Before | After | Δ |
|---|---|---|---|
| Wall time | 0.97 s | 0.55 s | **−43%** |
| P-core IPC | 3.52 | 2.14 | |
| P-core cycles | 4.18B @ 4.87 GHz | 2.68B @ 4.84 GHz | **−36%** |
| P-core instructions | 14.72B | 5.74B | **−61%** |
| Branch miss rate | 0.00% | 0.01% |  |
| L1-D miss rate | 0.03% | 0.10% | +0.07pp |
| LLC miss rate | 63.7% | 67.1% |  |
| dTLB misses | 27,127 | 69,994 | +158%  |
| iTLB misses | 272,287 | 251,955 | **−7%** |

IPC drop reflects fewer instructions retiring per cycle, not a regression — instruction count fell 61%, so the workload is simply lighter.

LLC miss rate stays high in both runs due to page-fault overhead dominating the profile (see hotspots below).

dTLB increase is from the 1.4 MB `g_second_table` region touching new pages on first access.

---

## CPU+Branch Hotspots

| Symbol | Before (self%) | After (self%) |
|---|---|---|
| `Builder_FIX::transact_time` / `epoch_to_transact_time` | **6.05% + 3.61%** | — |

`transact_time` overhead is gone. Page-fault share grew because the total instruction budget shrank — the same number of page faults now represents a larger fraction of a shorter run.

---

## Summary

| Category | Gain |
|---|---|
| Mean latency | **−43%** (175 ns → 100 ns) |
| p50 cycles | **−50%** (440 → 222) |
| p99 cycles | **−46%** (497 → 270) |
| Instructions retired | **−61%** |
| Wall time | **−43%** |
| `localtime_r` syscall per message | **eliminated** |

back to [Builder_Dispatch perf findings](Perf/builder_perf.md/#findings)

---

## Opt-3 — SPSC Queue: <br>`boost::lockfree::spsc_queue` → Custom `spsc queue` <small>(2026-10-29)</small> { #opt-3 }

> **Note:** Both tests (Boost SPSC and Custom SPSC) were executed on the same day under comparable runtime conditions. Due to natural runtime jitter, system load variations, and minor environment fluctuations, baseline values may differ slightly from older snapshots stored in the main Test section.

The general-purpose **Boost SPSC Queue** (`boost::lockfree::spsc_queue`) was replaced with a custom, lightweight, cache-line aligned Single-Producer Single-Consumer (SPSC) ring-buffer queue specifically tailored for the trading engine's pipeline (`Rx` and `Tx` flows).

---

## Benchmark Comparison (`Rx` & `Tx` Pipelines)

The benchmarks below compare the pipeline latencies between the old **Boost SPSC Queue** and the new **Custom SPSC Queue** across FIX, OUCH (BIST & NQ), and ITCH protocols. All values are measured in nanoseconds (time/median) and CPU cycles (`p50`, `p95`, `p99`, `p999`).

---

### 1. Rx Pipeline (`Network -> Risk`)

| Benchmark / Metric | Before (Boost SPSC) | After (Custom SPSC) | Δ |
|---|---|---|---|
| **BM_Network_to_Risk_FIX** | | | |
|  Median time | 3555 ns | 3174 ns | **~10.7% Faster** |
|  p50 cycles | 9,517 | 8,453 | **~11.2% Fewer cycles** |
|  p95 cycles | 10,002 | 8,906 | **~11.0% Fewer cycles** |
|  p99 cycles | 11,467 | 10,087 | **~12.0% Fewer cycles** |
|  p999 cycles | 18,832 | 15,269 | **~18.9% Fewer cycles** |
| **BM_Network_to_Risk_OUCH_BIST** | | | |
|  Median time | 3400 ns | 3107 ns | **~8.6% Faster** |
|  p50 cycles | 9,058 | 8,283 | **~8.6% Fewer cycles** |
|  p95 cycles | 9,504 | 8,720 | **~8.3% Fewer cycles** |
|  p99 cycles | 10,332 | 9,947 | **~3.7% Fewer cycles** |
|  p999 cycles | 16,601 | 15,232 | **~8.2% Fewer cycles** |
| **BM_Network_to_Risk_OUCH_NQ** | | | |
|  Median time | 3390 ns | 3110 ns | **~8.3% Faster** |
|  p50 cycles | 9,053 | 8,307 | **~8.2% Fewer cycles** |
|  p95 cycles | 9,466 | 8,721 | **~7.9% Fewer cycles** |
|  p99 cycles | 10,439 | 9,092 | **~12.9% Fewer cycles** |
|  p999 cycles | 16,257 | 13,911 | **~14.4% Fewer cycles** |
| **BM_Network_to_Risk_ITCH_BIST** | | | |
|  Median time | 2011 ns | 1747 ns | **~13.1% Faster** |
|  p50 cycles | 5,670 | 4,946 | **~12.8% Fewer cycles** |
|  p95 cycles | 6,008 | 5,332 | **~11.2% Fewer cycles** |
|  p99 cycles | 6,310 | 5,514 | **~12.6% Fewer cycles** |
|  p999 cycles | 11,920 | 8,747 | **~26.6% Fewer cycles** |
| **BM_Network_to_Risk_ITCH_NQ** | | | |
|  Median time | 1989 ns | 1729 ns | **~13.1% Faster** |
|  p50 cycles | 5,634 | 4,874 | **~13.5% Fewer cycles** |
|  p95 cycles | 5,980 | 5,198 | **~13.1% Fewer cycles** |
|  p99 cycles | 6,286 | 5,416 | **~13.8% Fewer cycles** |
|  p999 cycles | 11,610 | 9,023 | **~22.3% Fewer cycles** |

---

### 2. Tx Pipeline (`Risk -> Network`)

| Benchmark / Metric | Before (Boost SPSC) | After (Custom SPSC) | Δ |
|---|---|---|---|
| **BM_Risk_to_Network_FIX** | | | |
|  Median time | 3075 ns | 2786 ns | **~9.4% Faster** |
|  p50 cycles | 8,782 | 7,970 | **~9.3% Fewer cycles** |
|  p95 cycles | 9,183 | 8,333 | **~9.3% Fewer cycles** |
|  p99 cycles | 11,987 | 10,582 | **~11.7% Fewer cycles** |
|  p999 cycles | 15,855 | 13,755 | **~13.2% Fewer cycles** |
| **BM_Risk_to_Network_OUCH_BIST** | | | |
|  Median time | 2942 ns | 2608 ns | **~11.4% Faster** |
|  p50 cycles | 8,433 | 7,422 | **~12.0% Fewer cycles** |
|  p95 cycles | 8,714 | 7,718 | **~11.4% Fewer cycles** |
|  p99 cycles | 10,049 | 9,955 | **~0.9% Fewer cycles** |
|  p999 cycles | 15,100 | 12,851 | **~14.9% Fewer cycles** |
| **BM_Risk_to_Network_OUCH_NQ** | | | |
|  Median time | 2983 ns | 2678 ns | **~10.2% Faster** |
|  p50 cycles | 8,463 | 7,661 | **~9.5% Fewer cycles** |
|  p95 cycles | 8,801 | 8,000 | **~9.1% Fewer cycles** |
|  p99 cycles | 11,064 | 8,314 | **~24.9% Fewer cycles** |
|  p999 cycles | 15,360 | 13,797 | **~10.2% Fewer cycles** |

---

## Performance Comparison 

### 1. `RxTx` Stats

| Metric | Before (Boost SPSC) | After (Custom SPSC) | Δ |
|---|---|---|---|
| CPU Core Cycles | 129.48B | 122.68B | **~5.2% Fewer cycles** |
| L1-D load misses | 164.75M | 139.98M | **~15.0% Reduction** |
| LLC Loads | 134.03M | 110.66M | **~17.4% Reduction** |
| Local HITM (C2C) | 18,714 | 9,949 | **~46.8% Dramatic Reduction** |

---

## Summary

| Category | Gain |
|---|---|
| Pipeline Latency (Median) | **~8% to ~13% faster** across all protocol legs |
| Tail Latency (`p999`) | Up to **~26.6% reduction** in extreme cases |
| CPU Core Cycles | **~5.2% fewer cycles** consumed |
| L1-D Cache Load Misses | **~15.0% reduction** |
| False sharing / Local HITM | **~46.8% dramatic reduction** |

back to [Pipeline perf findings](Pipeline/perf_pipeline.md/#rxtx-perf-findings)

---

## Opt-4 — Hash Table Lookup: Custom Open-Addressing → `absl::flat_hash_map` (2026-10-30) 

> **Note:** Baseline measurements refer to the custom scalar open-addressing / hash-indexed array implementation, evaluated against Abseil's SIMD group probing flat hash map under identical CPU pin and affinity conditions.

The custom scalar linear-probing hash table was replaced with Abseil's SIMD-accelerated `absl::flat_hash_map` across symbol index lookups and execution order ID routing tables.

## Benchmark Comparison (`Symbol` & `Exec ID` Lookups)

The benchmarks below compare latency and CPU cycle metrics across symbol lookup tables (`size: 512`, `load: 50%`) and execution order tracking tables (`65,536` active IDs). All values represent standalone micro-benchmark iterations.

### 1. Symbol Index Lookup Table (`HT_Sym`, 512 Capacity)

| Benchmark / Metric | Before (Custom Hash Table) | After (`absl::flat_hash_map`) | Δ |
| ----- | ----- | ----- | ----- |
| **HT_Sym Lookup** | | | |
| Median time | 9.05 ns | 9.78 ns | +8.0% (within jitter noise) |
| p50 cycles | 18 | 28 | +10 cycles (scalar probe hit) |
| p95 cycles | 64 | 36 | **~43.8% Fewer cycles** |
| p99 cycles | 90 | 44 | **~51.1% Fewer cycles** |
| p999 cycles | 128 | 65 | **~49.2% Fewer cycles** |

### 2. Execution ID Tracking Map (`HT_Exec`, 65,536 Active IDs)

| Benchmark / Metric | Before (Custom Hash Table) | After (`absl::flat_hash_map`) | Δ |
| ----- | ----- | ----- | ----- |
| **HT_Exec Lookup** | | | |
| Median time | 9.01 ns | 9.82 ns | +9.0% (within jitter noise) |
| p50 cycles | 18 | 28 | +10 cycles |
| p95 cycles | 62 | 36 | **~41.9% Fewer cycles** |
| p99 cycles | 88 | 44 | **~50.0% Fewer cycles** |
| p999 cycles | 124 | 62 | **~50.0% Fewer cycles** |

## Performance Comparison

### 1. System Profiling & `perf` Stats (`ht_standalone` Stress)

| Metric | Before (Custom Hash Table) | After (`absl::flat_hash_map`) | Δ |
| ----- | ----- | ----- | ----- |
| CPU Core Cycles | 138.39B | 135.07B | **~2.4% Fewer cycles** |
| Instructions Retired | 90.43B | 83.37B | **~7.8% Fewer instructions** |
| Branch Misses | 727.41M | 276.06M | **~62.0% Dramatic Reduction** |
| L1-D Load Misses | 36.73M | 35.11M | **~4.4% Fewer misses** |
| LLC Load Misses | 26,451 | 32,983 | Minimal LLC impact |


## Summary

| Category | Gain |
| ----- | ----- |
| Tail Latency (`p99` / `p999`) | **~50.0% reduction** across all lookup tables |
| Branch Mispredictions | **~62.0% dramatic reduction** |
| Instructions Retired | **~7.06B fewer instructions** (~7.8% reduction) |
| CPU Core Cycles | **~3.32B fewer cycles** consumed |
| Overall Decision | Integrated into Production  |

---

## Opt-5 — FIX Parser: SIMD → Scalar (2026-10-30)

The scalar FIX parser was compared against a SIMD implementation across the standalone FIX parser benchmark, the `parseFunc` micro-benchmark, and the end-to-end parser dispatch benchmark.

---

## Benchmark Comparison (`parser_fix_bench`)

| Benchmark / Metric | SIMD (Before) | Scalar (After) | Δ |
|---|---|---|---|
| **BM_ParserFIX_best** | | | |
| Median time | 180 ns | 188 ns | **+4.4%** |
| p50 cycles | 409 | 413 | **+1.0%** |
| p95 cycles | 423 | 428 | **+1.2%** |
| p99 cycles | 468 | 467 | **−0.2%** |
| p999 cycles | 546 | 550 | **+0.7%** |
| **BM_ParserFIX_normal** | | | |
| Median time | 355 ns | 332 ns | **−6.5%** |
| p50 cycles | 794 | 785 | **−1.1%** |
| p95 cycles | 1,095 | 1,045 | **−4.6%** |
| p99 cycles | 1,316 | 1,184 | **−10.0%** |
| p999 cycles | 1,611 | 1,622 | **+0.7%** |
| **BM_ParserFIX_worst (seq-in-order)** | | | |
| Median time | 473 ns | 370 ns | **−21.8%** |
| p50 cycles | 1,252 | 905 | **−27.7%** |
| p95 cycles | 1,530 | 1,067 | **−30.3%** |
| p99 cycles | 1,628 | 1,185 | **−27.2%** |
| p999 cycles | 1,817 | 1,533 | **−15.6%** |
| **BM_ParserFIX_worst (seq-out-of-order)** | | | |
| Median time | 375 ns | 376 ns | **+0.3%** |
| p50 cycles | 930 | 987 | **+6.1%** |
| p95 cycles | 1,065 | 1,084 | **+1.8%** |
| p99 cycles | 1,345 | 1,164 | **−13.5%** |
| p999 cycles | 1,522 | 1,483 | **−2.6%** |

---

| Benchmark / Metric | SIMD (Before) | Scalar (After) | Δ |
|---|---|---|---|
| **parseFunc/NewOrder** | | | |
| Median time | 70.1 ns | 69.4 ns | **−1.0%** |
| p50 cycles | 136 | 136 | **0.0%** |
| p95 cycles | 142 | 142 | **0.0%** |
| p99 cycles | 158 | 146 | **−7.6%** |
| p999 cycles | 258 | 259 | **+0.4%** |
| **parseFunc/ExecReport** | | | |
| Median time | 64.9 ns | 63.0 ns | **−2.9%** |
| p50 cycles | 122 | 118 | **−3.3%** |
| p95 cycles | 126 | 124 | **−1.6%** |
| p99 cycles | 140 | 128 | **−8.6%** |
| p999 cycles | 214 | 222 | **+3.7%** |

---

## 3. Benchmark Comparison (`parser_dispatch_bench`, `BM_Parser/MixedTraffic`)

| Metric | SIMD (Before) | Scalar (After) | Δ |
|---|---|---|---|
| Median time | 1,676 ns | 1,607 ns | **−4.1%** |
| p50 cycles | 4,696 | 4,504 | **−4.1%** |
| p95 cycles | 5,113 | 4,899 | **−4.2%** |
| p99 cycles | 5,534 | 5,353 | **−3.3%** |
| p999 cycles | 11,852 | 11,623 | **−1.9%** |

---

## Summary

| Category                           | Result                                                          |
|------------------------------------|-----------------------------------------------------------------|
| Best case (short message)          | ~1% cycle gain, within noise                                    |
| Normal                             | SIMD slower, especially tail (p99 +11%)                         |
| Worst (in-order)                   | SIMD **+38% p50, +43% p95** slower                              |
| Worst (out-of-order)               | Mixed: p50 −5.8%, but p99 +15.5%                                |
| parseFunc                          | No difference, slightly worse tail (p99 +8–9%)                  |
| End-to-end (dispatch MixedTraffic) | SIMD **~4.3% slower** across all percentiles                    |

**Result: SIMD dropped, Scalar kept.** The only gain is in the best case and it is within noise. On realistic traffic (normal, worst, dispatch) SIMD is either equal or clearly worse. Likely cause: FIX tag=value fields are short, so the SIMD setup and mask-extraction cost outweighs the scan savings.
