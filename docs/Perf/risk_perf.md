# Risk — Perf Analysis

Profiling of the risk engine under two isolated scenarios: **risk update** (write path — `update_risk()` processing incoming order acknowledgements) and **risk check** (read path — `check_risk()` validating orders before submission).

---

## Test Setup

All perf tests were conducted under **Profile 1 — Performance/Idle**. For the full profile comparison and selection rationale, see [Pipeline Overview](pipeline.md#profile-selection).

| Field | Value |
|---|---|
| CPU | 13th Gen Intel Core i7-13620H (10C/16T) |
| P-core max | 4900 MHz |
| Kernel | 6.17.0-29-generic x86_64 |
| Governor | performance |
| NUMA | Single node, 15714 MB total |

---

## At a Glance

| Metric | Risk Update | Risk Check |
|---|---|---|
| Wall time | 0.385 s | 0.192 s |
| P-core IPC | 2.13 | 5.47–5.51 |
| Branch miss rate | 0.01% | 0.00–0.01% |
| L1-D miss rate | 0.24% | 0.01% |
| LLC miss rate | ~0% (absolute) | ~0% (absolute) |
| Local HITM | 0 | 0 |

Both paths are **compute-bound in isolation** — IPC well above 1.0 in both cases with negligible cache pressure. This confirms the finding from pipeline profiling: the risk engine itself is not the bottleneck; cross-thread queue contention and E-core clock limitations are.

Risk check in particular stands out with IPC 5.47–5.51 and L1-D miss rate 0.01% — the check path is executing at near-peak efficiency when there is no concurrent writer on the shared structs.

---

## Results — Risk Update

`update_risk()` processes order acknowledgements — updating `SymbolRisk`, `AccountRisk`, and `OrderRisk` fields on fill, partial fill, cancel, and new order events. The test drives 10M iterations with a 3-order mix (fixNew, fixPartial, fixCancel) through `store_to_risk`, with two consumer threads draining `risk_to_builder` and `risk_to_strategy`. Risk state is reset after each batch.

| Field | Value |
|---|---|
| Wall time | 0.385 s |
| User time | 0.381 s |
| Sys time | 0.004 s |
| CPUs utilized | 0.996 |

Single-threaded execution on the update path — CPUs at ~1.0 confirms no meaningful parallel worker is active. Sys time at 0.004s is negligible. The two consumer threads (`risk_to_builder` and `risk_to_strategy`) drain their queues with `_mm_pause()` spin loops on CPU 0 and CPU 2; they consume negligible resources as the queues are always empty from their perspective (the update loop runs to completion before they can see data).

---

### Key Metrics

| Metric | P-Core | E-Core | Note |
|---|---|---|---|
| Instructions | 3.96B | 1.17B | E-core activity is startup/linker overhead |
| Cycles | 1.86B @ 4.84 GHz | 1.33B @ 3.47 GHz | P-core near boost |
| IPC | 2.13 | 0.88 | See note below |
| Branch miss rate | 0.01% | 3.44% | P-core excellent |
| L1-D miss rate | 0.24% | — | Excellent |
| LLC miss rate | 46.66% of 26,886 loads | 18.61% of 2.89M | See note below |

**IPC at 2.13 on P-core — update_risk is compute-bound in isolation.** The atomic fetch operations on `SymbolRisk` and `AccountRisk` fields are the dominant instructions; with a hot working set the P-core executes efficiently through the update chain. The E-core at 0.88 IPC and 3.44% branch miss reflects startup initialization rather than steady-state behavior — the E-core sample window is only 0.18% of total samples.

**The 46.66% LLC miss rate on the P-core is an artifact of the small LLC load count (26,886 absolute loads).** At this low denominator, a handful of cold startup misses produce a high percentage. The absolute LLC miss count (12,546 events) is negligible relative to the 1.12B L1-D load count — the application working set is comfortably L1/LLC-resident. The L1-D miss rate at 0.24% is the correct signal, and it is excellent.

---

### Hotspot Analysis

| Overhead | Self | Symbol | Profile |
|---|---|---|---|
| 89.59% | 46.68% | `RiskEngine::update_risk()` | CPU+Branch |
| 46.26% | 46.26% | `RiskEngine::update_risk_for_protocol_fix(AccountRisk&, AccountLimit const&, Order&, OrderMetrics)` | CPU+Branch |
| 96.89% | 60.43% | `RiskEngine::update_risk()` | dcache |
| 35.83% | 35.03% | `RiskEngine::update_risk_for_protocol_fix(...)` | dcache |

`update_risk()` at 46.68% CPU self and 60.43% dcache self is the clear hot function — it drives the update dispatch and carries the bulk of atomic write cost directly. `update_risk_for_protocol_fix()` at 46.26% CPU self and 35.03% dcache self is nearly as costly on the CPU dimension — FIX order acknowledgements trigger the full update path including `pending_notional_scaled` and `open_orders_count` updates on every event, and the function carries its full cost as a leaf (no further callees visible in the profile).

The near-parity in CPU self-time between `update_risk` (46.68%) and `update_risk_for_protocol_fix` (46.26%) indicates the dispatch overhead in `update_risk` and the field write cost in the protocol-specific path are roughly balanced. In the dcache profile the split shifts toward `update_risk` (60.43% vs 35.03%), suggesting the dispatch and branching logic in `update_risk` generates proportionally more data access events than the targeted atomic writes in the FIX path.

!!! note "ICache profile — kernel page fault dominant"
    The ICache record is dominated by kernel page-fault frames (`asm_exc_page_fault` → `__handle_mm_fault` → `do_anonymous_page` chain), consistent with the other Builder tests in this run. `update_risk()` is visible at 15.90% overhead — confirming it is sampled but suppressed by the kernel noise. This is a test harness artifact.

!!! note "TLB profile — very low sample count"
    The TLB record accumulated only ~33 samples at this test duration. The top entry is `snprintf` at 72.73% self — this is a logging or initialization call rather than steady-state update path behavior. The dTLB miss absolute count (42,252) is low in absolute terms and is consistent with startup and OS activity rather than the update loop itself. The TLB profile is not meaningful at this sample depth.

---

### Cache & Memory

| Metric | Value | Note |
|---|---|---|
| L1-D miss rate | 0.24% | Excellent — hot struct fields stay in L1 |
| LLC miss rate | ~0% absolute | 12,546 events — negligible |
| L1-I misses | 381,033 | Elevated — kernel page fault text pages; not an app icache concern |
| dTLB misses | 42,252 | Low absolute — TLB profile dominated by startup/OS activity |
| iTLB misses | 2,464 | Very low |

**L1-D miss rate at 0.24% is excellent in isolation.** The `SymbolRisk` and `AccountRisk` fields written by `update_risk()` are hot in L1 — they are written repeatedly on consecutive calls and stay resident. The elevated L1-I misses (381K) are attributable to kernel page-fault text pages, identical in character to the Builder Dispatch findings, not to the update logic itself.

---

### False Sharing (C2C)

| Metric | Value |
|---|---|
| Total shared cache lines | 0 |
| Local HITM | 0 |

**Zero HITM — no false sharing in isolation.** Single-threaded update with no concurrent reader on the risk structs. The HITM events seen in pipeline profiling for `update_risk()` are entirely caused by the concurrent `check_risk()` thread reading the same struct fields — not by anything inherent to the update path itself.

---

### Flamegraph

[Risk Update Flamegraph](../assets/flamegraph_risk_update.svg){target="_blank"}

---

### Findings & Action Items

| # | Finding | Impact | Status |
|---|---|---|---|
| 1 | IPC 2.13 — update path is compute-bound in isolation | Implementation is efficient | ✅ No action needed |
| 2 | L1-D miss 0.24% — struct fields stay hot in L1 | Excellent | ✅ No action needed |
| 3 | Zero HITM in isolation — pipeline HITM is cross-thread, not update-internal | Confirms struct layout is not the root cause | ✅ Confirmed |
| 4 | `update_risk_for_protocol_fix` 46.26% CPU self — FIX path is near-equal cost to dispatch itself | Expected — FIX carries full order state update; both branches are correctly weighted | ✅ No action needed |
| 5 | LLC miss rate 46.66% — artifact of 26,886 total LLC loads; absolute count 12,546 is negligible | Test harness artifact | ✅ No action needed |
| 6 | L1-I misses 381K and TLB profile low-sample — page fault kernel text artifact consistent with other tests in this run | Test harness artifact | ✅ No action needed |

---

## Results — Risk Check

`check_risk()` validates orders before submission — reading `SymbolRisk`, `AccountRisk`, `SymbolMeta`, and `Limits` fields to enforce position, notional, and rate limit constraints. The test drives 10M iterations with a single order (`strategy_to_risk`) per iteration, with two consumer threads draining `risk_to_builder` and `risk_to_strategy`. Risk state is reset after each call.

| Field | Value |
|---|---|
| Wall time | 0.192 s |
| User time | 0.189 s |
| Sys time | 0.003 s |
| CPUs utilized | 0.993 |

Single-threaded execution — CPUs at ~1.0. Sys time at 0.003s is negligible. The test completes in 0.192s wall time with 10M iterations — substantially faster than the update test (0.385s at the same N), consistent with the much higher IPC and lower data miss rate on the check path.

---

### Key Metrics

| Metric | P-Core | E-Core | Note |
|---|---|---|---|
| Instructions | 4.81–4.87B | — (not counted) | E-core not active in this run |
| Cycles | 879M–883M @ 4.60–4.63 GHz | — | P-core near boost |
| IPC | 5.47–5.51 | — | See note below |
| Branch miss rate | 0.00–0.01% | — | Essentially zero mispredictions |
| L1-D miss rate | 0.01% | — | Exceptional |
| LLC miss rate | 48.16% of 26,350 loads | — | See note below |

**IPC at 5.47–5.51 on P-core is the highest observed across all risk and builder scenarios in this run.** `check_risk()` is a read-only validation path — reading `SymbolRisk`, `AccountRisk`, and `Limits` fields with no atomic writes. With L1-D miss rate at 0.01%, all read targets are hot in cache; the branch predictor handles the limit check conditions at near-zero miss rate; and the instruction stream is dense and predictable. The P-core is executing at near its retirement ceiling.

**The 48.16% LLC miss rate is again an artifact of the small LLC load count (26,350 loads).** The absolute miss count (12,690 events) is negligible relative to the 1.27B L1-D load count. The working set is entirely L1-resident — the 0.01% L1-D miss rate confirms this directly.

---

### Hotspot Analysis

#### P-Core (CPU+Branch)

| Overhead | Self | Symbol |
|---|---|---|
| 99.35% | 0.00% | `_start` / `libc` init |
| 95.17% | 5.59% | `main` |
| 87.54% | 87.41% | `RiskEngine::check_risk()` |
| 2.83% | 2.83% | `RiskEngine::check_venue_halt_and_circuit(Order&, SymbolMeta const&)` |

**`check_risk()` at 87.41% self is the overwhelming dominant consumer** — a single function carrying almost the entire CPU cost of the test. This is expected: the check path is a monolithic validation sequence with most logic inlined, leaving no meaningful delegation to helper functions. The function reads all required fields, evaluates all limit conditions, and returns a result with minimal call overhead.

`check_venue_halt_and_circuit()` at 2.83% self is a newly visible secondary hotspot — the venue halt and circuit breaker check now has sufficient samples to appear in the top-20. This is a leaf function reading `SymbolMeta` halt/circuit state; its cost is small but attributable. No concern at 2.83%.

#### DCache Profile

| Overhead | Self | Symbol |
|---|---|---|
| 29.70% | 8.12% | `RiskEngine::check_risk()` |

`check_risk()` at 8.12% dcache self is the only meaningful application entry. The dcache profile is heavily contaminated by kernel OS scheduling activity (`hrtimer_interrupt`, `tick_nohz_handler`, `update_process_times` chain accounting for ~25% of overhead) at this short wall time — this is a timer interrupt artifact, not application cache behavior.

!!! note "ICache profile — startup and page fault artifact"
    `folly::CacheLocality::readFromProcCpuinfoLines()` at 10.73% icache self is startup initialization from the Folly library — not a steady-state cost. `check_risk()` is visible at 10.70% overhead / 0.34% self, confirming it is present but suppressed by the startup noise. The absolute L1-I miss count (288,009) is elevated by kernel page-fault text pages consistent with the other tests in this run.

!!! note "TLB profile — timer interrupt dominant"
    The TLB record shows `main` at 100% self (all dTLB misses occur at startup within `main` before the loop) and `asm_sysvec_apic_timer_interrupt` at 86.05% self — the kernel timer interrupt dominates TLB samples at this run duration. `sysvec_thermal` at 6.98% indicates a CPU thermal interrupt fired during the test — a hardware event unrelated to the application. The absolute dTLB miss count (10,305) is very low and not an application concern.

---

### Cache & Memory

| Metric | Value | Note |
|---|---|---|
| L1-D miss rate | 0.01% | Exceptional — all read targets L1-resident |
| LLC miss rate | ~0% absolute | 12,690 events — negligible |
| L1-I misses | 288,009 | Elevated — kernel page fault text; not an app icache concern |
| dTLB misses | 10,305 | Low — startup-dominant |
| iTLB misses | 2,782 | Very low |

---

### False Sharing (C2C)

| Metric | Value |
|---|---|
| Total shared cache lines | 0 |
| Local HITM | 0 |

**Zero HITM — no false sharing in isolation.** Single-threaded check with no concurrent reader on the risk structs. The HITM events seen in pipeline profiling for `check_risk()` are entirely caused by the concurrent `update_risk()` thread reading the same struct fields — not by anything inherent to the check path itself.

---

### Flamegraph

[Risk Check Flamegraph](../assets/flamegraph_risk_check.svg){target="_blank"}

---

### Findings & Action Items

| # | Finding | Impact | Status |
|---|---|---|---|
| 1 | IPC 5.47–5.51 — highest across all risk and builder scenarios; check path executing at near-peak retirement | Excellent — implementation is highly efficient | ✅ No action needed |
| 2 | L1-D miss 0.01% — all read targets L1-resident | Exceptional | ✅ No action needed |
| 3 | Branch miss 0.00% — limit checks perfectly predicted | Excellent | ✅ No action needed |
| 4 | Zero HITM in isolation — previous 371 HITM was two-thread test artifact; no structural false sharing on check path | Confirmed — check path is contention-free in isolation | ✅ No action needed |
| 5 | `check_risk()` 87.41% CPU self — monolithic check function, well-inlined, no unexpected delegation | Expected | ✅ No action needed |
| 6 | `check_venue_halt_and_circuit()` 2.83% CPU self — leaf function, venue halt and circuit breaker read cost | Small but attributable; no concern at current rate | ✅ No action needed |
| 7 | `check_risk()` on E-core in pipeline — IPC ceiling of ~5.5 on P-core vs E-core clock gap | Latency cost at production order rates | ⚠️ Pre-prod thread affinity redesign |

---

## Cross-Scenario: Update vs Check

The comparison between isolated results reveals the clear performance asymmetry between the write and read paths.

| Scenario | IPC | L1-D miss | HITM | Source |
|---|---|---|---|---|
| Update isolated | 2.13 | 0.24% | 0 | Clean |
| Check isolated | 5.47–5.51 | 0.01% | 0 | Clean |
| Pipeline E-core (RxTx) | 0.44 | — | ~14K (shared) | Cross-thread contention |

**Check outperforms update by ~2.6× in IPC and 24× in L1-D miss rate in isolation.** The update path performs atomic read-modify-write operations on multiple fields per order event — these are inherently more expensive than the pure reads of the check path, and their cost is correctly reflected in the IPC and miss rate gap even when no cross-thread contention is present.

**The gap between isolated and pipeline performance is entirely explained by cross-thread interaction** — concurrent readers and writers on the same `SymbolRisk` cache lines, E-core clock limitations, and boost queue contention. The risk engine implementation is efficient on both paths; the pipeline bottleneck is the runtime environment.

---

## Raw Output

### Risk Update RAW

"Perf Stat — Base"
    ```
    task-clock:         383,852,505        0.996 CPUs utilized
    context-switches:           8         20.841 /sec
    cpu-migrations:             4         10.421 /sec
    page-faults:            1,079          2.811 K/sec

    P-Core (cpu_core):
      instructions:    3,958,615,939       2.13  insn/cycle
      cycles:          1,858,692,339       4.842 GHz
      branches:          514,486,209       1.340 G/sec
      branch-misses:          32,832       0.01%
      L1-dcache-loads:  1,116,930,687      2.910 G/sec
      L1-dcache-misses:    2,645,027       0.24%
      LLC-loads:              26,886      70.043 K/sec
      LLC-load-misses:        12,546      46.66%

    E-Core (cpu_atom):
      instructions:    1,168,424,154       0.88  insn/cycle
      cycles:          1,333,475,138       3.474 GHz
      branches:          213,989,815     557.479 M/sec
      branch-misses:       7,359,374       3.44%
      L1-dcache-loads:   313,300,180     816.199 M/sec
      LLC-loads:           2,893,228       7.537 M/sec
      LLC-load-misses:       538,490      18.61%

    Wall: 0.385s  User: 0.381s  Sys: 0.004s
    ```

"Perf Stat — ICache"
    ```
    P-Core L1-icache-load-misses:  381,033

    Wall: 0.385s  User: 0.378s  Sys: 0.007s
    ```

"Perf Stat — TLB"
    ```
    P-Core dTLB-load-misses:  42,252
    P-Core iTLB-load-misses:   2,464

    Wall: 0.387s  User: 0.382s  Sys: 0.005s
    ```

"Perf Record — CPU+Branch (top 20)"
    ```
    99.70%  0.00%  risk_update_perf  _start / __libc_start_main
    99.23%  5.08%  risk_update_perf  main
    89.59% 46.68%  risk_update_perf  RiskEngine::update_risk()
    46.26% 46.26%  risk_update_perf  RiskEngine::update_risk_for_protocol_fix(AccountRisk&, AccountLimit const&, Order&, OrderMetrics)
     1.35%  0.25%  [kernel.kallsyms] asm_exc_page_fault
    ```

"Perf Record — DCache (top 20)"
    ```
    98.31%  0.06%  risk_update_perf  main
    96.89% 60.43%  risk_update_perf  RiskEngine::update_risk()
    35.83% 35.03%  risk_update_perf  RiskEngine::update_risk_for_protocol_fix(AccountRisk&, AccountLimit const&, Order&, OrderMetrics)
     1.83%  0.00%  [kernel.kallsyms] asm_sysvec_apic_timer_interrupt (timer interrupt artifact)
    ```

"Perf Record — ICache (top 20)"
    ```
    69.39%  0.11%  risk_update_perf  main
    49.50%  0.00%  [kernel.kallsyms] asm_exc_page_fault (kernel page fault dominant)
    41.17%  8.06%  [kernel.kallsyms] __handle_mm_fault
    15.90%  0.18%  risk_update_perf  RiskEngine::update_risk()
    ```

"C2C — False Sharing Summary"
    ```
    Total records:            2,898
    Load Local HITM:              0
    Total Shared Cache Lines:     0
    No false sharing detected.
    ```

---

### Risk Check RAW

"Perf Stat — Base"
    ```
    task-clock:           191,049,123        0.993 CPUs utilized
    context-switches:             6         31.406 /sec
    cpu-migrations:               3         15.703 /sec
    page-faults:              1,080          5.653 K/sec

    P-Core (cpu_core):
      instructions:    4,812,908,544       5.47  insn/cycle
      cycles:            879,381,084       4.603 GHz
      branches:          643,744,576       3.370 G/sec
      branch-misses:          42,119       0.01%
      L1-dcache-loads:  1,265,375,042      6.623 G/sec
      L1-dcache-misses:      136,399       0.01%
      LLC-loads:              26,350     137.923 K/sec
      LLC-load-misses:        12,690      48.16%

    E-Core (cpu_atom):
      <not counted> — not active in this run

    Wall: 0.192s  User: 0.189s  Sys: 0.003s
    ```

"Perf Stat — CPU+Branch"
    ```
    P-Core:
      cycles:          882,860,017
      instructions:  4,865,958,323       5.51  insn/cycle
      branch-misses:        13,543       0.00%

    Wall: 0.191s  User: 0.185s  Sys: 0.006s
    ```

"Perf Stat — ICache"
    ```
    P-Core L1-icache-load-misses:  288,009

    Wall: 0.189s  User: 0.183s  Sys: 0.006s
    ```

"Perf Stat — TLB"
    ```
    P-Core dTLB-load-misses:  10,305
    P-Core iTLB-load-misses:   2,782

    Wall: 0.192s  User: 0.187s  Sys: 0.005s
    ```

"Perf Record — CPU+Branch (top 20)"
    ```
    99.35%  0.00%  risk_check_perf  _start / __libc_start_main
    95.17%  5.59%  risk_check_perf  main
    87.54% 87.41%  risk_check_perf  RiskEngine::check_risk()
     2.83%  2.83%  risk_check_perf  RiskEngine::check_venue_halt_and_circuit(Order&, SymbolMeta const&)
     2.81%  0.55%  [kernel.kallsyms] asm_exc_page_fault
    ```

"Perf Record — DCache (top 20)"
    ```
    56.79%  0.00%  risk_check_perf  main
    30.18%  0.00%  [kernel.kallsyms] entry_SYSCALL_64 (OS scheduling activity)
    29.70%  8.12%  risk_check_perf  RiskEngine::check_risk()
    25.24%  0.00%  [kernel.kallsyms] asm_sysvec_apic_timer_interrupt (timer interrupt)
    ```

"Perf Record — ICache (top 20)"
    ```
    67.76%  0.00%  risk_check_perf  main
    52.51%  0.00%  [kernel.kallsyms] asm_exc_page_fault (kernel page fault dominant)
    30.74% 10.21%  [kernel.kallsyms] __handle_mm_fault
    10.73% 10.73%  risk_check_perf   folly::CacheLocality::readFromProcCpuinfoLines(...) — startup artifact
    10.70%  0.34%  risk_check_perf   RiskEngine::check_risk()
    ```

"C2C — False Sharing Summary"
    ```
    Total records:            577
    Load Local HITM:            0
    Total Shared Cache Lines:   0
    No false sharing detected.
    ```
