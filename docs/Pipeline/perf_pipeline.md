# Pipeline — Perf Analysis

End-to-end profiling of the full trading pipeline under sustained load. Three test scenarios are covered: **RxTx** (combined receive and transmit threads running concurrently), **Rx** (receive path only), and **Tx** (transmit path only).

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

## Results (RxTx)

Both the network receive and transmit threads run simultaneously. This is the most representative scenario — it reflects real deployment load and exposes cross-thread contention that single-path tests cannot reveal.

| Field | Value |
|---|---|
| Wall time | 7.05 s |
| User time | 48.82 s |
| Sys time | 6.68 s |
| CPUs utilized | 7.877 |

Wall time is the real elapsed duration of the test. User time is the sum of CPU time spent in userspace across all threads — with ~7 threads running, 48s user over 7s wall is expected."Sys time (6.68s, ~12% of user) reflects kernel time spent on continuous socket maintenance and event loop (epoll) machinery on network layer; this is on the higher side and shows kernel bypass importance."

---

### Key Metrics

| Metric | P-Core | E-Core | Note |
|---|---|---|---|
| Instructions | 63.4B | 16.4B | — |
| Cycles | 137.6B @ 2.48 GHz | 37.4B @ 0.67 GHz | E-cores running well below max clock |
| IPC | 0.46 | 0.44 | See note below |
| Branch miss rate | 0.18% | 0.15% | Excellent — predictable control flow |
| L1-D miss rate | 0.97% | — | Healthy — hot data mostly in L1 |
| LLC miss rate | 0.22% | 0.05% | Very low — working set fits in LLC |

**IPC at 0.46 reflects the pipeline's architecture, not a problem.** A compute-bound workload runs 2.0–4.0 IPC on modern cores. This pipeline continuously reads and writes structs over SPSC queues with atomic operations — it is memory-bound by design. An IPC of 0.4–0.6 is the expected characteristic of this access pattern. No action needed until replacing boost SPSC queue with custom queue.

---

### Hotspot Analysis

Overhead is the inclusive cost of a function and all its callees — high overhead on a dispatcher is expected. Self is the cost of the function's own instructions only — this is the actionable signal. High self time means the function itself is consuming CPU, not delegating to children.

#### P-Core

| Overhead | Self | Symbol |
|---|---|---|
| 82.81% | 0.00% | `libc` / `libstdc++` init (startup) |
| 17.19% | 0.00% | `main` |
| 16.47% | 0.00% | `std::thread` → `NetworkFlow::start()` |
| 16.44% | 9.57% | `main::{lambda(int,int)#1}::operator()` |
| 16.39% | 0.40% | `NetworkIO::recv_send()` |
| 16.39% | 13.81% | `OrderFlow::start()` lambda |
| 16.05% | 14.35% | `RiskFlow::start()` lambda |
| 15.66% | 15.65% | `BuilderFlow::start()` lambda |
| 15.39% | 14.62% | `ParserFlow::start()` lambda |
| 13.51% | 0.24% | `entry_SYSCALL_64` [kernel] |
| 13.25% | 0.43% | `do_syscall_64` [kernel] |
| 11.83% | 0.16% | `x64_sys_call` [kernel] |
| 9.44% | 1.33% | `epoll_wait` |
| 4.66% | 0.00% | `__x64_sys_sendto` [kernel] |
| 4.64% | 0.04% | `__sys_sendto` [kernel] |

P-core hosts all flow threads — `OrderFlow`, `RiskFlow`, `BuilderFlow`, `ParserFlow` all appear at 13–16% self time, load is evenly distributed. `NetworkIO::recv_send()` acts as a dispatcher at 0.40% self. The syscall chain (`entry_SYSCALL_64` → `sendto`) descends from 13.5% to 4.6% — this is the source of the icache miss pressure observed in the icache profile. `epoll_wait` at 1.33% self is expected.

#### E-Core

| Overhead | Self | Symbol |
|---|---|---|
| 47.26% | 38.96% | `RiskFlow::start()` lambda |
| 46.91% | 41.60% | `OrderFlow::start()` lambda |
| 10.92% | 10.88% | `RiskEngine::check_risk()` |
| 8.31% | 8.28% | `ClickhouseWriter::store()` |
| 0.04% | 0.04% | `RiskEngine::check_venue_halt_and_circuit()` |

`RiskFlow` and `OrderFlow` spin loops account for nearly all E-core cycles — their high self time confirms they are the actual workers. `RiskEngine::check_risk()`.

!!! warning "Optimization Candidates"
    **`RiskEngine::check_risk()` — 10.88% E-core self time.** 
    `check_risk` was intentionally placed on an E-core given its read-heavy profile. However at 10.88% self time and given the 36% clock frequency gap between E-core (3600 MHz max) and P-core (4900 MHz max), this placement has a measurable latency cost on the order path. Pre-production thread affinity redesign and hardware limit analysis should revisit this assignment.


---

### Cache & Memory

| Metric | Value | Assessment |
|---|---|---|
| L1-D miss rate (P-core) | 0.97% | Healthy |
| LLC miss rate (P-core) | 0.22% | Very low — working set fits in LLC |
| LLC miss rate (E-core) | 0.05% | Excellent |
| L1-I misses (P-core) | 2.05B events | Elevated — syscall driven, see below |
| dTLB misses (P-core) | 249K | Low |
| iTLB misses (P-core) | 700K | Moderate — consistent with syscall pattern |

**L1-I misses are elevated at 2.05B** — roughly 12× the L1-D miss count. The icache record confirms this is syscall-driven, not application code size. The top samples from the icache profile are shown in the raw output; `__x64_sys_sendto` and `__sys_sendto` appear at ~50%, directly attributing the pressure to the Tx `sendto` path. Each syscall entry drags kernel text into the icache and pollutes the iTLB, which explains the 700K iTLB miss count.
    
---

### False Sharing (C2C) { #rxtx-c2c-back }

| Metric | Value |
|---|---|
| Total shared cache lines | 3,724 |
| Load HITs on shared lines | 39,792 |
| Local HITM | 14,392 |
| Locked accesses on shared lines | 2,637 |
| Blocked accesses on shared lines | 3,023 |

**14,392 Local HITM events are present, but detailed cache line analysis shows no application struct false sharing.** The majority of HITM originates from `boost::lockfree` queue internal head/tail state. Remaining events are test-only artifacts absent in production. See [C2C details](#rxtx-c2c).

!!! warning "Optimization Candidate"
    The dominant false sharing source is `boost::lockfree` queue state shared between `RiskFlow` and `OrderFlow` threads. A custom SPSC queue with explicit head/tail cache line separation would eliminate this. Pre-production redesign should evaluate replacing `boost::lockfree` with a purpose-built queue.

---

### Flamegraph

[RxTx Flamegraph](../assets/flamegraph_engine_RxTx.svg){target="_blank"}

---

### Findings & Action Items

| # | Finding | Impact | Status |
|---|---|---|---|
| 1 | IPC 0.46 — memory-bound by design, SPSC queue struct access pattern | Expected pipeline characteristic | ✅ No action needed |
| 2 | `RiskEngine::check_risk()` — 10.88% E-core self, 36% clock gap vs P-core | Direct latency cost on order path | ⚠️ Identified, pre-prod redesign |
| 3 | L1-I miss 2.05B + sys time ~12% — sendto syscall overhead on Tx path | Icache/iTLB pressure + kernel time | ⚠️ Kernel bypass (DPDK/io_uring) candidate — pre-prod hardware redesign |
| 5 | Branch miss rate 0.15–0.18% | Negligible | ✅ No action needed |
| 6 | LLC miss rate 0.05–0.22% — working set fits in LLC | Healthy | ✅ No action needed |

---

## Results (Rx)

Receive path only — no transmit thread running. Used to isolate the Rx contribution to RxTx numbers and verify path-specific behavior without cross-path contention.

| Field | Value | vs RxTx |
|---|---|---|
| Wall time | 6.60 s | ~6% shorter — no Tx thread competing |
| User time | 45.84 s | Expected: ~1 thread fewer |
| Sys time | 6.16 s | ~8% lower — sampling variance |
| CPUs utilized | 7.881 | Near-identical to RxTx |

**The Rx-only sys time (6.16s)** establishes a nearly identical baseline to the RxTx-heavy scenarios, confirming that the kernel footprint is dominated by the active socket connections maintained with the simulator. Since the overhead persists even without the internal engine's Tx path, it proves that the kernel-space context switching and socket state management—driven by the fake server's traffic—accounts for the bulk of the sys time, rather than the Tx processing logic itself."

---

### Key Metrics

| Metric | P-Core | E-Core | vs RxTx |
|---|---|---|---|
| IPC | 0.45–0.46 | 0.44 | Identical — memory-bound characteristic unchanged |
| Branch miss rate | 0.19% | 0.09–0.10% | Marginal, within noise |
| L1-D miss rate | 0.96% | — | Matches RxTx — Rx owns most of the data working set |
| LLC miss rate | 0.11% | 0.08% | **Lower than RxTx (0.22%)** — see note |
| L1-I misses (P-core) | 1.91B | — | Lower than RxTx (2.05B) — syscall icache pressure reduced |

**LLC miss rate drops to 0.11% in Rx-only.** In RxTx the P-core LLC miss rate is 0.22% — the Tx path is responsible for the delta. Tx brings additional working set (packet build, send buffers) that competes for LLC capacity. In isolation, the Rx working set fits comfortably.

---

### Hotspot Analysis

#### P-Core

| Overhead | Self | Symbol |
|---|---|---|
| 82.87% | 0.00% | `libc` / `libstdc++` init (startup) |
| 17.12% | 0.00% | `main` |
| 17.11% | 10.08% | `main` (self) |
| 16.47% | 0.00% | `std::thread` → `NetworkFlow::start()` |
| 16.40% | 0.43% | `NetworkIO::recv_send()` |
| 16.40% | 13.67% | `OrderFlow::start()` lambda |

Structure mirrors RxTx P-core exactly — same dispatcher pattern, same self-time distribution. `NetworkIO::recv_send()` at 0.43% self is consistent with RxTx (0.40%). The absence of `BuilderFlow` and `ParserFlow` in the top 10 here is a sampling artifact from the shorter symbol list, not a structural change; `OrderFlow` at 13.67% self confirms the thread load is equivalent.

#### E-Core

| Overhead | Self | Symbol |
|---|---|---|
| 47.42% | 40.97% | `RiskFlow::start()` lambda |
| 46.69% | 41.32% | `OrderFlow::start()` lambda |
| 9.00% | 8.99% | `RiskEngine::check_risk()` |
| 8.51% | 8.48% | `ClickhouseWriter::store()` |

`RiskEngine::check_risk()` drops from 10.88% self (RxTx) to 8.99% in Rx-only. 
In Rx-only the Tx thread is absent — no orders are submitted, so `check_risk()` 
spins idle with no work to validate. The reduced self time reflects the lower 
call rate from zero outgoing order flow, not a change in per-call cost. 
The 36% E-core clock gap concern from RxTx remains valid regardless — 
frequency is fixed, only call volume changes.

---

### Cache & Memory

| Metric | Value | vs RxTx |
|---|---|---|
| L1-D miss rate (P-core) | 0.96% | Matches RxTx — Rx owns the data working set |
| LLC miss rate (P-core) | 0.11% | **Half of RxTx (0.22%)** — Tx working set removed |
| LLC miss rate (E-core) | 0.08% | Slightly higher than RxTx (0.05%) — sampling variance |
| L1-I misses (P-core) | 1.91B | 7% lower than RxTx — fewer syscalls |
| dTLB misses (P-core) | 179K | Lower than RxTx (249K) — Tx buffer allocations absent |
| iTLB misses (P-core) | 1.16M | **Higher than RxTx (700K)** — see note |

**iTLB misses at 1.16M in Rx-only vs 700K in RxTx is counterintuitive.** With fewer syscalls the iTLB should be under less pressure. The likely explanation is sampling normalization: in RxTx the iTLB miss events are distributed across more active threads and wall time is slightly longer, diluting the per-run count. The icache profile confirms the syscall chain is still the dominant iTLB stressor in Rx-only.

---

### False Sharing (C2C)

| Metric | Value | vs RxTx |
|---|---|---|
| Total shared cache lines | 335 | **91% fewer than RxTx (3,724)** |
| Load HITs on shared lines | 2,254 | 94% fewer |
| Local HITM | 772 | **95% fewer than RxTx (14,392)** |
| Locked accesses on shared lines | 151 | 94% fewer |

**HITM collapses from 14,392 to 772 in Rx-only.** The same 
boost::lockfree queues are active in both scenarios, but 
Rx-only HITM per queue is 11–17x lower than RxTx. With only 
the Rx path running, producer-consumer timing naturally 
staggers and queue pointer contention rarely materializes. 
Adding the Tx path in RxTx creates simultaneous bidirectional 
pressure on the same queues, amplifying contention sharply. 
boost::lockfree performs well under single-path load — the 
contention is a cross-path interaction effect, not a 
boost-specific deficiency. A custom queue with explicit 
cache line separation would eliminate the residual contention 
regardless.

---

### Flamegraph

[Rx Flamegraph](../assets/flamegraph_engine_Rx.svg){target="_blank"}

---

### Findings & Action Items

Rx-only surfaces no independent optimization candidates. All findings from RxTx hold and the Rx path contributes cleanly:

- LLC miss rate in isolation (0.11%) confirms the Rx working set is well-contained — the 0.22% in RxTx is Tx-driven.
- False sharing in Rx-only is negligible (772 HITM) — the `boost::lockfree` contention identified in RxTx is a cross-path artifact, not an Rx-internal issue.
- `RiskEngine::check_risk()` E-core placement concern from RxTx applies here at reduced load (8.99% vs 10.88%) — same root cause, lower magnitude.

---

## Results (Tx)

Transmit path only — no receive thread running. Used to isolate the Tx contribution to RxTx numbers. Tx is the dominant source of syscall pressure, icache pollution, LLC contention, and false sharing in the combined run.

| Field | Value | vs RxTx |
|---|---|---|
| Wall time | 6.57 s | ~7% shorter |
| User time | 44.54 s | Expected: ~1 thread fewer |
| Sys time | 6.72 s | **Near-identical to RxTx (6.68s)** — see note |
| CPUs utilized | 7.804 | Near-identical to RxTx |

**Sys time in Tx-only (6.72s) matches RxTx (6.68s) almost exactly,** which primarily reflects the heavy network overhead generated by our local fake server—acting as a test artifact—to simulate exchange traffic. Since the Rx-only sys time (6.16s) is also similar, this confirms that the overall kernel footprint is dominated by the active socket connections and baseline epoll machinery sustained between our engine and the simulator, rather than the internal routing of the Rx path itself.

---

### Key Metrics

| Metric | P-Core | E-Core | vs RxTx |
|---|---|---|---|
| IPC | 0.46 | 0.45 | Identical |
| Branch miss rate | 0.14% | 0.06% | Slightly lower — Tx path has simpler branching than Rx parse |
| L1-D miss rate | 0.63% | — | **Lower than RxTx (0.97%)** — see note |
| LLC miss rate | 0.41% | 0.03% | **P-core higher than RxTx (0.22%)** — see note |
| L1-I misses (P-core) | 2.23B | — | **Higher than RxTx (2.05B)** — Tx is the dominant icache stressor |

**L1-D miss rate drops to 0.63% in Tx-only vs 0.97% in RxTx.** The Rx parse path (packet struct reads, ring buffer writes) is the heavier L1-D consumer. Tx without Rx has a narrower data working set — packet build and send buffer access — which fits more comfortably in L1.

**LLC miss rate rises to 0.41% in Tx-only vs 0.22% in RxTx.** This is the most surprising inversion. In RxTx the Rx working set is hot in LLC from continuous use; removing Rx leaves the LLC colder between Tx accesses, increasing miss rate. The Tx send buffers and packet structs are accessed at high frequency but without the Rx path warming adjacent LLC lines, eviction pressure increases.

**L1-I misses at 2.23B in Tx-only exceed RxTx (2.05B).** Tx is confirmed as the dominant icache stressor. The icache profile shows `__tcp_transmit_skb` appearing at 58.5% overhead — a symbol absent from the RxTx and Rx profiles — indicating TCP stack traversal on the send path contributes meaningfully to kernel text pressure beyond `sendto` alone.

---

### Hotspot Analysis

#### P-Core

| Overhead | Self | Symbol |
|---|---|---|
| 82.77% | 0.00% | `libc` / `libstdc++` init (startup) |
| 17.23% | 0.00% | `main` |
| 17.22% | 10.41% | `main` (self) |
| 16.45% | 0.00% | `std::thread` → `NetworkFlow::start()` |
| 16.40% | 0.53% | `NetworkIO::recv_send()` |
| 16.35% | 15.39% | `OrderFlow::start()` lambda |

`NetworkIO::recv_send()` self time rises to 0.53% vs 0.40% in RxTx. In Tx-only the dispatcher is spending proportionally more time on the send path — with no Rx work interleaved, send loop iterations dominate its budget.

`OrderFlow` self at 15.39% vs 13.81% in RxTx — with no Rx parse feeding new orders, OrderFlow spins more on the send side per sample.

#### E-Core

| Overhead | Self | Symbol |
|---|---|---|
| 47.09% | 38.95% | `RiskFlow::start()` lambda |
| 46.56% | 43.78% | `OrderFlow::start()` lambda |
| 10.90% | 10.88% | `RiskEngine::check_risk()` |
| 6.10% | 6.08% | `ClickhouseWriter::store()` |

`RiskEngine::check_risk()` holds at 10.88% self — identical to RxTx. The Tx path drives risk checks at the same rate as the combined run, meaning Tx order flow is the primary risk check trigger. The Rx path contribution in RxTx is small enough to not move this number.

---

### Cache & Memory

| Metric | Value | vs RxTx |
|---|---|---|
| L1-D miss rate (P-core) | 0.63% | Lower — Rx parse working set absent |
| LLC miss rate (P-core) | 0.41% | **Higher than RxTx (0.22%)** — LLC colder without Rx |
| L1-I misses (P-core) | 2.23B | Higher than RxTx — Tx is the dominant icache source |
| dTLB misses (P-core) | 60K | **75% lower than RxTx (249K)** — Rx struct allocations absent |
| iTLB misses (P-core) | 1.28M | Higher than RxTx (700K) — Tx syscall stream undiluted |

**dTLB misses at 60K vs RxTx 249K** — the largest relative delta across all metrics. The Rx path (packet struct population, ring buffer slot mapping) generates the majority of data TLB pressure in the combined run. Tx alone touches a narrow, stable set of data addresses.

iTLB at 1.28M vs 700K in RxTx mirrors the L1-I pattern — Tx syscall stream alone generates more iTLB pressure than the combined run's average, because in RxTx the Rx thread's non-syscall samples dilute the iTLB miss rate.

---

### False Sharing (C2C)

| Metric | Value | vs RxTx |
|---|---|---|
| Total shared cache lines | 1,179 | 68% fewer than RxTx (3,724) |
| Load HITs on shared lines | 18,927 | 52% fewer |
| Local HITM | 5,669 | **39% of RxTx total (14,392)** |
| Locked accesses on shared lines | 1,514 | 43% fewer |

**Tx-only produces 5,669 HITM — 39% of the RxTx total.** Combined 
with Rx-only's 772 HITM, cross-path interaction in RxTx generates 
the remaining ~8,000 HITM beyond what either path produces in 
isolation. In Tx-only the dominant HITM sources shift — test 
artifacts (BuilderFlow/main lambda interaction, FakeTCPServer) and 
kernel network stack spin locks lead the cache line table, with 
boost::lockfree contributing at index 4 (104 HITM) via the 
NetworkFlow→Builder queue. The boost RiskFlow/OrderFlow contention 
that dominates RxTx is absent here because without the Rx parse 
path feeding orders, that queue sees minimal bidirectional pressure.

Tx-only's 1,179 shared cache lines vs RxTx's 3,724 confirms that 
RxTx contention is not simply additive — the cross-path interaction 
creates contention on lines that neither path stresses alone.

---

### Flamegraph

[Tx Flamegraph](../assets/flamegraph_engine_Tx.svg){target="_blank"}

---

### Findings & Action Items

Tx-only confirms and sharpens the RxTx findings. No new optimization candidates emerge, but causality is clearer:

- Sys time equivalence (Rx=6.16s ≈ Tx=6.72s ≈ RxTx=6.68s) proves the syscall budget is not directly related to Rx or Tx paths themselves.
- L1-I miss leadership (Tx=2.23B > RxTx=2.05B > Rx=1.91B) confirms Tx as the icache stressor. `__tcp_transmit_skb` appearing in the Tx icache profile but not in RxTx or Rx indicates the TCP stack traversal is a secondary contributor alongside `sendto`.
- HITM at 5,669 in Tx-only, 772 in Rx-only, 14,392 in RxTx: the ~8K gap is interaction-driven. The `boost::lockfree` replacement identified in RxTx findings eliminates the root cause regardless of which path is active.
- LLC miss inversion (Tx=0.41% > RxTx=0.22%) is expected behavior — not a regression, an isolation artifact. In production the combined working set keeps LLC warmer.

---

## Raw Output

### RxTx RAW

"Perf Stat — Base"
    ```
    task-clock:         55,503,157,765       7.877 CPUs utilized
    context-switches:          176           3,171 /sec
    cpu-migrations:             11           0.198 /sec
    page-faults:            21,119         380,501 /sec

    P-Core:
      instructions:    63,405,708,915       0.46 insn/cycle
      cycles:         137,601,082,473       2.479 GHz
      branches:        12,030,765,585     216.758 M/sec
      branch-misses:       21,201,120       0.18%
      L1-dcache-loads: 18,138,690,800     326.805 M/sec
      L1-dcache-misses:   175,387,197       0.97%
      LLC-loads:          126,819,047       2.285 M/sec
      LLC-load-misses:        284,728       0.22%

    E-Core:
      instructions:    16,443,671,642       0.44 insn/cycle
      cycles:          37,422,191,537       0.674 GHz
      branches:         3,085,564,133      55.593 M/sec
      branch-misses:        4,511,257       0.15%
      L1-dcache-loads:  4,862,877,380      87.614 M/sec
      LLC-loads:           16,454,371     296.458 K/sec
      LLC-load-misses:          8,058       0.05%

    Wall: 7.046s  User: 48.823s  Sys: 6.682s
    ```

"Perf Stat — ICache"
    ```
    P-Core L1-icache-load-misses:  2,053,630,364
    E-Core L1-icache-load-misses:      173,383

    Wall: 7.070s  User: 48.972s  Sys: 6.767s
    ```

"Perf Stat — TLB"
    ```
    P-Core dTLB-load-misses:   249,801
    P-Core iTLB-load-misses:   700,098
    E-Core dTLB-load-misses:     3,339
    E-Core iTLB-load-misses:   381,406

    Wall: 7.143s  User: 49.524s  Sys: 6.816s
    ```

"Perf Record — P-Core (top 20)"
    ```
    82.81%  0.00%  libc.so.6             0x00007f2a16929c6c
    82.81%  0.00%  libc.so.6             0x00007f2a1689caa4
    82.81%  0.00%  libstdc++.so.6.0.33   0x00007f2a16cecdb4
    17.19%  0.00%  engine_RxTx_perf      _start
    17.19%  0.00%  libc.so.6             __libc_start_main
    17.19%  0.00%  libc.so.6             0x00007f2a1682a1ca
    17.19%  0.00%  engine_RxTx_perf      main
    16.47%  0.00%  engine_RxTx_perf      std::thread → NetworkFlow::start()
    16.44%  9.57%  engine_RxTx_perf      main::{lambda(int,int)#1}::operator()
    16.39%  0.40%  engine_RxTx_perf      NetworkIO::recv_send()
    16.39%  13.81% engine_RxTx_perf      std::thread::_State_impl<std::thread::_Invoker<std::tuple<OrderFlow::start()::{lambda()#1}> > >::_M_run()
    16.05%  14.35% engine_RxTx_perf      std::thread::_State_impl<std::thread::_Invoker<std::tuple<RiskFlow::start()::{lambda()#2}> > >::_M_run()
    15.66%  15.65% engine_RxTx_perf      std::thread::_State_impl<std::thread::_Invoker<std::tuple<BuilderFlow::start()::{lambda()#1}> > >::_M_run()
    15.39%  14.62% engine_RxTx_perf      std::thread::_State_impl<std::thread::_Invoker<std::tuple<ParserFlow::start()::{lambda()#1}> > >::_M_run()
    13.51%  0.24%  [kernel.kallsyms]     entry_SYSCALL_64_after_hwframe
    13.25%  0.43%  [kernel.kallsyms]     do_syscall_64
    11.83%  0.16%  [kernel.kallsyms]     x64_sys_call
     9.44%  1.33%  libc.so.6             epoll_wait
     4.66%  0.00%  [kernel.kallsyms]     __x64_sys_sendto
     4.64%  0.04%  [kernel.kallsyms]     __sys_sendto
    ```

"Perf Record — E-Core (top 10)"
    ```
    100.00%  0.00%  libc.so.6             0x00007fae54929c6c
    100.00%  0.00%  libc.so.6             0x00007fae5489caa4
    100.00%  0.00%  libstdc++.so.6.0.33   0x00007fae54cecdb4
     47.26% 38.96%  engine_RxTx_perf      RiskFlow::start() lambda
     46.91% 41.60%  engine_RxTx_perf      OrderFlow::start() lambda
     10.92% 10.88%  engine_RxTx_perf      RiskEngine::check_risk()
      8.31%  8.28%  engine_RxTx_perf      ClickhouseWriter::store()
      0.17%  0.17%  [kernel.kallsyms]     __irqentry_text_end
      0.05%  0.01%  [kernel.kallsyms]     asm_sysvec_apic_timer_interrupt
      0.04%  0.04%  engine_RxTx_perf      RiskEngine::check_venue_halt_and_circuit()
    ```

"Perf Record — ICache (top 10)"
    ```
    88.07%  0.49%  [kernel]  entry_SYSCALL_64_after_hwframe
    87.52%  0.63%  [kernel]  do_syscall_64
    85.83%  0.30%  [kernel]  x64_sys_call
    64.19%  0.00%  engine_RxTx_perf  _start
    64.19%  0.00%  libc.so.6         __libc_start_main
    64.18%  0.00%  libc.so.6         0x00007f6e68e2a1ca
    64.18%  0.00%  engine_RxTx_perf  main
    63.94%  1.74%  engine_RxTx_perf  main::{lambda(int,int)#1}::operator()
    51.03%  0.05%  [kernel]  __x64_sys_sendto
    50.93%  0.59%  [kernel]  __sys_sendto
    ```

<a id="rxtx-c2c"></a>

"C2C — False Sharing Summary" 
    
    
    Total records:                 291,203
    Locked Load/Store Operations:   14,591
    Load Operations:               115,262
    Load Fill Buffer Hit:           27,015
    Load L1D hit:                   42,538
    Load LLC hit:                   20,118
    Load Local HITM:                14,392
    Load Local DRAM:                    21
    Store Operations:              103,238
    Store L1D Hit:                 100,477

    Total Shared Cache Lines:        3,724
    Load HITs on shared lines:      39,792
    Fill Buffer Hits on shared lines: 17,766
    LLC hits on shared lines:        19,403
    Locked Access on shared lines:    2,637
    Blocked Access on shared lines:   3,023
    Store HITs on shared lines:       3,399


    Total records:                 291,203
    Locked Load/Store Operations:   14,591
    Load Operations:               115,262
    Load Fill Buffer Hit:           27,015
    Load L1D hit:                   42,538
    Load LLC hit:                   20,118
    Load Local HITM:                14,392
    Load Local DRAM:                    21
    Store Operations:              103,238
    Store L1D Hit:                 100,477
    Total Shared Cache Lines:        3,724
    Load HITs on shared lines:      39,792
    Fill Buffer Hits on shared lines: 17,766
    LLC hits on shared lines:        19,403
    Locked Access on shared lines:    2,637
    Blocked Access on shared lines:   3,023
    Store HITs on shared lines:       3,399

    Index     Address               LclHitm    %       Source
      0 🔴    0x7f65ed8e8fc0           861    5.98%    std::mutex internal — Parser_Dispatch/NetworkFlow contention
      1 🔴    0x7f65ed8f3080           725    5.04%    boost::lockfree ringbuffer — RiskFlow/OrderFlow queue pointer
      2       0x7f65ed8eb000           529    3.68%    boost::lockfree spsc — OrderFlow/ParserFlow queue pointer
      3       0x7f65eee88d80           417    2.90%    test artifact — RiskEngine::pipeline_seq (main polls, RiskFlow writes)
      4       0x7f65ed8eb040           399    2.77%    boost::lockfree spsc — ParserFlow/OrderFlow queue pointer
      5       0x7f65ef288dc0           275    1.91%    test artifact — pipeline_seq cache line padding (prefetcher)
      6       0x7f65edda0600           223    1.55%    boost::lockfree spsc — NetworkFlow pushes RxPacket, ParserFlow pops
      7       0x7f65eda731c0           195    1.35%    test artifact — main lambda internal (single thread)
      8       0x7f65edb79300           153    1.06%    boost::lockfree ringbuffer — OrderFlow queue pointer
      9       0x7f65ed8f30c0           145    1.01%    boost::lockfree ringbuffer — OrderFlow/RiskFlow queue pointer
     10   0xffff898787f7f200           141    0.98%    kernel epoll mutex — NetworkIO epoll_wait
     11       0x7f65edb81300           135    0.94%    test artifact — Parser_Dispatch::flush_DbQueue (test cleanup)

>All critical (red) entries are boost::lockfree infrastructure.
Indices 3, 5, 7, 11 are test-only artifacts — absent in production.
No application struct false sharing detected in top 12 cache lines.

back to [RxTx False Sharing (C2C)](#rxtx-c2c-back). 

### Rx RAW

"Perf Stat — Base"
```
task-clock:         51,999,859,979       7.881 CPUs utilized
context-switches:          167           3,212 /sec
cpu-migrations:             11           0.212 /sec
page-faults:             1,515          29,135 /sec

P-Core:
  instructions:    58,218,951,907       0.45 insn/cycle
  cycles:         128,427,815,050       2.470 GHz
  branches:        11,089,071,300     213.252 M/sec
  branch-misses:       20,656,780       0.19%
  L1-dcache-loads: 16,604,026,455     319.309 M/sec
  L1-dcache-misses:   159,127,875       0.96%
  LLC-loads:          121,689,033       2.340 M/sec
  LLC-load-misses:        131,450       0.11%

E-Core:
  instructions:    15,445,639,375       0.44 insn/cycle
  cycles:          34,903,736,966       0.671 GHz
  branches:         2,885,167,628      55.484 M/sec
  branch-misses:        2,633,615       0.09%
  L1-dcache-loads:  4,586,871,442      88.209 M/sec
  LLC-loads:            9,917,900     190.729 K/sec
  LLC-load-misses:          8,104       0.08%

Wall: 6.598s  User: 45.839s  Sys: 6.162s
```

"Perf Stat — ICache"
```
P-Core L1-icache-load-misses:  1,906,141,916
E-Core L1-icache-load-misses:      154,308

Wall: 6.538s  User: 45.365s  Sys: 6.165s
```

"Perf Stat — TLB"
```
P-Core dTLB-load-misses:   179,334
P-Core iTLB-load-misses: 1,162,250
E-Core dTLB-load-misses:     3,335
E-Core iTLB-load-misses:   350,133

Wall: 6.567s  User: 45.455s  Sys: 6.309s
```

"Perf Record — P-Core (top 10)"
```
82.87%  0.00%  engine_Rx_perf  libc.so.6            0x000071b61c929c6c
82.87%  0.00%  engine_Rx_perf  libc.so.6            0x000071b61c89caa4
82.87%  0.00%  engine_Rx_perf  libstdc++.so.6.0.33  0x000071b61ccecdb4
17.12%  0.00%  engine_Rx_perf  engine_Rx_perf       _start
17.12%  0.00%  engine_Rx_perf  libc.so.6            __libc_start_main
17.12%  0.00%  engine_Rx_perf  libc.so.6            0x000071b61c82a1ca
17.11% 10.08%  engine_Rx_perf  engine_Rx_perf       main
16.47%  0.00%  engine_Rx_perf  engine_Rx_perf       std::thread → NetworkFlow::start()
16.40%  0.43%  engine_Rx_perf  engine_Rx_perf       NetworkIO::recv_send()
16.40% 13.67%  engine_Rx_perf  engine_Rx_perf       OrderFlow::start() lambda
```

"Perf Record — E-Core (top 10)"
```
100.00%  0.00%  engine_Rx_perf  libc.so.6            0x00007622ce729c6c
100.00%  0.00%  engine_Rx_perf  libc.so.6            0x00007622ce69caa4
100.00%  0.00%  engine_Rx_perf  libstdc++.so.6.0.33  0x00007622ceaecdb4
 47.42% 40.97%  engine_Rx_perf  engine_Rx_perf       RiskFlow::start() lambda
 46.69% 41.32%  engine_Rx_perf  engine_Rx_perf       OrderFlow::start() lambda
  9.00%  8.99%  engine_Rx_perf  engine_Rx_perf       RiskEngine::check_risk()
  8.51%  8.48%  engine_Rx_perf  engine_Rx_perf       ClickhouseWriter::store()
  0.13%  0.01%  engine_Rx_perf  [kernel.kallsyms]    asm_sysvec_apic_timer_interrupt
  0.11%  0.00%  engine_Rx_perf  [kernel.kallsyms]    sysvec_apic_timer_interrupt
  0.11%  0.01%  engine_Rx_perf  [kernel.kallsyms]    hrtimer_interrupt
```

"Perf Record — ICache (top 10)"
```
88.29%  0.44%  engine_Rx_perf  [kernel.kallsyms]    entry_SYSCALL_64_after_hwframe
87.78%  0.63%  engine_Rx_perf  [kernel.kallsyms]    do_syscall_64
85.95%  0.35%  engine_Rx_perf  [kernel.kallsyms]    x64_sys_call
61.91%  0.00%  engine_Rx_perf  engine_Rx_perf       _start
61.91%  0.00%  engine_Rx_perf  libc.so.6            __libc_start_main
61.91%  0.00%  engine_Rx_perf  libc.so.6            0x00007dacd802a1ca
61.82%  3.12%  engine_Rx_perf  engine_Rx_perf       main
52.99%  0.08%  engine_Rx_perf  [kernel.kallsyms]    __x64_sys_sendto
52.83%  0.95%  engine_Rx_perf  [kernel.kallsyms]    __sys_sendto
49.78%  0.12%  engine_Rx_perf  [kernel.kallsyms]    inet_sendmsg
```

"Perf Record — TLB (top 10)"
```
57.28%  0.00%  engine_Rx_perf  engine_Rx_perf       _start
57.28%  0.00%  engine_Rx_perf  libc.so.6            __libc_start_main
57.28%  0.00%  engine_Rx_perf  libc.so.6            0x0000708d5ee2a1ca
57.27%  0.10%  engine_Rx_perf  engine_Rx_perf       main
42.64%  0.00%  engine_Rx_perf  libc.so.6            0x0000708d5ef29c6c
42.63%  0.00%  engine_Rx_perf  libc.so.6            0x0000708d5ee9caa4
42.62%  0.00%  engine_Rx_perf  libstdc++.so.6.0.33  0x0000708d5f2ecdb4
38.86%  0.00%  engine_Rx_perf  engine_Rx_perf       OrderFlow::start() lambda
38.53%  0.33%  engine_Rx_perf  engine_Rx_perf       OrderManager::store()
31.08%  0.00%  engine_Rx_perf  engine_Rx_perf       TradingEngine::TradingEngine()
```

"C2C — False Sharing Summary"
```
Total records:               1,275,239
Locked Load/Store Operations:   105,695
Load Operations:                418,345
Load Fill Buffer Hit:             1,904
Load L1D hit:                   270,055
Load LLC hit:                     1,152
Load Local HITM:                    772
Load Local DRAM:                     12
Store Operations:               708,724
Store L1D Hit:                  708,022

Total Shared Cache Lines:           335
Load HITs on shared lines:        2,254
Fill Buffer Hits on shared lines:  1,059
LLC hits on shared lines:          1,069
Locked Access on shared lines:       151
Blocked Access on shared lines:      191
Store HITs on shared lines:          280
```

---

### Tx RAW

"Perf Stat — Base"
```
task-clock:         51,261,264,842       7.804 CPUs utilized
context-switches:          184           3,589 /sec
cpu-migrations:             12           0.234 /sec
page-faults:            96,567           1,884 K/sec

P-Core:
  instructions:    58,285,294,995       0.46 insn/cycle
  cycles:         126,626,647,382       2.470 GHz
  branches:        10,990,080,251     214.393 M/sec
  branch-misses:       15,565,918       0.14%
  L1-dcache-loads: 16,712,643,267     326.029 M/sec
  L1-dcache-misses:   105,565,049       0.63%
  LLC-loads:           93,146,715       1.817 M/sec
  LLC-load-misses:        380,493       0.41%

E-Core:
  instructions:    15,562,864,599       0.45 insn/cycle
  cycles:          34,311,590,912       0.669 GHz
  branches:         2,908,384,397      56.736 M/sec
  branch-misses:        1,886,295       0.06%
  L1-dcache-loads:  4,599,778,730      89.732 M/sec
  LLC-loads:            7,585,325     147.974 K/sec
  LLC-load-misses:          2,462       0.03%

Wall: 6.569s  User: 44.544s  Sys: 6.719s
```

"Perf Stat — ICache"
```
P-Core L1-icache-load-misses:  2,225,136,364
E-Core L1-icache-load-misses:      164,624

Wall: 6.519s  User: 44.489s  Sys: 6.709s
```

"Perf Stat — TLB"
```
P-Core dTLB-load-misses:    59,997
P-Core iTLB-load-misses: 1,277,849
E-Core dTLB-load-misses:     3,682
E-Core iTLB-load-misses:   376,268

Wall: 6.509s  User: 44.426s  Sys: 6.711s
```

"Perf Record — P-Core (top 10)"
```
82.77%  0.00%  engine_Tx_perf  libc.so.6            0x00007d8438d29c6c
82.77%  0.00%  engine_Tx_perf  libc.so.6            0x00007d8438c9caa4
82.77%  0.00%  engine_Tx_perf  libstdc++.so.6.0.33  0x00007d84390ecdb4
17.23%  0.00%  engine_Tx_perf  engine_Tx_perf       _start
17.23%  0.00%  engine_Tx_perf  libc.so.6            __libc_start_main
17.23%  0.00%  engine_Tx_perf  libc.so.6            0x00007d8438c2a1ca
17.22% 10.41%  engine_Tx_perf  engine_Tx_perf       main
16.45%  0.00%  engine_Tx_perf  engine_Tx_perf       std::thread → NetworkFlow::start()
16.40%  0.53%  engine_Tx_perf  engine_Tx_perf       NetworkIO::recv_send()
16.35% 15.39%  engine_Tx_perf  engine_Tx_perf       OrderFlow::start() lambda
```

"Perf Record — E-Core (top 10)"
```
100.00%  0.00%  engine_Tx_perf  libc.so.6            0x00007785d2729c6c
100.00%  0.00%  engine_Tx_perf  libc.so.6            0x00007785d269caa4
100.00%  0.00%  engine_Tx_perf  libstdc++.so.6.0.33  0x00007785d2aecdb4
 47.09% 38.95%  engine_Tx_perf  engine_Tx_perf       RiskFlow::start() lambda
 46.56% 43.78%  engine_Tx_perf  engine_Tx_perf       OrderFlow::start() lambda
 10.90% 10.88%  engine_Tx_perf  engine_Tx_perf       RiskEngine::check_risk()
  6.10%  6.08%  engine_Tx_perf  engine_Tx_perf       ClickhouseWriter::store()
  0.16%  0.02%  engine_Tx_perf  [kernel.kallsyms]    asm_sysvec_apic_timer_interrupt
  0.14%  0.00%  engine_Tx_perf  [kernel.kallsyms]    sysvec_apic_timer_interrupt
  0.12%  0.00%  engine_Tx_perf  [kernel.kallsyms]    __sysvec_apic_timer_interrupt
```

"Perf Record — ICache (top 10)"
```
93.06%  0.47%  engine_Tx_perf  [kernel.kallsyms]    entry_SYSCALL_64_after_hwframe
92.56%  0.67%  engine_Tx_perf  [kernel.kallsyms]    do_syscall_64
90.60%  0.46%  engine_Tx_perf  [kernel.kallsyms]    x64_sys_call
58.51%  2.92%  engine_Tx_perf  [kernel.kallsyms]    __tcp_transmit_skb
56.74%  0.00%  engine_Tx_perf  libc.so.6            0x000077ed5cf29c6c
56.74%  0.00%  engine_Tx_perf  libc.so.6            0x000077ed5ce9caa4
56.74%  0.00%  engine_Tx_perf  libstdc++.so.6.0.33  0x000077ed5d2ecdb4
56.52%  0.00%  engine_Tx_perf  engine_Tx_perf       std::thread → NetworkFlow::start()
56.50%  0.34%  engine_Tx_perf  engine_Tx_perf       NetworkIO::recv_send()
54.59%  1.01%  engine_Tx_perf  engine_Tx_perf       NetworkIO::send(TxPacket&, int, unsigned char, SocketState&)
```

"Perf Record — TLB (top 10)"
```
74.66%  0.00%  engine_Tx_perf  engine_Tx_perf       _start
74.66%  0.00%  engine_Tx_perf  libc.so.6            __libc_start_main
74.66%  0.00%  engine_Tx_perf  libc.so.6            0x00007cca1a22a1ca
74.66%  0.00%  engine_Tx_perf  engine_Tx_perf       main
70.70%  0.00%  engine_Tx_perf  libc.so.6            0x00007cca1a38954a
70.46%  0.00%  engine_Tx_perf  engine_Tx_perf       TradingEngine::TradingEngine()
60.57% 60.57%  engine_Tx_perf  libc.so.6            0x000000000018954a
27.55%  0.00%  engine_Tx_perf  [kernel.kallsyms]    entry_SYSCALL_64_after_hwframe
27.55%  0.00%  engine_Tx_perf  [kernel.kallsyms]    do_syscall_64
27.53%  0.00%  engine_Tx_perf  [kernel.kallsyms]    x64_sys_call
```

"C2C — False Sharing Summary"
```
Total records:                 207,591
Locked Load/Store Operations:    9,011
Load Operations:                66,768
Load Fill Buffer Hit:           12,575
Load L1D hit:                   30,985
Load LLC hit:                    6,376
Load Local HITM:                 5,669
Load Local DRAM:                    14
Store Operations:               98,129
Store L1D Hit:                  96,875

Total Shared Cache Lines:        1,179
Load HITs on shared lines:      18,927
Fill Buffer Hits on shared lines: 11,285
LLC hits on shared lines:        6,160
Locked Access on shared lines:   1,514
Blocked Access on shared lines:  1,274
Store HITs on shared lines:      2,004
```

