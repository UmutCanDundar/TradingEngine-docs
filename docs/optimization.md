# Recent Optimization 

The optimizations on this page were identified through profiling and benchmarking runs conducted against the baseline implementation. Perf results highlighted the bottlenecks; the changes below are the direct response to those findings.

---

## Opt-1 — OrderManager:: <br>`folly::ConcurrentHashMap` → `ConcurrentFixedFlatMap` <small>(2026-06-03)</small> 

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

## Opt-2 — Builder_FIX::transact_time: <br>`localtime_r` per call → precomputed second table <br><small>(2026-06-03)</small> 

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
