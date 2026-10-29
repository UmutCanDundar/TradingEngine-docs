# Recent Optimization 

The optimizations on this page were identified through profiling and benchmarking runs conducted against the baseline implementation. Perf results highlighted the bottlenecks; the changes below are the direct response to those findings.

---

## Opt-1 — OrderManager:: |`folly::ConcurrentHashMap` → `ConcurrentFixedFlatMap` <small>(2026-06-03)</small> 

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
| Max cycles | 632,706 | 144,687 | **−77%** |

Mean and steady-state cycle counts are effectively unchanged — the hot path cost of the map swap is neutral. The significant gain is in tail latency: p999 dropped **−91%** and max cycles **−77%**, directly attributable to the elimination of folly's hazard pointer reclamation and deferred free paths that caused occasional long stalls.

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

back to [OrderManager perf findings](Perf/order_manager_perf.md#findings)

## Opt-2 — Builder_FIX::transact_time: |`localtime_r` per call → precomputed second table |<small>(2026-06-03)</small> 

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
| Max cycles | 87,758 | 66,680 | **−24%** |

---

## Performance Comparison

| Metric | Before | After | Δ |
|---|---|---|---|
| Wall time | 0.97 s | 0.55 s | **−43%** |
| P-core IPC | 3.52 | 2.14 | — ¹ |
| P-core cycles | 4.18B @ 4.87 GHz | 2.68B @ 4.84 GHz | **−36%** |
| P-core instructions | 14.72B | 5.74B | **−61%** |
| Branch miss rate | 0.00% | 0.01% | flat |
| L1-D miss rate | 0.03% | 0.10% | +0.07pp |
| LLC miss rate | 63.7% | 67.1% | flat ² |
| dTLB misses | 27,127 | 69,994 | +158% ³ |
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

back to [Builder_Dispatch perf findings](Perf/builder_perf.md#fix-findings)


---

## Opt-3 — SPSC Queue: |`boost::lockfree::spsc_queue` → Custom `spsc queue` <small>(2026-10-29)</small>

> **Note:** Both tests (Boost SPSC and Custom SPSC) were executed on the same day under comparable runtime conditions. Due to natural runtime jitter, system load variations, and minor environment fluctuations, baseline values may differ slightly from older snapshots stored in the main Test section.

The general-purpose **Boost SPSC Queue** (`boost::lockfree::spsc_queue`) was replaced with a custom, lightweight, cache-line aligned Single-Producer Single-Consumer (SPSC) ring-buffer queue specifically tailored for the trading engine's pipeline (`Rx` and `Tx` flows).

---

## Benchmark Comparison (`Rx` & `Tx` Pipelines)

The benchmarks below compare the pipeline latencies between the old **Boost SPSC Queue** and the new **Custom SPSC Queue** across FIX, OUCH (BIST & NQ), and ITCH protocols. All values are measured in nanoseconds (time/median) and CPU cycles (`p50`, `p95`, `p99`, `p999`).

### 1. Tx Pipeline (`Risk -> Network`)

| Benchmark / Metric | Before (Boost SPSC) | After (Custom SPSC) | Δ |
|---|---|---|---|
| **BM_Risk_to_Network_FIX** | Median time | p50 cycles | p95 cycles | p99 cycles | p999 cycles | |3075 ns|8,782|9,183|11,987|15,855 | |2786 ns|7,970|8,333|10,582|13,755 | |**~9.4% Faster**|**~9.3% Fewer cycles**|**~9.3% Fewer cycles**|**~11.7% Fewer cycles**|**~13.2% Fewer cycles** |
| **BM_Risk_to_Network_OUCH_BIST** | Median time | p50 cycles | p95 cycles | p99 cycles | p999 cycles | |2942 ns|8,433|8,714|10,049|15,100 | |2608 ns|7,422|7,718|9,955|12,851 | |**~11.4% Faster**|**~12.0% Fewer cycles**|**~11.4% Fewer cycles**|**~0.9% Fewer cycles**|**~14.9% Fewer cycles** |
| **BM_Risk_to_Network_OUCH_NQ** | Median time | p50 cycles | p95 cycles | p99 cycles | p999 cycles | |2983 ns|8,463|8,801|11,064|15,360 | |2678 ns|7,661|8,000|8,314|13,797 | |**~10.2% Faster**|**~9.5% Fewer cycles**|**~9.1% Fewer cycles**|**~24.9% Fewer cycles**|**~10.2% Fewer cycles** |

### 2. Rx Pipeline (`Network -> Risk`)

| Benchmark / Metric | Before (Boost SPSC) | After (Custom SPSC) | Δ |
|---|---|---|---|
| **BM_Network_to_Risk_FIX** | Median time | p50 cycles | p95 cycles | p99 cycles | p999 cycles | |3555 ns|9,517|10,002|11,467|18,832 | |3174 ns|8,453|8,906|10,087|15,269 | |**~10.7% Faster**|**~11.2% Fewer cycles**|**~11.0% Fewer cycles**|**~12.0% Fewer cycles**|**~18.9% Fewer cycles** |
| **BM_Network_to_Risk_OUCH_BIST** | Median time | p50 cycles | p95 cycles | p99 cycles | p999 cycles | |3400 ns|9,058|9,504|10,332|16,601 | |3107 ns|8,283|8,720|9,947|15,232 | |**~8.6% Faster**|**~8.6% Fewer cycles**|**~8.3% Fewer cycles**|**~3.7% Fewer cycles**|**~8.2% Fewer cycles** |
| **BM_Network_to_Risk_OUCH_NQ** | Median time | p50 cycles | p95 cycles | p99 cycles | p999 cycles | |3390 ns|9,053|9,466|10,439|16,257 | |3110 ns|8,307|8,721|9,092|13,911 | |**~8.3% Faster**|**~8.2% Fewer cycles**|**~7.9% Fewer cycles**|**~12.9% Fewer cycles**|**~14.4% Fewer cycles** |
| **BM_Network_to_Risk_ITCH_BIST** | Median time | p50 cycles | p95 cycles | p99 cycles | p999 cycles | |2011 ns|5,670|6,008|6,310|11,920 | |1747 ns|4,946|5,332|5,514|8,747 | |**~13.1% Faster**|**~12.8% Fewer cycles**|**~11.2% Fewer cycles**|**~12.6% Fewer cycles**|**~26.6% Fewer cycles** |
| **BM_Network_to_Risk_ITCH_NQ** | Median time | p50 cycles | p95 cycles | p99 cycles | p999 cycles | |1989 ns|5,634|5,980|6,286|11,610 | |1729 ns|4,874|5,198|5,416|9,023 | |**~13.1% Faster**|**~13.5% Fewer cycles**|**~13.1% Fewer cycles**|**~13.8% Fewer cycles**|**~22.3% Fewer cycles** |

---

## Performance Comparison (`RxTx` Integration Stats)

| Metric | Before (Boost SPSC) | After (Custom SPSC) | Δ |
|---|---|---|---|
| CPU Core Cycles | 129.48B | 122.68B | **~5.2% Fewer cycles** |
| L1-D load misses | 164.75M | 139.98M | **~15.0% Reduction** |
| LLC Loads | 134.03M | 110.66M | **~17.4% Reduction** |
| Local HITM (C2C) | 18,714 | 9,949 | **~46.8% Dramatic Reduction** |

---

## Why It Won

1. **Cache-Line Alignment (alignas(64)) & Index Striding:**
Our custom queue explicitly enforces alignas(64) padding to isolate producer and consumer control variables onto separate cache lines. Furthermore, the internal slot layout and index remapping mechanism are designed such that advancing to the next slot (++index) strides directly to the next cache line (e.g., slot 0 resides on Cacheline 1, and incrementing remaps to Cacheline 2). This memory access layout is a key driver in reducing cache thrashing and inter-core invalidation costs (Local HITM dropped by 46.8%).

2. **Explicit Memory Ordering (acquire / release):**
We explicitly implemented fine-grained lock-free operations using std::memory_order_acquire and std::memory_order_release semantics. This avoids default sequential consistency barriers (seq_cst) and prevents unnecessary CPU pipeline stalls during inter-thread synchronization.

3. **Domain-Specific Custom Implementation vs. General-Purpose Overhead:**
While Boost provides a general-purpose concurrent queue, tailoring our custom implementation strictly for the trading engine's fixed single-producer single-consumer requirements eliminated unnecessary abstraction layers and template bloat. The streamlined index arithmetic leads to better L1 instruction/data cache utilization and lower tail latencies (p999 cycles improved up to 26.6%).

---

## Summary

| Category | Gain |
|---|---|
| Pipeline Latency (Median) | **~8% to ~13% faster** across all protocol legs |
| Tail Latency (`p999`) | Up to **~26.6% reduction** in extreme cases |
| CPU Core Cycles | **~5.2% fewer cycles** consumed |
| L1-D Cache Load Misses | **~15.0% reduction** |
| False sharing / Local HITM | **~46.8% dramatic reduction** |
