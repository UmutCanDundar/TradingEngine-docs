# Project Measurement Summary

> **Current state of the system's main pipelines (hot path).** Last updated: 2026-10-30. This page is refreshed after every change; it always shows the latest measurements only.
>
> **Measurement:** values are medians; cv of time ≤ 1.0% and of p50 ≤ 1.4% on every leg (max and p999 cycles are inherently noisy). Threads pinned per core (`SCHED_FIFO`), SMT on, `performance` governor, turbo on. Host: 13th-gen Intel Core i7-13620H (hybrid P/E cores).

* * *

## 1. End-to-End Latency (benchmarks)

### Rx (Network → Risk)

| Leg | Median (ns) | p50 cycles | p95 cycles | p99 cycles | p999 cycles | cv (time) |
|---|---|---|---|---|---|---|
| FIX | 3,149 | 8,419 | 8,808 | 9,226 | 15,074 | 0.61% |
| OUCH BIST | 3,078 | 8,202 | 8,593 | 8,971 | 14,876 | 0.59% |
| OUCH NQ | 3,099 | 8,277 | 8,649 | 9,055 | 15,128 | 0.96% |
| ITCH BIST | 1,753 | 4,981 | 5,337 | 5,511 | 9,035 | 1.00% |
| ITCH NQ | 1,711 | 4,846 | 5,188 | 5,374 | 8,987 | 0.82% |

### Tx (Risk → Network)

| Leg | Median (ns) | p50 cycles | p95 cycles | p99 cycles | p999 cycles | cv (time) |
|---|---|---|---|---|---|---|
| FIX | 2,753 | 7,852 | 8,224 | 10,211 | 14,449 | 0.70% |
| OUCH BIST | 2,567 | 7,313 | 7,624 | 7,998 | 13,887 | 0.82% |
| OUCH NQ | 2,668 | 7,580 | 7,972 | 8,232 | 14,227 | 0.78% |

### Sum of the measured legs

| Path | Latency |
|---|---|
| Market data in (ITCH Rx) | **1.71 – 1.75 µs** |
| Order-flow in (FIX / OUCH Rx) | **3.08 – 3.15 µs** |
| Order out (Tx) | **2.57 – 2.75 µs** |
| Order round trip, Rx + Tx | FIX **5.90 µs** · OUCH BIST **5.65 µs** · OUCH NQ **5.77 µs** |
| ITCH in → OUCH order out | BIST **4.32 µs** · NQ **4.38 µs** |

The sums are arithmetic sums of separately measured legs. They exclude strategy logic and wire time, and the legs were not measured as one continuous path. p99 stays within +9…+30% of p50 on every leg; p999 is ≈1.8–1.9× p50.

* * *

## 2. Perf (RxTx integration)

| Counter | Value |
|---|---|
| P-core cycles / instructions | 122.25B / 59.69B (IPC 0.49) |
| E-core cycles | 33.05B |
| Branch miss rate | 0.19% (20.6M of 10.65B) |
| L1-D load misses | 138.1M |
| L1-I load misses | 2.08B |
| LLC loads / LLC load misses | 112.2M / 714k |
| dTLB / iTLB misses | 282k / 1.10M |
| Shared cache lines / Local HITM (`perf c2c`, sampled) | 3,827 / 13,379 |

IPC of 0.49 is expected for this design: the pipeline threads busy-poll their queues, and wait loops (especially with `pause`) and the syscall / `epoll` path retire very few instructions per cycle. It is therefore not a measure of the work done per message; use the per-leg latencies above. dTLB and iTLB misses are negligible (1.4M events in total, well under 0.3% of cycles even at 200 cycles per miss).

* * *

## 3. Findings

Where the time and the misses go, taken from the function-level perf profiles:

1. **Most cycles are spent in wait loops.** Every pipeline thread shows 14–16% of P-core samples; the E-core `RiskFlow` / `OrderFlow` lambdas show 36–39% of E-core samples. This is the busy-poll design (lowest wake-up latency at the cost of burning a core), not per-message work.
2. **Cache misses come from the network path, not from the engine stages.** In both the D-cache and I-cache miss profiles the only functions above 1% are `NetworkIO::recv_send()` (30.5% D / 36.3% I, inclusive) and the lambda that drives it (42.3% D / 58.1% I, inclusive). Parser, Order Manager, Risk and Builder do not exceed 1% individually. The largest share of the 2.08B L1-I misses is therefore on the socket / syscall path and its kernel code.
3. **Network and kernel cost is not engine code.** `NetworkIO::recv_send()` is 16.3% and `epoll_wait` 9.8% of P-core samples, with unresolved kernel frames. In this benchmark the socket path is loopback TCP against an in-process `FakeTCPServer`, so the kernel cost seen here is the general-purpose OS socket stack (syscalls, `epoll`, skb handling, wake-ups), not engine code, and a real NIC path will look different. It cannot be reduced inside the current I/O model (see conclusion).
4. **Cross-core cache-line traffic (sampled `perf c2c`, ~10–13k local HITM, 3,827 shared lines).** In the 119 listed lines (41% of all HITM) about 45% sit on kernel addresses (loopback socket data: `__libc_send` / `__libc_recv` / `FakeTCPServer`) and about 55% in user space. Of the user-space part: Order objects handed from the producer (`Filler*`) to the Risk Update thread ~26%, parser output / DB queue ~21%, the Risk Update SPSC queue index ~21%, the benchmark's `pipeline_seq` progress counter ~12% (instrumentation, not in production), the Order Manager awaiting-ack container ~9%, MarketBook best bid/ask read by `update_symbol_risk` ~5%, RX buffer hand-off ~4%. All are true sharing: a producer thread writes, a consumer thread reads, and the line moves once per hand-off. No `AccountRisk` / `SymbolRisk` / `OrderRisk` line and no `check_risk` function appears among the shared lines of this run. An Order is read by the consumer from at least two cache lines (e.g. `update_risk()` at +0x38 of one line, `update_risk_for_protocol_fix` at +0x8 of the next), each costing a transfer of roughly 140–175 cycles per sample.
5. **TLB.** Misses are negligible (see section 2); no hot-path function stands out.
6. **Tail.** p999 ≈ 1.8–1.9× p50, p99 stays tight. The profiles do not show an engine function responsible; the remaining candidates sit in the OS / hardware layer (interrupts, frequency scaling on a laptop part).

* * *

## 4. Conclusion

**Software hot path: in good shape.** The engine behaves tightly (p99 within +9…+30% of p50, and no engine stage exceeds 1% in the cache-miss profiles). The measured legs are **1.7 µs (market data in), 3.1 µs (order-flow in) and 2.6–2.75 µs (order out)**, about **4.3 µs ITCH → order** and **5.6–5.9 µs order round trip**, before strategy time and before any real wire, NIC and exchange-gateway latency. What remains of that cost sits in the kernel socket path (`recv_send`, `epoll_wait`, syscalls) on a general-purpose OS and a hybrid-core development host, and it cannot be removed by tuning the engine.

What a production deployment would add:

1. **Kernel-bypass I/O** (e.g. DPDK / AF_XDP / a vendor user-space stack) with busy-polled NIC rings in place of `epoll` + socket syscalls. This is the dominant remaining cost.
2. **Separate pipelines per protocol.** Market data (ITCH, 1.7 µs) and order flow (FIX / OUCH, 3.1 µs) differ by ~1.4 µs and should not share cores or queues, so the heavier order-flow path never delays market data.
3. **Server-grade host.** Dedicated, isolated, fixed-frequency P-cores only (Risk Check and the database run on E-cores today), a NIC with hardware timestamping, then an end-to-end re-measurement with wire timestamps.


**Please Note That:**
As observed in the current measurements, the system's metrics do not reflect ultra-low-latency production standards, and for millisecond-scale strategies, these figures are far more than sufficient. This is a deliberate architectural outcome rather than an oversight.

This project was built primarily for educational breadth and architectural exploration, prioritizing multi-protocol support, complex state management, and broad exchange integration. To achieve this, the codebase has been structured with strict modularity, ensuring that components, protocol parsers, and connection managers are cleanly decoupled.

Consequently, while the current broad-scope implementation serves as a robust learning and testing ground, this modular foundation will allow the architecture to be easily scaled down, stripped of generalized abstractions, and specialized into a focused, production-grade, ultra-low-latency engine dedicated strictly to single-exchange protocols (such as BIST OUCH/ITCH) in future iterations.
