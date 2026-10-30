# Experiments 

This page documents architectural explorations, design pattern evaluations, and optimization candidates benchmarked during the development.

---

## Exp-1 — Parser Dispatch and Protocol Parsers: Function Pointer Table (`fptr`) vs. Switch-Case 

### Overview
Message type dispatching across all binary and text protocol parsers (Mixed traffic, ITCH BIST, ITCH NASDAQ, and OUCH BIST) was evaluated to compare the performance trade-offs between static branch-based routing (`switch-case`) and indirect function pointer (`fptr`) dispatch.

*Note: Both implementations run on the identical SPSC pipeline (`boost::lockfree` queue), ensuring queue-induced cache metrics remain baseline-equivalent.*

---

### Part A: Micro-Benchmark Comparisons (Before vs. After)

#### 1. Mixed Traffic Dispatch (`parser_dispatch_bench`)
| Metric | Before (`fptr` Baseline) | After (Switch-Case) | Delta / Decision |
|---|---|---|---|
| **Median (ns)** | 1460 ns | 1585 ns | +8.6% |
| **p50 cycles** | 4030 cycles | 4413 cycles | +9.5% |
| **p99 cycles** | 5094 cycles | 5426 cycles | +6.5% |
| **p999 cycles** | 7938 cycles | 11204 cycles | +41.1% |

#### 2. ITCH BIST Protocol Parser (`ParserITCH_BIST`)
| Scenario / Metric | Before (`fptr` Baseline) | After (Switch-Case) | Delta / Decision |
|---|---|---|---|
| **Best p50 cycles** | 246 cycles | 312 cycles | +26.8% |
| **Best p999 cycles** | 383 cycles | 534 cycles | +39.4% |
| **Worst p50 cycles** | 573 cycles | 752 cycles | +31.2% |
| **Worst p999 cycles** | 1149 cycles | 1254 cycles | +9.1% |

#### 3. ITCH NASDAQ Protocol Parser (`ParserITCH_NASDAQ`)
| Scenario / Metric | Before (`fptr` Baseline) | After (Switch-Case) | Delta / Decision |
|---|---|---|---|
| **Best p50 cycles** | 228 cycles | 312 cycles | +36.8% |
| **Best p999 cycles** | 350 cycles | 421 cycles | +20.3% |
| **Worst p50 cycles** | 556 cycles | 758 cycles | +36.3% |
| **Worst p999 cycles** | 1027 cycles | 1119 cycles | +9.0% |

#### 4. OUCH BIST Protocol Parser (`ParserOUCH_BIST`)
| Scenario / Metric | Before (`fptr` Baseline) | After (Switch-Case) | Delta / Decision |
|---|---|---|---|
| **Best p50 cycles** | 1020 cycles | 1109 cycles | +8.7% |
| **Best p999 cycles** | 1450 cycles | 1653 cycles | +14.0% |
| **Normal p50 cycles** | 7800 cycles | 8508 cycles | +9.1% |
| **Normal p999 cycles** | 14200 cycles | 15617 cycles | +10.0% |

---

### Part B: System Profiling & `perf` Metrics (`parser_dispatch_perf`)

| Perf Metric | `fptr` Dispatch (Baseline) | Switch-Case Dispatch | Impact / Observation |
|---|---|---|---|
| **Wall Time / User Time** | ~10.36 s / ~30.65 s | ~10.91 s / ~32.28 s | Switch-case increases total CPU execution time (~5% higher user time). |
| **Branch Misses** | 50.66M | Higher branch variance | Switch structures introduce heavier branch predictor strain across diverse message IDs. |
| **L1-D Load Misses** | ~742.4M | ~742.5M | Equivalent queue buffer footprint, dominated by SPSC ring-buffer writes. |
| **dTLB Load Misses** | ~1.72M | ~1.72M | Identical queue span and page boundary crossing overhead. |
| **Local HITM (False Sharing)** | 27,085 | 27,085 | Driven entirely by queue internal states (identical SPSC mechanism). |

---

### Engineering Rationale & Decision

* **Result:** While both architectures share identical inter-thread queue overheads, static `switch-case` routing causes measurable performance degradation (~8.6% to ~36.8% higher p50 cycles across micro-benchmarks and ~5% higher total CPU user time in full pipeline `perf` runs), alongside severe tail latency spikes (p999 up to +41%).
* **Why it was rejected:** Large `switch-case` blocks over frequent message type discriminators increase branch misprediction penalties and instruction cache pressure in hot loops. Function pointers (`fptr`) provide clean, predictable, and isolated execution routing without bloating the dispatch block.
* **Outcome:** Switch-case dispatch is rejected across the board; **`fptr` routing remains in production**.

---

## Exp-2 — MarketBook Data Structure: `absl::btree` vs. `std::vector`

### Overview
The core order book data structure within the standalone MarketBook component was evaluated to compare performance trade-offs between cache-friendly B-Tree node indexing (`absl::btree`) and contiguous flat-array storage (`std::vector`). 

Tests were conducted across realistic market replay workloads (64 symbols, up to 500 depth per side) under identical pipeline environments.

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

### Engineering Rationale & Decision

- **Result:** `std::vector` causes severe performance degradation in mixed order book operations (median latency **+89.5% slower**, p99 tail latency **~7x higher**) and inflates order book heap memory footprint by **~18x** (35.37 MB vs 1.97 MB).

- **Why `std::vector` was rejected:** Maintaining sorted array invariants forces $O(N)$ linear memory shifts (`memmove`) on every order book update. Under deep book depths, this produces massive instruction bloat (229.6B vs 81.9B instructions retired) and pollutes L1 Data Caches. High hardware IPC in `vector` is an artifact of raw memory copying rather than productive business logic.

- **Outcome:** `std::vector` is rejected for production MarketBook usage; **`absl::btree` remains in production**.
