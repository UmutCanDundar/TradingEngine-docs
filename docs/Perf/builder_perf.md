# Builder — Perf Analysis

End-to-end profiling of the Builder unit across four test scenarios: **Builder Dispatch** (mixed protocol traffic through the dispatch layer), **Builder FIX** (FIX protocol message construction in isolation), **Builder OUCH BIST** (OUCH BIST venue message construction in isolation), and **Builder OUCH NASDAQ** (OUCH NASDAQ venue message construction in isolation).

The Dispatch test is the primary production-representative scenario. The protocol-specific tests isolate per-path contribution and confirm per-format characteristics without cross-protocol interference. The isolated tests exercise a narrow, single-message dataset and complete quickly — their value is in per-function branch/cache behavior, not in absolute throughput figures.

---

## Test Setup

All perf tests were conducted under **Profile 1 — Performance/Idle**. For the full profile comparison and selection rationale, see [Pipeline Overview](pipeline.md#profile-selection).

| Field | Value |
|---|---|
| CPU | 13th Gen Intel Core i7-13620H (10C/16T) |
| P-core max | 4900 MHz |
| E-core max | 3600 MHz |
| Kernel | 6.17.0-29-generic x86_64 |
| Governor | performance |
| NUMA | Single node, 15714 MB total |

---

## At a Glance

| Metric | Dispatch | FIX | OUCH BIST | OUCH NASDAQ |
|---|---|---|---|---|
| Wall time | 1.09 s | 0.97 s | 0.34 s | 0.32 s |
| CPUs utilized | 0.999 | 0.886 | 0.997 | 0.985 |
| P-core IPC | 3.45 | 3.52 | 2.75 | 2.87 |
| Branch miss rate | 0.00% | 0.00% | 0.01% | 0.01% |
| L1-D miss rate | 0.03% | 0.03% | 0.08% | 0.06% |
| LLC miss rate | ~0% (absolute) | ~0% (absolute) | ~0% (absolute) | ~0% (absolute) |
| Local HITM | 0 | 0 | 0 | 0 |

All scenarios are compute-bound and contention-free — zero HITM across the board, negligible LLC pressure, and L1-D miss rates well below 0.1%. The Builder unit's working set (Order structs, output buffers, dispatch table) fits comfortably in L1 across all paths.

FIX carries the highest IPC in isolation (3.52) with all data L1-resident. The OUCH scenarios show slightly lower IPC (2.75–2.87); branch miss at 0.01% and L1-D miss at 0.06–0.08% confirm the OUCH build loops execute cleanly. The two actionable findings (checksum accumulation and timestamp syscall) are FIX-path specific and are isolated in findings 2 and 3 of the Dispatch section.

---

## Results (Builder Dispatch)

Mixed protocol traffic routed through `Builder_Dispatch::dispatch()`. This is the production-representative scenario — the dispatch layer selects the correct builder per message type and protocol, exercising the full builder surface under realistic ordering.

| Field | Value |
|---|---|
| Wall time | 1.09 s |
| User time | 0.93 s |
| Sys time | 0.16 s |
| CPUs utilized | 0.999 |

Single-threaded execution — CPUs at ~1.0 confirms no worker thread is active in this run.

---

### Key Metrics

| Metric | Value | Note |
|---|---|---|
| Instructions | 18.1B | — |
| Cycles | 5.26B @ 4.84 GHz | P-core near boost |
| IPC | 3.45 | Compute-lean dispatch path |
| Branch miss rate | 0.00% | Excellent |
| L1-D miss rate | 0.03% | Healthy |
| LLC miss rate | 70.02% of 571K loads | Startup artifact — absolute miss count (~400K) does not reflect steady-state; L1-D miss rate is the representative figure |

**IPC at 3.45 on P-core reflects a compute-lean dispatch path with good instruction throughput.** Unlike the pipeline's memory-bound SPSC queue pattern (IPC ~0.46), Builder Dispatch performs message field formatting and protocol routing — an instruction-rich, low-memory-pressure workload.

**Branch miss rate at 0.00%** and **L1-D miss rate at 0.03%** confirm the dispatch and build path executes cleanly against an L1-resident working set.

---

### Hotspot Analysis

Overhead is the inclusive cost of a function and all its callees. Self is the cost of the function's own instructions only — the actionable signal.

#### P-Core (CPU+Branch)

| Overhead | Self | Symbol |
|---|---|---|
| ~75% | ~0% | `libc` / `libstdc++` thread infrastructure |
| 99.73% | 0.28% | `main` |
| 81.08% | 0.49% | `Builder_Dispatch::dispatch()` |
| 69.53% | 5.23% | `Builder_Dispatch::buildFIX(Order*)` |
| 63.33% | 0.82% | `Builder_Dispatch::buildFIX_app(Order*, unsigned char)` |
| 52.87% | 2.91% | `Buffer_FIX* Builder_FIX::build<(FIXTypes)68, Order&>(...)` |
| 25.93% | 0.95% | `Builder_FIX::buildHeader(...)` |
| 25.01% | 4.51% | `Builder_FIX::transact_time()` |
| 17.83% | 1.63% | `Builder_FIX::buildNewOrderSingle(Order const&, char*, unsigned char)` |
| 16.26% | 3.12% | `Builder_FIX::epoch_to_transact_time(unsigned long)` |
| 10.96% | 0.84% | `OrderManager::add_awaitingAck_order(Order&)` |
| 9.45% | 9.45% | `Builder_FIX::finalizeChecksum(char*, unsigned long)` |

`finalizeChecksum` at 9.45% self is the single largest CPU consumer in application code — it is a leaf function with no callees and carries the full cost of the FIX checksum accumulation loop on every message. `transact_time()` at 4.51% self and `epoch_to_transact_time` at 3.12% self together represent 7.63% CPU self — the timestamp formatting cost, consistent with the `localtime_r` syscall path identified in finding 3. `buildFIX` at 5.23% self routes into the FIX build chain; from there the call path descends through `buildFIX_app` (0.82% self) → `Builder_FIX::build<>` (2.91%) → `buildNewOrderSingle` (1.63%) → `buildHeader` (0.95%).

`dispatch()` at 0.49% self confirms the routing layer remains lightweight.

#### ICache Profile

The actionable icache signal is in the CPU+Branch self-time column. `finalizeChecksum` (9.45%), `transact_time` (4.51%), and `epoch_to_transact_time` (3.12%) are the top three application self-time consumers and are the correct optimization targets.

---

### Cache & Memory

| Metric | Value | Assessment |
|---|---|---|
| L1-D miss rate (P-core) | 0.03% | Healthy — application working set L1-resident |
| LLC miss rate (P-core) | 70.02% of 571K loads | Startup artifact — not representative |
| L1-I misses (P-core) | 1,191,997 | — |
| dTLB misses (P-core) | 34,350 | Low |
| iTLB misses (P-core) | 213,482 | — |

L1-D miss rate at 0.03% confirms the Builder Dispatch working set (Order structs, FIX buffer, dispatch table) fits comfortably in L1.

---

### False Sharing (C2C)

| Metric | Value |
|---|---|
| Total shared cache lines | 0 |
| Load HITs on shared lines | 0 |
| Local HITM | 0 |
| Locked accesses on shared lines | 0 |
| Blocked accesses on shared lines | 0 |

**Zero shared cache lines and zero HITM.** Single-threaded execution produces no cross-core cache line contention.

---

### Flamegraph

[Builder Dispatch Flamegraph](../assets/flamegraph_builder.svg){target="_blank"}

---

### Findings & Action Items { #builder-dispatch-findings }

| # | Finding | Impact | Status |
|---|---|---|---|
| 1 | IPC 3.45 — compute-lean dispatch, healthy instruction throughput for field formatting workload | Expected characteristic | ✅ No action needed |
| 2 | `finalizeChecksum` 9.45% CPU self — full buffer iteration on every message, largest single application self-time entry | Unnecessary CPU cost on every FIX message | ⚠️ Accumulate running checksum during field writes, remove final pass |
| 3 | `transact_time` + `epoch_to_transact_time` 7.63% CPU self — `localtime_r` syscall on every message | Per-message syscall overhead on hot path | ⚠️ Cache `tm` struct, refresh only on second boundary <br>**(handled - see** [Opt-2 in Recent Optimizations Page](../optimization.md/#opt-2)**)** |
| 4 | Zero HITM / zero shared cache lines — single-threaded run, no contention possible | Expected | ✅ No action needed |

---

## Results (Builder FIX)

FIX protocol message construction in isolation — `Builder_FIX::build<FIXTypes::NewOrderSingle>()` called directly, no dispatch overhead, no OUCH paths active. Used to characterise pure FIX build performance and isolate the contribution of FIX to Dispatch numbers. Single order object, single message type, 5M iterations.

| Field | Value |
|---|---|
| Wall time | 0.97 s |
| User time | 0.71 s |
| Sys time | 0.15 s |
| CPUs utilized | 0.886 |

Single-threaded execution.

---

### Key Metrics

| Metric | Value | Note |
|---|---|---|
| Instructions | 14.7B | — |
| Cycles | 4.18B @ 4.87 GHz | P-core near boost |
| IPC | 3.52 | Compute-bound, slightly higher than Dispatch — no routing overhead |
| Branch miss rate | 0.00% | Essentially zero mispredictions |
| L1-D miss rate | 0.03% | Healthy — data hot in L1 |
| LLC miss rate | 63.74% of 505K loads | Startup artifact — not representative |

**IPC at 3.52 on P-core confirms FIX building is compute-bound with good throughput.** With data largely L1-resident (0.03% miss rate) and near-zero branch mispredictions, the P-core executes efficiently through the FIX field serialisation, checksum accumulation, and timestamp formatting loop. The IPC is slightly higher than Dispatch (3.45) because there is no dispatch routing overhead diluting the build work — the call path goes directly into the build functions.

---

### Hotspot Analysis

#### P-Core (CPU+Branch)

| Overhead | Self | Symbol |
|---|---|---|
| 88.21% | 3.71% | `main` |
| 33.16% | 1.31% | `Builder_FIX::buildHeader(...)` |
| 31.85% | 6.05% | `Builder_FIX::transact_time()` |
| 21.15% | 2.62% | `Builder_FIX::buildNewOrderSingle(Order const&, char*, unsigned char)` |
| 18.42% | 3.61% | `Builder_FIX::epoch_to_transact_time(unsigned long)` |
| 11.00% | 11.00% | `Builder_FIX::finalizeChecksum(char*, unsigned long)` |

`finalizeChecksum` at 11.00% self is the dominant application leaf — the complete checksum accumulation loop cost with no callees. `transact_time()` at 6.05% self and `epoch_to_transact_time` at 3.61% self together represent 9.66% CPU self — the timestamp formatting cost is proportionally more visible here than in Dispatch (7.63%) because there is no dispatch routing overhead diluting the samples. `buildNewOrderSingle` at 2.62% self and `buildHeader` at 1.31% self are the field serialisation contributions.

In isolation, `finalizeChecksum` and the timestamp chain together account for over 20% of P-core CPU self — confirming findings 2 and 3 from Dispatch are FIX-path specific and originate entirely within this build loop.

---

### Cache & Memory

| Metric | Value | Assessment |
|---|---|---|
| L1-D miss rate (P-core) | 0.03% | Identical to Dispatch — FIX working set fully L1-resident |
| LLC miss rate (P-core) | 63.74% of 505K loads | Startup artifact — not representative |
| L1-I misses (P-core) | 827,117 | Lower than Dispatch (1,191,997) — no routing overhead icache contribution |
| dTLB misses (P-core) | 27,127 | Low |
| iTLB misses (P-core) | 272,287 | Elevated — `localtime_r` kernel text pressure confirming Dispatch finding 3 |

**iTLB at 272,287 remains elevated**, consistent with `localtime_r` kernel text page pressure on every message call — confirming Dispatch finding 3.

---

### False Sharing (C2C)

| Metric | Value |
|---|---|
| Total shared cache lines | 0 |
| Local HITM | 0 |

Zero shared cache lines and zero HITM. Single-threaded execution; no contention possible.

---

### Flamegraph

[Builder FIX Flamegraph](../assets/flamegraph_builder_fix.svg){target="_blank"}

---

### Findings & Action Items

Builder FIX isolates the pure FIX build path and confirms the findings from Dispatch:

- IPC 3.52 confirms FIX building is compute-bound and well-optimised at the field-formatting level.
- `finalizeChecksum` at 11.00% CPU self (vs 9.45% in Dispatch) — in isolation the full checksum loop cost is proportionally larger with no routing overhead diluting it. See Dispatch finding 2.
- `transact_time` + `epoch_to_transact_time` at 9.66% CPU self (vs 7.63% in Dispatch) — `localtime_r` syscall path is the dominant timestamp cost. See Dispatch finding 3.
- L1-D miss rate of 0.03% confirms the FIX buffer and Order struct fit in L1. No data working set concern.
- iTLB at 272,287 confirms `localtime_r` kernel text as the primary iTLB stressor on the FIX path — consistent with Dispatch finding 3.

---

## Results (Builder OUCH BIST)

OUCH BIST venue message construction in isolation — `Builder_OUCH_BIST::build()` called directly per order. The test dataset includes a realistic mix of order types across 5M iterations. Used to characterise BIST-specific build performance and verify absence of false sharing or cache pressure on the OUCH path.

| Field | Value |
|---|---|
| Wall time | 0.34 s |
| User time | 0.15 s |
| Sys time | 0.19 s |
| CPUs utilized | 0.997 |

Single-threaded execution.

---

### Key Metrics

| Metric | Value | Note |
|---|---|---|
| Instructions | 4.47B | — |
| Cycles | 1.63B @ 4.78 GHz | P-core near boost |
| IPC | 2.75 | Near-zero branch and L1-D miss confirm application loop is clean |
| Branch miss rate | 0.01% | Near-zero mispredictions |
| L1-D miss rate | 0.08% | Low |
| LLC miss rate | 67.73% of 596K loads | Startup artifact — not representative |

**Branch miss rate at 0.01% and L1-D miss rate at 0.08% confirm the OUCH BIST build loop executes cleanly** against an L1-resident working set.

---

### Hotspot Analysis

#### P-Core (CPU+Branch)

| Overhead | Self | Symbol |
|---|---|---|
| 97.23% | 1.16% | `main` |
| 19.73% | 19.73% | `Builder_OUCH_BIST::build(Order const&)` |
| 15.75% | 15.75% | `Builder_OUCH_BIST::buildEnterOrder(Order const&, char*)` |

**`build()` at 19.73% self and `buildEnterOrder()` at 15.75% self are the primary application workers** — both are leaf or near-leaf functions carrying their own instruction costs directly. These are the expected hotspots for a binary-format build loop: the order entry path is the highest-frequency message type and its self-time is directly attributable to struct field writes and buffer population.

No unexpected delegation to deeper call chains is visible — self time in the build functions confirms the cost sits where expected.

---

### Cache & Memory

| Metric | Value | Assessment |
|---|---|---|
| L1-D miss rate (P-core) | 0.08% | Low — OUCH struct and output buffer L1-resident |
| LLC miss rate (P-core) | 67.73% of 596K loads | Startup artifact — not representative |
| L1-I misses (P-core) | 1,178,370 | — |
| dTLB misses (P-core) | 45,423 | — |
| iTLB misses (P-core) | 274,078 | — |

---

### False Sharing (C2C)

| Metric | Value |
|---|---|
| Total shared cache lines | 0 |
| Local HITM | 0 |

Zero shared cache lines and zero HITM. Single-threaded; no contention possible.

---

### Flamegraph

[Builder OUCH BIST Flamegraph](../assets/flamegraph_builder_ouch_bist.svg){target="_blank"}

---

### Findings & Action Items

| # | Finding | Impact | Status |
|---|---|---|---|
| 1 | `build()` 19.73% self, `buildEnterOrder()` 15.75% self — correct primary workers for binary format build | Expected hotspot distribution | ✅ No action needed |
| 2 | Branch miss 0.01%, L1-D miss 0.08% — application loop executes cleanly | Expected | ✅ No action needed |
| 3 | Zero HITM / zero shared cache lines — single-threaded, no contention | Expected | ✅ No action needed |

---

## Results (Builder OUCH NASDAQ)

OUCH NASDAQ venue message construction in isolation. Structurally equivalent to OUCH BIST but targeting the NASDAQ venue message format, which carries additional order fields. The dataset includes a broader order mix (new, replace, modify, cancel) across 5M iterations. Used to verify parity between OUCH venue implementations and confirm no venue-specific cache or branch anomalies.

| Field | Value |
|---|---|
| Wall time | 0.32 s |
| User time | 0.16 s |
| Sys time | 0.16 s |
| CPUs utilized | 0.985 |

Runtime profile matches OUCH BIST closely — confirming the two venues are structurally comparable in build cost at this dataset size.

---

### Key Metrics

| Metric | Value | vs OUCH BIST | Note |
|---|---|---|---|
| Instructions | 4.36B | Slightly fewer — NASDAQ dataset mix differs | — |
| Cycles | 1.52B @ 4.83 GHz | Near-identical clock | — |
| IPC | 2.87 | Slightly higher than BIST (2.75) | Within run-to-run variance |
| Branch miss rate | 0.01% | Identical | — |
| L1-D miss rate | 0.06% | Identical range | — |
| LLC miss rate | 63.94% of 441K loads | — | Startup artifact — not representative |

**IPC at 2.87 vs OUCH BIST's 2.75 is within run-to-run variance** — both tests are in the same performance regime. Branch miss rate at 0.01% and L1-D miss rate at 0.06% are effectively identical to BIST — parity confirmed at the hardware counter level.

---

### Hotspot Analysis

#### P-Core (CPU+Branch)

| Overhead | Self | Symbol |
|---|---|---|
| 96.42% | 1.88% | `main` |
| 17.83% | 17.83% | `Builder_OUCH_NASDAQ::build(Order&)` |
| 16.29% | 16.29% | `Builder_OUCH_NASDAQ::buildEnterOrder(Order&, char*)` |

**`build()` at 17.83% self and `buildEnterOrder()` at 16.29% self are the primary workers**, matching the BIST pattern. The self-time split between the two is nearly identical to BIST (`build` 19.73%, `buildEnterOrder` 15.75%), confirming structural parity between the two venue implementations. `buildReplaceOrder` does not appear above the 1% cutoff — it is exercised but at lower frequency, consistent with the order mix in the dataset.

---

### Cache & Memory

| Metric | Value | vs OUCH BIST |
|---|---|---|
| L1-D miss rate (P-core) | 0.06% | Identical range |
| LLC miss rate (P-core) | 63.94% of 441K loads | Startup artifact — not representative |
| L1-I misses (P-core) | 706,247 | Within sampling variance |
| dTLB misses (P-core) | 27,120 | Within sampling variance |
| iTLB misses (P-core) | 162,436 | Within sampling variance |

All metrics confirm parity with OUCH BIST within expected variance.

---

### False Sharing (C2C)

| Metric | Value |
|---|---|
| Total shared cache lines | 0 |
| Local HITM | 0 |

Zero shared cache lines and zero HITM. Single-threaded; no contention possible.

---

### Flamegraph

[Builder OUCH NASDAQ Flamegraph](../assets/flamegraph_builder_ouch_nasdaq.svg){target="_blank"}

---

### Findings & Action Items

| # | Finding | Impact | Status |
|---|---|---|---|
| 1 | IPC 2.87 — near-identical to BIST (2.75), parity confirmed | Expected | ✅ No action needed |
| 2 | `build()` 17.83% self, `buildEnterOrder()` 16.29% self — correct primary workers, structural parity with BIST confirmed | Expected hotspot distribution | ✅ No action needed |
| 3 | Branch miss 0.01%, L1-D miss 0.06% — hardware counter parity with BIST; no venue-specific anomaly | Confirms implementation symmetry | ✅ No action needed |
| 4 | `buildReplaceOrder` below top-20 threshold — exercised but lower frequency than EnterOrder, consistent with order mix | Expected | ✅ No action needed |
| 5 | Zero HITM / zero shared cache lines — single-threaded, no contention | Expected | ✅ No action needed |

---

## Raw Output

### Builder Dispatch RAW

```
task-clock:         1,087,154,509        0.999 CPUs utilized
context-switches:           7            6.439 /sec
cpu-migrations:             3            2.759 /sec
page-faults:          104,101           95.755 K/sec

P-Core (cpu_core):
  instructions:    18,134,892,123       3.45  insn/cycle
  cycles:           5,260,367,901       4.839 GHz
  branches:         2,952,785,612       2.716 G/sec
  branch-misses:           82,964       0.00%
  L1-dcache-loads:  3,412,554,682       3.139 G/sec
  L1-dcache-misses:       988,811       0.03%
  LLC-loads:              571,368     525.563 K/sec
  LLC-load-misses:        400,065      70.02%

E-Core (cpu_atom):
  instructions:     3,288,503,157       0.87  insn/cycle
  cycles:           3,792,157,963       3.488 GHz
  branches:           601,044,438     552.860 M/sec
  branch-misses:       20,954,016       3.49%
  L1-dcache-loads:    882,107,691     811.391 M/sec
  LLC-loads:            8,253,859       7.592 M/sec
  LLC-load-misses:      1,833,845      22.22%

Wall: 1.088s  User: 0.925s  Sys: 0.163s
```

```
P-Core L1-icache-load-misses:  1,191,997

Wall: 1.084s  User: 0.920s  Sys: 0.164s
```

```
P-Core dTLB-load-misses:   34,350
P-Core iTLB-load-misses:  213,482

Wall: 1.080s  User: 0.918s  Sys: 0.162s
```

CPU+Branch top application frames:
```
 99.87%  0.00%  builder_dispatch_perf  _start
 99.87%  0.00%  libc.so.6              __libc_start_main
 99.85%  0.00%  libc.so.6              0x00007fc63322a1ca
 99.73%  0.28%  builder_dispatch_perf  main
 81.08%  0.49%  builder_dispatch_perf  Builder_Dispatch::dispatch()
 69.53%  5.23%  builder_dispatch_perf  Builder_Dispatch::buildFIX(Order*)
 63.33%  0.82%  builder_dispatch_perf  Builder_Dispatch::buildFIX_app(Order*, unsigned char)
 52.87%  2.91%  builder_dispatch_perf  Buffer_FIX* Builder_FIX::build<(FIXTypes)68, Order&>(unsigned char, Order&)
 25.93%  0.95%  builder_dispatch_perf  Builder_FIX::buildHeader(char*, unsigned long, unsigned long, VenueUserInfo_FIX const&, Sequence_FIX&)
 25.01%  4.51%  builder_dispatch_perf  Builder_FIX::transact_time()
 17.83%  1.63%  builder_dispatch_perf  Builder_FIX::buildNewOrderSingle(Order const&, char*, unsigned char)
 16.26%  3.12%  builder_dispatch_perf  Builder_FIX::epoch_to_transact_time(unsigned long)
 16.16%  0.00%  libc.so.6              0x00007fc63338954a
 14.50%  4.79%  [kernel.kallsyms]      asm_exc_page_fault
 13.89%  0.00%  libc.so.6              0x00007fc6332e11b2
 10.96%  0.84%  builder_dispatch_perf  OrderManager::add_awaitingAck_order(Order&)
  9.69%  0.02%  [kernel.kallsyms]      exc_page_fault
  9.51%  0.11%  [kernel.kallsyms]      do_user_addr_fault
  9.45%  9.45%  builder_dispatch_perf  Builder_FIX::finalizeChecksum(char*, unsigned long)
  9.05%  0.11%  [kernel.kallsyms]      handle_mm_fault
```

DCache top frames:
```
 97.00%  0.00%  builder_dispatch_perf  _start
 96.97%  0.05%  builder_dispatch_perf  main
 39.72%  0.11%  builder_dispatch_perf  Builder_Dispatch::dispatch()
 34.57%  0.00%  [kernel.kallsyms]      entry_SYSCALL_64_after_hwframe
 34.57%  0.00%  [kernel.kallsyms]      do_syscall_64 / x64_sys_call
 33.36%  0.00%  libc.so.6              __munmap
 33.36%  0.00%  [kernel.kallsyms]      __x64_sys_munmap → __vm_munmap → do_vmi_munmap
 32.76%  0.00%  [kernel.kallsyms]      unmap_vmas → unmap_single_vma → unmap_page_range
 32.76%  0.00%  [kernel.kallsyms]      zap_pmd_range → zap_pte_range
```

ICache top frames:
```
 94.12%  0.05%  builder_dispatch_perf  main
 69.85%  2.09%  [kernel.kallsyms]      asm_exc_page_fault
 67.76%  0.19%  [kernel.kallsyms]      do_user_addr_fault
 55.57%  7.06%  [kernel.kallsyms]      __handle_mm_fault
 40.76%  2.48%  [kernel.kallsyms]      do_anonymous_page
 25.12%  2.36%  [kernel.kallsyms]      alloc_anon_folio
 19.90%  0.40%  builder_dispatch_perf  Builder_Dispatch::dispatch()
 16.48%  0.15%  builder_dispatch_perf  Builder_Dispatch::buildFIX(Order*)
 13.86%  0.07%  builder_dispatch_perf  Builder_Dispatch::buildFIX_app(Order*, unsigned char)
```

TLB top frames:
```
 99.53%  0.00%  builder_dispatch_perf  main
 86.16%  0.00%  libc.so.6              0x00007f50b798954a
 82.06% 82.06%  libc.so.6              0x000000000018954a
 14.80% 10.62%  [kernel.kallsyms]      asm_exc_page_fault
```

C2C:
```
Total Shared Cache Lines:          0
Load HITs on shared lines:         0
Load Local HITM:                   0
Locked Access on shared lines:     0
Blocked Access on shared lines:    0
```

---

### Builder FIX RAW

```
task-clock:           858,825,911        0.886 CPUs utilized
context-switches:             5          5.822 /sec
cpu-migrations:               1          1.164 /sec
page-faults:             79,392         92.442 K/sec

P-Core (cpu_core):
  instructions:    14,723,219,732       3.52  insn/cycle
  cycles:           4,183,070,695       4.871 GHz
  branches:         2,362,661,089       2.751 G/sec
  branch-misses:           71,657       0.00%
  L1-dcache-loads:  2,466,990,919       2.873 G/sec
  L1-dcache-misses:       689,054       0.03%
  LLC-loads:              505,113     588.144 K/sec
  LLC-load-misses:        321,956      63.74%

Wall: 0.969s  User: 0.709s  Sys: 0.151s
```

```
P-Core L1-icache-load-misses:   827,117

Wall: 0.858s  User: 0.716s  Sys: 0.142s
```

```
P-Core dTLB-load-misses:   27,127
P-Core iTLB-load-misses:  272,287

Wall: 0.860s  User: 0.713s  Sys: 0.147s
```

CPU+Branch top frames:
```
100.00%  0.00%  builder_fix_perf  _start / __libc_start_main
 88.21%  3.71%  builder_fix_perf  main
 33.16%  1.31%  builder_fix_perf  Builder_FIX::buildHeader(...)
 31.85%  6.05%  builder_fix_perf  Builder_FIX::transact_time()
 21.15%  2.62%  builder_fix_perf  Builder_FIX::buildNewOrderSingle(...)
 18.42%  3.61%  builder_fix_perf  Builder_FIX::epoch_to_transact_time(unsigned long)
 17.57%  5.27%  [kernel.kallsyms]  asm_exc_page_fault
 11.00% 11.00%  builder_fix_perf  Builder_FIX::finalizeChecksum(char*, unsigned long)
```

DCache top frames:
```
 96.47%  0.17%  builder_fix_perf   main
 49.41%  0.00%  libc.so.6          0x00007f9fab98954a
 48.30%  1.48%  [kernel.kallsyms]  asm_exc_page_fault
 45.82%  0.00%  [kernel.kallsyms]  entry_SYSCALL_64 / do_syscall_64
 45.76%  0.00%  libc.so.6          __munmap → __x64_sys_munmap → __vm_munmap
 45.76%  0.00%  [kernel.kallsyms]  do_vmi_munmap → do_vmi_align_munmap
 45.18%  0.00%  [kernel.kallsyms]  unmap_vmas → unmap_single_vma → unmap_page_range
```

C2C:
```
Total Shared Cache Lines:          0
Load Local HITM:                   0
```

---

### Builder OUCH BIST RAW

```
task-clock:           340,456,394        0.997 CPUs utilized
context-switches:             2          5.874 /sec
cpu-migrations:               2          5.874 /sec
page-faults:            137,132        402.789 K/sec

P-Core (cpu_core):
  instructions:     4,473,921,468       2.75  insn/cycle
  cycles:           1,627,075,492       4.779 GHz
  branches:           654,698,695       1.923 G/sec
  branch-misses:           77,982       0.01%
  L1-dcache-loads:  1,109,823,079       3.260 G/sec
  L1-dcache-misses:       913,192       0.08%
  LLC-loads:              595,770       1.750 M/sec
  LLC-load-misses:        403,507      67.73%

E-Core (cpu_atom):
  instructions:     1,283,186,600       1.06  insn/cycle
  cycles:           1,207,768,515       3.547 GHz
  branch-misses:        7,166,154       3.14%

Wall: 0.341s  User: 0.153s  Sys: 0.188s
```

```
P-Core L1-icache-load-misses:  1,178,370

Wall: 0.350s
```

```
P-Core dTLB-load-misses:   45,423
P-Core iTLB-load-misses:  274,078

Wall: 0.338s
```

CPU+Branch top frames:
```
 97.23%  1.16%  builder_ouch_bist_perf  main
 48.28% 16.16%  [kernel.kallsyms]       asm_exc_page_fault
 29.31%  1.23%  [kernel.kallsyms]       __handle_mm_fault
 27.20%  1.24%  [kernel.kallsyms]       do_anonymous_page
 19.73% 19.73%  builder_ouch_bist_perf  Builder_OUCH_BIST::build(Order const&)
 15.75% 15.75%  builder_ouch_bist_perf  Builder_OUCH_BIST::buildEnterOrder(Order const&, char*)
```

C2C:
```
Total Shared Cache Lines:          0
Load Local HITM:                   0
```

---

### Builder OUCH NASDAQ RAW

```
task-clock:           314,477,938        0.985 CPUs utilized
context-switches:             5         15.899 /sec
cpu-migrations:               1          3.180 /sec
page-faults:             81,434        258.950 K/sec

P-Core (cpu_core):
  instructions:     4,357,458,896       2.87  insn/cycle
  cycles:           1,517,574,289       4.826 GHz
  branches:           526,404,576       1.674 G/sec
  branch-misses:           51,161       0.01%
  L1-dcache-loads:  1,124,194,425       3.575 G/sec
  L1-dcache-misses:       621,884       0.06%
  LLC-loads:              441,352       1.403 M/sec
  LLC-load-misses:        282,210      63.94%

E-Core (cpu_atom):
  instructions:     1,084,697,687       1.04  insn/cycle
  cycles:           1,039,947,457       3.307 GHz
  branch-misses:        5,887,258       3.06%

Wall: 0.319s  User: 0.158s  Sys: 0.157s
```

```
P-Core L1-icache-load-misses:   706,247

Wall: 0.308s
```

```
P-Core dTLB-load-misses:   27,120
P-Core iTLB-load-misses:  162,436

Wall: 0.308s
```

CPU+Branch top frames:
```
 96.42%  1.88%  builder_ouch_nasdaq_perf  main
 44.41% 14.13%  [kernel.kallsyms]         asm_exc_page_fault
 26.99%  0.70%  [kernel.kallsyms]         __handle_mm_fault
 25.40%  1.05%  [kernel.kallsyms]         do_anonymous_page
 17.83% 17.83%  builder_ouch_nasdaq_perf  Builder_OUCH_NASDAQ::build(Order&)
 16.29% 16.29%  builder_ouch_nasdaq_perf  Builder_OUCH_NASDAQ::buildEnterOrder(Order&, char*)
```

C2C:
```
Total Shared Cache Lines:          0
Load Local HITM:                   0
```
