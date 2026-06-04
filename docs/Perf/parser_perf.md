# Parser — Perf Analysis

Profiling of the parser under sustained load across all supported protocols. The parser dispatch test runs all protocols concurrently through the full dispatch pipeline. Per-protocol tests isolate each parser individually to expose protocol-specific cache and branch behavior.

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

## Protocol Comparison at a Glance {#protocol-comparison-back}

Before diving into individual results, the cross-protocol comparison reveals the most important structural difference in parser behavior.

| Protocol | P-core IPC | L1-D miss | L1-I misses | Branch miss | Local HITM |
|---|---|---|---|---|---|
| Dispatch (all) | 0.46 | 4.96% | 12.3M | 0.37% | 16,880 |
| FIX | 1.01 | 2.90% | 208K | 0.28% | 4,528 |
| ITCH BIST | 0.19 | 10.32% | 44K | 2.37% | 2,243 |
| ITCH NASDAQ | 0.20 | 10.27% | 54K | 2.56% | 2,860 |
| OUCH BIST | 0.18 | 11.36% | 247K | 1.59% | 9,265 |
| OUCH NASDAQ* | 2.26 | 0.30% | 5K | 0.01% | 0 |

*OUCH NASDAQ was profiled as an isolated parse function test (no dispatch pipeline) — see [OUCH NASDAQ](#results(ouch-nasdaq)) for context.

**The dominant pattern:** FIX is compute-bound (IPC 1.01) with moderate cache pressure. ITCH and OUCH are severely memory-bound (IPC 0.18–0.20) — their high L1-D miss rates are driven by SPSC queue cache line contention, not struct complexity — as confirmed by the OUCH NASDAQ isolation test (IPC 2.26, L1-D miss 0.30% with no queue). The dispatch overhead and boost queue contention compound this in the combined test.

---

## Results (Parser Dispatch)

All protocols running concurrently through the full dispatch pipeline. Two worker threads plus main, 3 CPUs utilized.

| Field | Value |
|---|---|
| Wall time | 10.70 s |
| User time | 31.64 s |
| Sys time | 0.13 s |
| CPUs utilized | 2.970 |

### Key Metrics

| Metric | Value | Note |
|---|---|---|
| IPC | 0.46 | P-core memory-bound |
| Branch miss rate | 0.37% | Low |
| L1-D miss rate | 4.96% | Elevated — queue write + struct access |
| LLC miss rate | 0.03% | Low — working set fits in LLC |

**P-core at 0.46 is stalling on memory** — queue writes, struct pointer chasing. The bottleneck is not the parsing itself but the inter-thread data movement via boost queue.

**L1-D miss at 4.96%** is the combined effect of boost queue head/tail pointer writes and the high L1-D miss inherent to ITCH/OUCH message structs (10%+ in isolation). FIX's lower miss rate (2.90%) partially offsets this.

### Hotspot Analysis

| Overhead | Self | Symbol |
|---|---|---|
| 33.14% | 31.31% | `main::{lambda()#1}` — feeder thread |
| 33.12% | 33.10% | `main::{lambda()#2}` — consumer thread |
| 33.07% | 9.72% | `main` |
| 23.34% | 2.78% | `Parser_Dispatch::dispatch()` |

Two near-equal worker threads dominate — feeder pushing packets, consumer dispatching. `dispatch()` at 2.78% self confirms it is a thin dispatcher, not itself the bottleneck. The 9.72% self on `main` reflects test driver overhead.

### Cache & Memory

| Metric | Value | Note |
|---|---|---|
| L1-D miss rate | 4.96% | Mixed — queue writes + ITCH/OUCH struct access |
| LLC miss rate | 0.03% | Very low — working set in LLC |
| L1-I misses | 12.3M | Moderate — dispatch table + multiple parse paths |
| dTLB misses | 1,452K | Elevated — boost queue buffer spans many pages |
| iTLB misses | 176K | Low |

**dTLB misses at 1.45M** stand out — the boost spsc queue with 1024 capacity holds large RxPacket pointers and its internal buffer spans multiple pages. Each queue push/pop potentially crosses a page boundary, adding TLB pressure. This disappears in the without-push test (153K dTLB misses) — confirming the queue buffer as the source.

### False Sharing (C2C)

| Metric | Value |
|---|---|
| Total shared cache lines | 2,020 |
| Local HITM | 16,880 |
| Blocked accesses on shared lines | 4,701 |

**16,880 HITM is the highest of all parser tests** and driven entirely by boost queue internal state — consistent with the pipeline RxTx findings. The without-push test shows 4,072 HITM, confirming ~75% of HITM in the dispatch test is boost-sourced.

!!! warning "Optimization Candidate"
    boost::lockfree queue head/tail contention is the dominant false sharing source. A custom SPSC queue with explicit cache line separation would reduce HITM by ~75% based on the without-push comparison. Pre-production redesign candidate.

### Flamegraph

[Parser Dispatch Flamegraph](../assets/flamegraph_parser.svg){target="_blank"}

### Findings & Action Items

| # | Finding | Impact | Status |
|---|---|---|---|
| 1 | P-core IPC 0.46 — memory-bound, boost queue dominates | Expected under queue-push load | ✅ Confirmed |
| 2 | L1-D miss 4.96% — queue writes + ITCH/OUCH struct access | Cache pressure on hot path | ⚠️ Custom queue candidate |
| 3 | dTLB miss 1.45M — boost queue buffer page spans | TLB pressure on every push/pop | ⚠️ Custom queue candidate |
| 4 | 16,880 HITM — boost queue internal state | ~75% boost-sourced per without-push test | ⚠️ Pre-prod redesign |

---

## Results (FIX)

| Field | Value |
|---|---|
| Wall time | 2.80 s |
| User time | 5.30 s |
| Sys time | 0.08 s |
| CPUs utilized | 1.920 |

### Key Metrics

| Metric | Value | Note |
|---|---|---|
| IPC | 1.01 | Compute-bound |
| Branch miss rate | 0.28% | Excellent |
| L1-D miss rate | 2.90% | Moderate |
| LLC miss rate | 0.00% | Negligible |

**FIX is the most compute-friendly protocol.** IPC at 1.01 (vs 0.18–0.20 for ITCH/OUCH) reflects the difference in message structure — FIX messages are tag-value pairs parsed sequentially with simple field extraction. No large struct pointer chasing, no variant dispatch lookup tables.

### Hotspot Analysis

| Overhead | Self | Symbol |
|---|---|---|
| 29.39% | 23.54% | `Parser_Dispatch::parseFIX()` |
| 3.93% | 3.93% | `Parser_FIX::find_type()` |
| — | 2.73% | `Parser_FIX::nextFixMsg()` |

`parseFIX` at 23.54% self is the clear hot function. `find_type` and `nextFixMsg` are secondary.

### Cache & Memory

| Metric | Value | Note |
|---|---|---|
| L1-D miss rate | 2.90% | Moderate — sequential buffer traversal |
| LLC miss rate | 0.00% | Negligible |
| L1-I misses | 208K | Low — but absl::btree visible in icache profile |
| dTLB misses | 30K | Low |
| iTLB misses | 7K | Very low |

**L1-I profile reveals absl::btree at 11.66% overhead.** FIX seqnum validation uses an abseil btree map on putting messages in order. The btree traversal code is large enough to create icache pressure — it appears at 11.66% overhead in the icache record alongside `parseFIX` at 73.24%. Sequential buffer traversal by `nextFixMsg` contributes to L1-D miss.

!!! warning "Optimization Candidate"
    `absl::btree` contributes meaningful icache pressure.

### False Sharing (C2C)

| Metric | Value |
|---|---|
| Total shared cache lines | 544 |
| Local HITM | 4,528 |

4,528 HITM — lower than the dispatch test because only FIX queue is active. Pattern consistent with boost spsc queue internal state.

### Flamegraph

[FIX Parser Flamegraph](../assets/flamegraph_parser_fix.svg){target="_blank"}

### Findings & Action Items

| # | Finding | Impact | Status |
|---|---|---|---|
| 1 | IPC 1.01 — FIX is compute-bound, best cache behavior of all protocols | Protocol characteristic | ✅ No action needed |
| 2 | `absl::btree` — icache pressure at 11.66% | Avoidable overhead on parse hot path | ⚠️ Identified, scheduled |
| 3 | `nextFixMsg` sequential buffer traversal — L1-D miss contributor | Moderate impact | ⚠️ Identified, monitored |

---

## Results (ITCH BIST)

| Field | Value |
|---|---|
| Wall time | 2.39 s |
| User time | 4.55 s |
| Sys time | 0.09 s |
| CPUs utilized | 1.945 |

### Key Metrics

| Metric | Value | Note |
|---|---|---|
| IPC | 0.19 | Severely memory-bound |
| Branch miss rate | 2.37% | Elevated — indirect call in handler lookup table |
| L1-D miss rate | 10.32% | High — SPSC queue cache line contention |
| LLC miss rate | 0.00% | Working set in LLC |

**IPC at 0.19 — the lowest P-core IPC of all parser tests.** The P-core is stalling on SPSC queue cache line contention, not parse logic. The message handler lookup table (`makeMessageHandlersLookup` lambda at 4.67% self in dcache) adds an indirection layer visible in the hotspot profile.

**Branch miss at 2.37%** is 8× higher than FIX — reflecting the indirect call in the message handler lookup table. Mixed message type stream prevents the branch predictor from learning the call target.

### Hotspot Analysis

| Overhead | Self | Symbol |
|---|---|---|
| 26.35% | 26.35% | `Parser_Dispatch::parseITCH_BIST()` |
| 4.67% | 4.67% | `makeMessageHandlersLookup` lambda |

`parseITCH_BIST` dominates with 26.35% self. The message handler lambda at 4.67% self shows the handler dispatch cost.

### Cache & Memory

| Metric | Value | Note |
|---|---|---|
| L1-D miss rate | 10.32% | High — SPSC queue cache line contention |
| LLC miss rate | 0.00% | Working set fits |
| L1-I misses | 44K | Very low — icache not stressed |
| dTLB misses | 1.4K | Very low |

**L1-D miss at 10.32%** is 3.6× FIX. It does not originate from parser logic itself. perf c2c identified 2,243 Local HITM events across 298 shared cache lines, indicating that the producer and consumer are contending on the same SPSC queue cache lines. Each push invalidates the cache line on the producer side, forcing an L1 miss on the next access.

### False Sharing (C2C)

| Metric | Value |
|---|---|
| Local HITM | 2,243 |
| Total shared cache lines | 298 |

Lowest HITM of all queue-based tests — ITCH messages are smaller on average, less queue write pressure per message.

### Flamegraph

[ITCH BIST Flamegraph](../assets/flamegraph_parser_itch_bist.svg){target="_blank"}

### Findings & Action Items

| # | Finding | Impact | Status |
|---|---|---|---|
| 1 | IPC 0.19 — severely memory-bound, queue contention dominant | High L1-D miss rate | ⚠️ Identified, pre-prod review |
| 2 | Branch miss 2.37% — indirect call in message handler lookup table | Mixed message type stream prevents branch predictor from learning call target | ⚠️ Identified, monitored |
| 3 | Handler lookup lambda 4.67% self | Indirection cost per message | ⚠️ Identified, monitored |

---

## Results (ITCH NASDAQ)

| Field | Value |
|---|---|
| Wall time | 2.33 s |
| User time | 4.44 s |
| Sys time | 0.09 s |
| CPUs utilized | 1.946 |

### Key Metrics

| Metric | Value | Note |
|---|---|---|
| IPC | 0.20 | Near-identical to ITCH BIST |
| Branch miss rate | 2.56% | Marginally higher than BIST |
| L1-D miss rate | 10.27% | Near-identical to ITCH BIST |
| LLC miss rate | 0.00% | — |

**ITCH NASDAQ and ITCH BIST are near-identical in all metrics** — same protocol family, similar message struct layout and handler dispatch pattern. NASDAQ has marginally more branch misses (2.56% vs 2.37%) reflecting a slightly different message type distribution.

### Hotspot Analysis

| Overhead | Self | Symbol |
|---|---|---|
| 24.55% | 24.55% | `Parser_Dispatch::parseITCH_NASDAQ()` |
| 7.48% | 7.48% | `makeMessageHandlersLookup` lambda (dcache) |

Handler lookup at 7.48% dcache self is higher than BIST (4.67%) — NASDAQ has more message types in its handler table.

### Cache & Memory

| Metric | Value | Note |
|---|---|---|
| L1-D miss rate | 10.27% | Near-identical to ITCH BIST |
| L1-I misses | 54K | Slightly higher than BIST (44K) — more message types |
| dTLB misses | 1.4K | Very low |

### False Sharing (C2C)

| Metric | Value |
|---|---|
| Local HITM | 2,860 |
| Total shared cache lines | 370 |

Slightly higher than BIST — consistent with larger handler table.

### Flamegraph

[ITCH NASDAQ Flamegraph](../assets/flamegraph_parser_itch_nasdaq.svg){target="_blank"}

### Findings & Action Items

Findings mirror ITCH BIST. No independent optimization candidates beyond those identified for ITCH BIST.

---

## Results (OUCH BIST)

| Field | Value |
|---|---|
| Wall time | 5.59 s |
| User time | 16.29 s |
| Sys time | 0.08 s |
| CPUs utilized | 2.928 |

### Key Metrics

| Metric | Value | Note |
|---|---|---|
| IPC | 0.18 | Worst P-core IPC of all tests |
| Branch miss rate | 1.59% | High |
| L1-D miss rate | 11.36% | Highest of all tests |
| LLC miss rate | 0.00% | — |

**OUCH BIST has the worst cache behavior of all protocols.** IPC 0.18 and L1-D miss at 11.36% and 9,265 Local HITM are the highest per-protocol values — driven by queue contention across the additional thread involved in OUCH packet handling. Wall time is significantly longer (5.59s vs ~2.4s for ITCH), consistent with higher per-message CPU cost.

### Hotspot Analysis

| Overhead | Self | Symbol | Profile |
|---|---|---|---|
| 28.28% | 28.28% | `Parser_Dispatch::parseOUCH_BIST()` | dcache |
| 6.32% | 6.32% | `Parser_Dispatch::dispatch()` | dcache |
| 13.79% | 13.79% | `main::{lambda()#1}` | icache |

`dispatch()` at 6.32% dcache self is notably higher than other protocols — OUCH dispatch involves more decision branching per packet.

### Cache & Memory

| Metric | Value | Note |
|---|---|---|
| L1-D miss rate | 11.36% | Highest — queue contention across additional thread |
| LLC miss rate | 0.00% | — |
| L1-I misses | 247K | Elevated — complex dispatch path |
| dTLB misses | 36.6K | Moderate |
| iTLB misses | 19.3K | Moderate |

### False Sharing (C2C)

| Metric | Value |
|---|---|
| Local HITM | 9,265 |
| Total shared cache lines | 852 |

Highest HITM of the per-protocol tests — driven by queue contention across the additional thread in OUCH packet handling.

### Flamegraph

[OUCH BIST Flamegraph](../assets/flamegraph_parser_ouch_bist.svg){target="_blank"}

### Findings & Action Items

| # | Finding | Impact | Status |
|---|---|---|---|
| 1 | IPC 0.18 + L1-D miss 11.36% — worst cache behavior | Queue contention across additional thread drives memory stalls | ⚠️ Identified, pre-prod review with custom queue |
| 2 | `dispatch()` 6.32% dcache self — highest of all protocols | OUCH dispatch branching cost | ⚠️ Identified, monitored |
| 3 | 9,265 HITM — highest per-protocol HITM | Queue contention across additional thread | ⚠️ Identified, monitored |

---

## Results (OUCH NASDAQ) {#results(ouch-nasdaq)}

!!! note "Test Scope"
    This test profiled the OUCH NASDAQ **parse function in isolation** — no dispatch pipeline, no queue, no concurrent threads as it has the identical dispatch path with OUCH BIST. It measures the pure parse function cost, not the full path. Results are not directly comparable to other protocol tests. -> [back to protocol comparison](#protocol-comparison-back)

| Field | Value |
|---|---|
| Wall time | 0.14 s |
| User time | 0.14 s |
| Sys time | 0.003 s |
| CPUs utilized | 0.991 |

### Key Metrics

| Metric | Value | Note |
|---|---|---|
| IPC (P-core) | 2.26 | Compute-bound — dispatch/queue overhead removed |
| Branch miss rate | 0.01% | Excellent |
| L1-D miss rate | 0.30% | Excellent |
| LLC miss rate | 52% relative | Only 116 absolute misses — negligible |
| Local HITM | 0 | Zero false sharing — single thread |

**IPC 2.26 in isolation vs 0.18 in full OUCH BIST dispatch test** — this is the most revealing number in all parser profiling. The parse function itself is highly efficient and compute-bound. The 12× IPC gap between isolated and dispatched execution is entirely attributable to dispatch overhead, boost queue contention, and cross-thread struct access. The parse implementation is not the problem.

### Flamegraph

[OUCH NASDAQ Flamegraph](../assets/flamegraph_parser_ouch_nasdaq.svg){target="_blank"}

---

## Bonus — Queue Push Overhead

Queue push removed from the parser dispatch test. All protocols still parse, but parsed messages are not pushed to the downstream queue. Isolates the pure parser dispatch cost from the queue transfer cost.

### Comparison

| Metric | With Queue Push | Without Queue Push | Delta |
|---|---|---|---|
| Wall time | 10.70 s | 5.67 s | **47% faster** |
| CPUs utilized | 2.970 | 1.959 | One fewer effective thread |
| P-core IPC | 0.46 | 1.12 | **2.4× improvement** |
| Branch miss rate | 0.37% | 0.07% | **5× fewer misses** |
| L1-D miss rate | 4.96% | 1.88% | **2.6× reduction** |
| LLC miss rate | 0.03% | 0.19% | Higher — cold LLC without consumer |
| L1-I misses | 12.3M | 4.1M | 3× fewer |
| dTLB misses | 1,452K | 154K | **9.4× reduction** |
| iTLB misses | 176K | 349K | Higher — see note |
| Local HITM | 16,880 | 4,072 | **4.1× reduction** |

**Every memory metric improves dramatically when the queue is removed.** The IPC improvement from 0.46 to 1.12 confirms the parse logic itself is not memory-bound — the queue push operation is responsible for the majority of cache and TLB pressure in the dispatch test.

**dTLB reduction from 1.45M to 154K (9.4×)** is the most dramatic delta. The boost spsc queue's large internal buffer (1024-element capacity) causes TLB misses on every push as it spans multiple pages. Without the push, data TLB accesses are limited to the packet buffers themselves.

**Hotspot shift without push:** `dispatch()` rises to 10.98% self and `parseFIX` to 8.27% self — previously hidden beneath queue push overhead, now visible. This confirms the queue push was masking parsing cost in the full test.

**LLC miss rate rises slightly (0.03% → 0.19%)** — without a downstream consumer thread keeping parsed data warm in LLC, eviction pressure increases marginally.

!!! warning "Optimization Candidate"
    The queue push operation dominates parser performance — not parsing itself. A custom SPSC queue with cache-line-aligned head/tail and smaller default capacity would reduce dTLB pressure significantly. Pre-production redesign should evaluate this alongside the `boost::lockfree` replacement identified in pipeline profiling.

---

## Bonus — Without FIX

FIX protocol removed from the dispatch test. Only ITCH BIST, ITCH NASDAQ, OUCH BIST, and OUCH NASDAQ handled. Isolates the FIX contribution to the combined dispatch numbers.

### Comparison

| Metric | With FIX | Without FIX | Delta |
|---|---|---|---|
| P-core IPC | 0.46 | 0.18 | **2.6× worse without FIX** |
| L1-D miss rate | 4.96% | 9.17% | **1.85× worse without FIX** |
| Branch miss rate | 0.37% | 1.38% | 3.7× worse |
| dTLB misses | 1,452K | 1,170K | Slightly lower — fewer queue writes |
| Local HITM | 16,880 | 9,823 | Lower — fewer total queues |

**Removing FIX makes the parser significantly worse in IPC and cache behavior** — this is counterintuitive but explained by FIX's superior cache characteristics. FIX messages are compact, sequentially parsed, and cache-friendly (IPC 1.01 in isolation). When FIX is present in the combined test, its cache-friendly access patterns warm up LLC lines and improve overall throughput. Without FIX, only ITCH and OUCH remain — their high L1-D miss rates (10%+) and low IPC (0.18–0.20) dominate the combined result.

!!! note
    The without-push dcache profile shows `NetworkIO::SBTPacketHandler` 
    at 17.55% self time — absent in the with-push test. OUCH packet 
    demultiplexing happens inside the network layer via SBT packet handler 
    before queue push, meaning part of OUCH processing cost lives in 
    NetworkIO rather than the parser. Without FIX in the mix, OUCH 
    dominates and this network-side cost becomes visible. The true OUCH 
    parse cost is therefore split across NetworkIO and Parser_Dispatch — 
    isolating just the parse function (as done in the OUCH NASDAQ parseFunc 
    test) gives the cleaner picture.

---

## Raw Output

### Parser Dispatch RAW

"Perf Stat — Base"
    ```
    P-Core:
      instructions:   65,045,630,018    0.46 insn/cycle
      cycles:        141,524,363,525    4.454 GHz
      branches:       12,566,316,397  395.524 M/sec
      branch-misses:      46,718,163    0.37%
      L1-dcache-loads: 14,452,718,355  454.898 M/sec
      L1-dcache-misses:   716,911,653    4.96%
      LLC-loads:          700,489,108   22.048 M/sec
      LLC-load-misses:        211,094    0.03%

    E-Core:
      instructions:   45,943,156,557    1.24 insn/cycle
      cycles:         37,119,695,992    1.168 GHz
      branches:        8,735,261,713  274.941 M/sec
      branch-misses:     202,319,788    2.32%
      L1-dcache-loads: 11,096,040,541  349.247 M/sec
      LLC-loads:          51,650,397    1.626 M/sec
      LLC-load-misses:     9,701,248   18.78%

    Wall: 10.698s  User: 31.643s  Sys: 0.129s
    ```

"Perf Stat — ICache + TLB"
    ```
    L1-icache-load-misses (P-core):  12,258,535
    dTLB-load-misses (P-core):        1,452,210
    iTLB-load-misses (P-core):          176,489
    ```

"Perf Record — CPU+Branch (≥1%)"
    ```
    66.31%   0.00%  libc/libstdc++ init (startup)
    33.14%  31.31%  main::{lambda()#1}
    33.12%  33.10%  main::{lambda()#2}
    33.07%   9.72%  main
    23.34%   2.78%  Parser_Dispatch::dispatch()
    ```

"Perf Record — DCache (≥1%)"
    ```
    45.88%   4.35%  main
    42.77%   7.09%  Parser_Dispatch::dispatch()
    38.65%  38.48%  main::{lambda()#2}
    13.77%   4.77%  main::{lambda()#1}
    ```

"C2C — False Sharing Summary"
    ```
    Total records:                 202,010
    Load Local HITM:                16,880
    Total Shared Cache Lines:        2,020
    Load HITs on shared lines:      54,341
    Blocked Access on shared lines:  4,701
    Store HITs on shared lines:     26,342
    ```

---

### FIX RAW

"Perf Stat — Base"
    ```
    P-Core:
      instructions:   25,304,195,735    1.01 insn/cycle
      cycles:         25,158,207,493    4.676 GHz
      branches:        5,096,306,044  947.246 M/sec
      branch-misses:      14,363,096    0.28%
      L1-dcache-loads:  4,330,238,782  804.858 M/sec
      L1-dcache-misses:   125,709,788    2.90%
      LLC-loads:          124,638,956   23.167 M/sec
      LLC-load-misses:          1,219    0.00%

    Wall: 2.802s  User: 5.297s  Sys: 0.084s
    ```

"Perf Stat — ICache + TLB"
    ```
    L1-icache-load-misses (P-core):    207,904
    dTLB-load-misses (P-core):          29,984
    iTLB-load-misses (P-core):           7,330
    ```

"Perf Record — CPU+Branch (≥1%)"
    ```
    29.39%  23.54%  Parser_Dispatch::parseFIX()
     3.93%   3.93%  Parser_FIX::find_type()
    ```

"Perf Record — DCache (≥1%)"
    ```
    29.57%  24.77%  Parser_Dispatch::parseFIX()
     2.73%   2.73%  Parser_FIX::nextFixMsg()
    ```

"Perf Record — ICache (≥1%)"
    ```
    73.24%  52.20%  Parser_Dispatch::parseFIX()
    11.66%   6.65%  absl::btree::insert_unique()
     5.46%   5.46%  Parser_FIX::nextFixMsg()
    ```

"C2C — False Sharing Summary"
    ```
    Total records:          32,703
    Load Local HITM:         4,528
    Total Shared Cache Lines:  544
    ```

---

### ITCH BIST RAW

"Perf Stat — Base"
    ```
    P-Core:
      instructions:    4,208,546,130    0.19 insn/cycle
      cycles:         21,644,812,579    4.658 GHz
      branches:          592,149,524  127.427 M/sec
      branch-misses:      14,035,685    2.37%
      L1-dcache-loads:  1,567,856,173  337.393 M/sec
      L1-dcache-misses:   161,794,739   10.32%
      LLC-loads:          161,033,129   34.653 M/sec
      LLC-load-misses:          1,105    0.00%

    Wall: 2.389s  User: 4.553s  Sys: 0.095s
    ```

"Perf Stat — ICache + TLB"
    ```
    L1-icache-load-misses (P-core):     43,696
    dTLB-load-misses (P-core):           1,402
    iTLB-load-misses (P-core):           6,842
    ```

"Perf Record — DCache (≥1%)"
    ```
    26.35%  26.35%  Parser_Dispatch::parseITCH_BIST()
     4.67%   4.67%  makeMessageHandlersLookup lambda
    ```

"C2C — False Sharing Summary"
    ```
    Total records:          27,297
    Load Local HITM:         2,243
    Total Shared Cache Lines:  298
    ```

---

### ITCH NASDAQ RAW

"Perf Stat — Base"
    ```
    P-Core:
      instructions:    4,166,001,149    0.20 insn/cycle
      cycles:         21,031,490,793    4.642 GHz
      branches:          576,694,857  127.284 M/sec
      branch-misses:      14,788,657    2.56%
      L1-dcache-loads:  1,513,360,798  334.019 M/sec
      L1-dcache-misses:   155,437,696   10.27%
      LLC-loads:          154,275,549   34.051 M/sec
      LLC-load-misses:          3,996    0.00%

    Wall: 2.329s  User: 4.443s  Sys: 0.089s
    ```

"Perf Stat — ICache + TLB"
    ```
    L1-icache-load-misses (P-core):     53,545
    dTLB-load-misses (P-core):           1,387
    iTLB-load-misses (P-core):           6,315
    ```

"Perf Record — DCache (≥1%)"
    ```
    20.17%  20.17%  Parser_Dispatch::parseITCH_NASDAQ()
     7.48%   7.48%  makeMessageHandlersLookup lambda
    ```

"C2C — False Sharing Summary"
    ```
    Total records:          27,098
    Load Local HITM:         2,860
    Total Shared Cache Lines:  370
    ```

---

### OUCH BIST RAW

"Perf Stat — Base"
    ```
    P-Core:
      instructions:   13,568,953,879    0.18 insn/cycle
      cycles:         74,000,551,735    4.519 GHz
      branches:        2,303,330,450  140.660 M/sec
      branch-misses:      36,687,703    1.59%
      L1-dcache-loads:  4,556,807,468  278.275 M/sec
      L1-dcache-misses:   517,428,455   11.36%
      LLC-loads:          509,172,076   31.094 M/sec
      LLC-load-misses:          2,058    0.00%

    Wall: 5.593s  User: 16.294s  Sys: 0.083s
    ```

"Perf Stat — ICache + TLB"
    ```
    L1-icache-load-misses (P-core):    247,146
    dTLB-load-misses (P-core):          36,621
    iTLB-load-misses (P-core):          19,313
    ```

"Perf Record — DCache (≥1%)"
    ```
    32.75%  32.75%  main::{lambda()#2}
    28.28%  28.28%  Parser_Dispatch::parseOUCH_BIST()
    ```

"C2C — False Sharing Summary"
    ```
    Total records:          96,436
    Load Local HITM:         9,265
    Total Shared Cache Lines:  852
    ```

---

### OUCH NASDAQ (parseFunc) RAW

"Perf Stat — Base"
    ```
    P-Core:
      instructions:    1,476,634,495    2.26 insn/cycle
      cycles:            653,079,976    4.763 GHz
      branches:          191,601,433    1.397 G/sec
      branch-misses:          15,890    0.01%
      L1-dcache-loads:   604,500,167    4.409 G/sec
      L1-dcache-misses:    1,797,222    0.30%
      LLC-loads:               2,524   18.409 K/sec
      LLC-load-misses:         1,324   52.46%

    Wall: 0.138s  User: 0.135s  Sys: 0.003s
    ```

"C2C — False Sharing Summary"
    ```
    Load Local HITM:         0
    Total Shared Cache Lines: 0
    No false sharing detected — single thread, no queue.
    ```
