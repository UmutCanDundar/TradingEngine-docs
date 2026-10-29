# OrderManager — Perf Analysis

End-to-end profiling of the OrderManager unit across six test scenarios: **OrderManager** (mixed protocol/venue traffic through the full store path), **OrderManager FIX** (BIST FIX message lifecycle in isolation), **OrderManager ITCH BIST** (BIST ITCH market data processing in isolation), **OrderManager ITCH NASDAQ** (NASDAQ ITCH market data processing in isolation), **OrderManager OUCH BIST** (BIST OUCH execution reports in isolation), and **OrderManager OUCH NASDAQ** (NASDAQ OUCH execution reports in isolation).

The Dispatch test is the primary production-representative scenario — it exercises the full `store()` dispatch path across all protocol and venue branches with a 17-message mixed dataset covering FIX, ITCH, and OUCH traffic for both BIST and NASDAQ. The isolated protocol tests each exercise a narrow 3–4 message sequence and are used to attribute per-path costs without cross-protocol interference.

The OrderManager is a deliberately broad orchestration component. It manages order lifecycle across multiple venues, maintains order pools, hash-based lookup maps, pending order tracking, MarketBook updates, and fan-out to downstream risk, strategy, and database queues — all within a strictly single-threaded store loop. The class comment explicitly notes that cache-line level micro-optimisation was intentionally deferred pending profiling; the results below constitute that first profiling baseline.

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

| Metric | Dispatch | FIX | ITCH BIST | ITCH NASDAQ | OUCH BIST | OUCH NASDAQ |
|---|---|---|---|---|---|---|
| Wall time | 13.63 s | 0.59 s | 0.68 s | 0.57 s | 2.70 s | 3.44 s |
| CPUs utilized | 3.996 | 3.894 | 0.983 | 0.979 | 3.936 | 3.949 |
| P-core IPC | 0.35 | 1.19 | 2.63 | 3.78 | 0.42 | 0.35 |
| Branch miss rate | 1.00% | 0.00% | 0.00% | 0.00% | 0.94% | 1.07% |
| L1-D miss rate | 3.48% | 0.02% | 0.01% | 0.01% | 3.03% | 4.08% |
| LLC miss rate | ~0% | ~0% | ~0% | ~0% | ~0% | ~0% |
| Local HITM | 45,310 | 1 | 0 | 0 | 8,400 | 11,270 |

**The pattern is immediately clear:** ITCH paths are compute-efficient and contention-free (IPC 2.6–3.8, L1-D miss ~0%, zero HITM), while OUCH and Dispatch paths show spin-wait-dominated IPC (~0.35–0.42), elevated L1-D miss rates (3–4%), and significant HITM. The HITM is structurally driven by the concurrent consumer threads spinning on the downstream queues while the store thread writes to shared data — not by any false sharing in the application structs themselves.

LLC miss rate is near zero across all scenarios, confirming the OrderManager working set (order pool, hash maps, pending queues) fits within LLC.

---

## Results (OrderManager)

Mixed protocol/venue traffic through `OrderManager::store()`. This is the primary production-representative scenario. The 17-message dataset covers: instrument directory (BIST + NASDAQ), FIX new/partial/cancel (BIST), BIST ITCH add/exec/cancel, NASDAQ ITCH add/exec/delete, BIST OUCH accepted/executed/cancelled, NASDAQ OUCH accepted/executed/cancelled. Three consumer threads drain `store_to_risk`, `store_to_strategy`, and `store_to_db`.

| Field | Value |
|---|---|
| Wall time | 13.63 s |
| User time | 54.46 s |
| Sys time | 0.02 s |
| CPUs utilized | 3.996 |

Four threads active: the store thread (main, CPU 6) and three consumer spin-loops. User time at 54.46s across ~4 CPUs and 13.63s wall time is consistent with all four threads fully loaded for the run duration. Sys time at 0.02s is negligible — the store loop is pure userspace with no kernel syscalls on the hot path.

---

### Key Metrics

| Metric | P-Core | E-Core | Note |
|---|---|---|---|
| Instructions | 79.8B | 59.2B | E-core carries significant consumer spin-loop work |
| Cycles | 226.9B @ 4.17 GHz | 48.2B @ 0.89 GHz | P-core near boost; E-core far below max — spin-wait |
| IPC | 0.35 | 1.23 | See note below |
| Branch miss rate | 1.00% | 1.77% | Elevated — variant dispatch branches |
| L1-D miss rate | 3.48% | — | Elevated — wide working set, cross-thread contention |
| LLC miss rate | 0.00% | 15.27% of 58M | P-core: full LLC-resident; E-core: startup + spin overhead |

**IPC at 0.35 on P-core is memory-bound and spin-wait dominated.** The store thread interleaves hash map lookups, order pool writes, variant dispatch, and five downstream queue pushes per message — all against a large, cross-thread-contested working set. The E-core IPC at 1.23 is higher than P-core despite doing less application work: the consumer spin loops (which land on E-cores as the OS migrates threads) consist of tight `_mm_pause()` + branch sequences that retire cheaply, inflating E-core IPC relative to the heavier store-path work on the P-core.

**Branch miss rate at 1.00%** is elevated relative to the isolated ITCH tests (0.00%) and reflects the `std::visit` / variant dispatch across the 17-message type mix. Each `store()` call dispatches through nested `std::visit` chains covering up to 14 BIST ITCH + 11 NASDAQ ITCH + 5 BIST OUCH + 11 NASDAQ OUCH + FIX message alternatives. The misprediction rate is not an application defect — it is inherent to the branch-per-variant-arm dispatch pattern across diverse message sequences.

**L1-D miss rate at 3.48%** reflects the combined pressure of the large order pool and hash map working set plus cross-thread cache line invalidation from the three consumer threads reading the same `Order*` pointers written by the store thread to the downstream queues. The LLC miss rate at 0.00% confirms the full working set is LLC-resident — the misses are L1-evictions caused by contention, not true capacity pressure.

---

### Hotspot Analysis

The store thread runs on the P-core. The three consumer lambdas (spin loops) each accumulate ~25% CPU self time in the CPU+Branch profile — they are the dominant inclusive consumers because `_mm_pause()` keeps them resident. The actionable signal is in the application frames below them.

#### P-Core (CPU+Branch)

| Overhead | Self | Symbol |
|---|---|---|
| ~75% | ~0% | `libc` / `libstdc++` thread infrastructure |
| ~25% | 24.98% | `std::thread` lambda#4 (consumer spin) |
| ~25% | 24.98% | `std::thread` lambda#3 (consumer spin) |
| ~25% | 24.96% | `std::thread` lambda#2 (consumer spin) |
| 24.90% | 0.11% | `main` |
| 21.36% | 2.95% | `OrderManager::store()` |
| 6.93% | 6.93% | `boost::lockfree::ringbuffer_base::push` (SPSC queue push) |
| 2.46% | 0.23% | `std::__do_visit` — NASDAQ OUCH variant dispatch |
| 2.19% | 2.19% | `boost::lockfree::ringbuffer_base<Order*>::push` |
| 1.68% | 0.91% | `std::__do_visit` — NASDAQ ITCH variant dispatch |
| 1.17% | 1.17% | `absl::flat_hash_map::find` (order lookup) |
| 1.10% | 0.06% | `OrderManager::add_awaitingAck_order` |

`OrderManager::store()` at 2.95% self is the dispatch loop itself — reading from `parser_to_store_`, visiting the variant, and delegating to the appropriate `update_order` overload. The `boost::lockfree::ringbuffer_base::push` entries at 6.93% and 2.19% self represent the SPSC queue writes to `parser_to_store_` (the large message queue pushing the 17-message variant) and `store_to_risk`/`store_to_strategy`/`store_to_db` respectively. These are structurally necessary and not reducible.

The consumer lambda self-time at ~25% each is a test harness artifact — the spin loops are present to drain queues without blocking the store thread, but they consume CPU samples that inflate inclusive overhead throughout the call tree.

#### ICache Profile

| Overhead | Self | Symbol |
|---|---|---|
| 83.72% | 2.81% | `OrderManager::store()` |
| 16.99% | 11.02% | `folly::ConcurrentHashMap::eraseImpl` |
| 14.66% | 1.21% | `OrderManager::update_order(FIXMessage*)` |
| 13.91% | 4.75% | `std::__do_visit` — NASDAQ ITCH |
| 13.18% | 13.13% | `HashTables::wyhash_small` |
| 10.40% | 1.20% | `std::__do_visit` — NASDAQ OUCH |
| 8.74% | 7.56% | `std::__gen_vtable_impl` — BIST ITCH dispatch |
| 7.64% | 0.49% | `OrderManager::add_awaitingAck_order` |
| 7.39% | 7.34% | `absl::flat_hash_map::find` |
| 6.74% | 1.01% | `folly::ConcurrentHashMap::emplace` |
| 5.29% | 2.21% | `OrderManager::add_market_order` |
| 5.13% | 5.11% | `pthread_mutex_lock` |

**`HashTables::wyhash_small` at 13.13% icache self is the largest single instruction footprint in application code.** This is the hash function called on FIX execution report ID hashing. It appears on the hot path for every FIX exec report processed through update_order(FIXMessage*).

**`folly::ConcurrentHashMap::eraseImpl` at 11.02% icache self** is the most notable cost attributable to the `awaitingAck_orders_` map. On every OUCH accepted/executed and FIX execution report, `awaitingAck_orders_.erase(client_order_id)` is called to remove the matched pending order. The `eraseImpl` footprint reflects folly's hazard-pointer-based reclamation machinery — each erase triggers a hazard pointer acquisition, bucket traversal, node unlink, and deferred reclamation enqueue. This is a structural cost of using a concurrent hash map on a path that is single-threaded in practice (the store loop never races with a writer on `awaitingAck_orders_` except through `add_awaitingAck_order`, which is also called from the same store thread in the test). **The use of `folly::ConcurrentHashMap` for `awaitingAck_orders_` is the primary icache overhead item on the OUCH and FIX paths.**

**`absl::flat_hash_map::find` at 7.34% icache self** is the order lookup cost on the market and our-order maps. With `market_orders_` and `our_orders_` both using `absl::flat_hash_map<OrderKey, Order*>`, each ITCH and OUCH message that requires an existing order lookup calls this function. The flat hash map has excellent cache characteristics (open addressing, linear probing), so this is expected and not a concern.

**`pthread_mutex_lock` at 5.11% icache self** is folly's internal mutex used by `ConcurrentHashMap` during bucket modification. This confirms that `ConcurrentHashMap`'s internal locking contributes meaningfully to the icache footprint even on the single-threaded store path — hazard pointer operations and mutex acquisition are invoked on every insert/erase regardless of actual concurrent access.

#### DCache Profile

| Overhead | Self | Symbol |
|---|---|---|
| 20.78% | 20.73% | `std::thread` lambda#4 (consumer spin) |
| 15.74% | 15.69% | `std::thread` lambda#3 (consumer spin) |
| 15.08% | 14.99% | `std::thread` lambda#2 (consumer spin) |
| 42.89% | 1.14% | `OrderManager::store()` |
| 13.84% | 13.74% | `boost::lockfree::ringbuffer_base::push` (SPSC, message queue) |
| 8.02% | 4.21% | `folly::ConcurrentHashMap::eraseImpl` |
| 6.56% | 6.54% | `boost::lockfree::ringbuffer_base<Order*>::push` |
| 3.73% | 3.62% | BIST OUCH cancelled handler lambda |

The consumer spin loops account for ~51% of combined dcache self — they repeatedly read the SPSC queue tail pointer written by the store thread, generating a steady stream of L1-D load misses as the cache line is invalidated by the producer write on the other core. This is the cross-thread contention pattern that drives both the 3.48% L1-D miss rate and the 45,310 HITM.

`boost::lockfree::ringbuffer_base::push` at 13.74% dcache self is the large message variant queue push — writing the 17-type variant struct into the SPSC ring buffer generates L1-D misses proportional to the variant size. The `Order*` push at 6.54% is the downstream queue writes to risk/strategy/db.

`folly::ConcurrentHashMap::eraseImpl` at 4.21% dcache self confirms the hazard pointer reclamation path also generates data cache pressure — the hazard pointer table and deferred reclamation queue are accessed on every erase.

---

### Cache & Memory

| Metric | Value | Assessment |
|---|---|---|
| L1-D miss rate (P-core) | 3.48% | Elevated — cross-thread invalidation + wide working set |
| LLC miss rate (P-core) | 0.00% | Excellent — full working set LLC-resident |
| L1-I misses (P-core) | 372,284,365 | Elevated — variant dispatch + folly hazptr instruction footprint |
| dTLB misses (P-core) | 1,495,210 | Moderate — large pool and map address space footprint |
| iTLB misses (P-core) | 132,210 | Low |

The L1-I miss count at 372M is the highest across all OrderManager scenarios and reflects the diverse instruction footprint of the full dispatch path: six `update_order` overloads, five filler classes, two hash map implementations, and the folly hazard pointer machinery all contribute unique instruction pages. The LLC miss rate at 0.00% is the key confirmation that despite high L1 pressure, no working set eviction is occurring — the design's large on-stack allocation strategy (avoiding heap fragmentation) is effective at keeping all data LLC-resident.

---

### False Sharing (C2C)

| Metric | Value |
|---|---|
| Total shared cache lines | 19 |
| Load HITs on shared lines | 112,642 |
| Local HITM | 45,310 |
| LLC hits on shared lines | 81,851 |
| Locked accesses on shared lines | 1 |
| Blocked accesses on shared lines | 1,653 |
| Store HITs on shared lines | 83,941 |

**45,310 Local HITM across 19 shared cache lines is the dominant cross-thread contention signal.** The 19 shared lines correspond to the SPSC queue ring buffer slots and metadata (head/tail pointers) written by the store thread and read by each of the three consumer spin loops. With three consumers each spinning on separate queues (`store_to_risk`, `store_to_strategy`, `store_to_db`), each queue write by the store thread causes an ownership transfer (HITM) as the relevant consumer reads the updated slot. This is structurally equivalent to the pipeline SPSC HITM and is a test harness artifact — in production, the consumer threads are doing meaningful work rather than busy-spinning.

No application struct false sharing is detected. The single locked access is a folly `ConcurrentHashMap` mutex acquisition, not a structural contention point.

---

### Flamegraph

[OrderManager Flamegraph](../assets/flamegraph_order_manager.svg){target="_blank"}

---

### Findings & Action Items { #findings }

| # | Finding | Impact | Status |
|---|---|---|---|
| 1 | IPC 0.35 — spin-wait consumer threads dominate; store-path IPC not directly measurable from this profile | Test harness artifact | ✅ No action needed |
| 2 | `folly::ConcurrentHashMap::eraseImpl` 11.02% icache self + 4.21% dcache self — hazard pointer machinery on every OUCH/FIX ack erase | Primary icache overhead on OUCH/FIX paths | ⚠️ Replace folly::concurrenthashmap with custom map <br>**(handled - see** [Opt-1 in Recent Optimizations Page](../optimization.md/#opt-1)**)** |
| 3 | HashTables::wyhash_small 13.13% icache self — hash function called on FIX execution report ID hashing | Optimization candidate — hash function is swappable | ⚠️ Identified, monitored |
| 4 | `absl::flat_hash_map::find` 7.34% icache self — order lookup on every message | Expected; flat_hash_map is the correct choice here | ✅ No action needed |
| 5 | `pthread_mutex_lock` 5.11% icache self — folly internal mutex on store path | Directly caused by finding 2; resolved by replacing `awaitingAck_orders_` | ⚠️ Resolved by finding 2 |
| 6 | 45,310 HITM on 19 shared lines — SPSC queue hand-off between store and 3 consumer spin loops | Test harness artifact; production consumers do meaningful work | ✅ No action needed |
| 7 | L1-D miss 3.48% — cross-thread invalidation from consumer spin-reads on shared SPSC lines | Test harness artifact; same root as finding 6 | ✅ No action needed |
| 8 | LLC miss 0.00% — full working set LLC-resident despite ~12.6 MB on-stack pool | Confirms pool allocation strategy is effective | ✅ No action needed |
| 9 | Branch miss 1.00% — variant dispatch across 17 message types | Expected for mixed-traffic dispatch; inherent to `std::visit` pattern | ✅ No action needed |

---

## Results (OrderManager FIX)

BIST FIX message lifecycle in isolation — `fix_new`, `fix_partial`, `fix_cancel` pushed through `parser_to_store_` per iteration, 5M iterations. One awaiting-ack order is added before each batch via `add_awaitingAck_order`. Three consumer threads drain downstream queues. FIX is the only path exercised — no ITCH or OUCH messages.

| Field | Value |
|---|---|
| Wall time | 0.59 s |
| User time | 2.29 s |
| Sys time | 0.01 s |
| CPUs utilized | 3.894 |

Four threads active: store thread and three consumers. The FIX path is the shortest of all OrderManager scenarios in wall time, consistent with its compact message set (3 messages vs 17 in Dispatch).

---

### Key Metrics

| Metric | P-Core | E-Core | Note |
|---|---|---|---|
| Instructions | 11.1B | 710M | E-core: startup + minor spin activity |
| Cycles | 9.34B @ 4.08 GHz | 628M @ 0.27 GHz | P-core near boost; E-core essentially idle |
| IPC | 1.19 | 1.13 | P-core: FIX build path is instruction-rich |
| Branch miss rate | 0.00% | 3.00% | P-core: excellent — FIX branch pattern well-predicted |
| L1-D miss rate | 0.02% | — | Excellent — FIX working set narrow and hot in L1 |
| LLC miss rate | 6.14% of 34,628 loads | — | Startup artifact — absolute count (2,127) negligible |

**IPC at 1.19 on P-core confirms the FIX path is compute-bound with a clean working set.** The 0.02% L1-D miss rate and near-zero branch misses indicate the FIX order lifecycle (awaiting-ack lookup → `add_our_order` → `fill_fix_exec_report` → three queue pushes) is executing against a fully L1-resident data set with predictable control flow.

---

### Hotspot Analysis

#### P-Core (CPU+Branch)

The CPU+Branch record shows three consumer spin lambdas at ~25% self each — identical structure to Dispatch. The store-path frames below them:

| Overhead | Self | Symbol |
|---|---|---|
| 24.86% | 0.08% | `main` |
| — | — | (consumer lambdas ~25% each) |

The CPU+Branch profile at this wall time is dominated by the spin loops with no application sub-function breakdown below 1% — all FIX store work is subsumed into the lambda overhead. The DCache and ICache records provide the actionable breakdown.

#### DCache Profile

| Overhead | Self | Symbol |
|---|---|---|
| 94.17% | 1.86% | `main` |
| 73.80% | 40.85% | `main::{lambda()#6}` — FIX store iteration loop |
| 43.35% | 33.58% | `OrderManager::store()` |
| 9.78% | 7.81% | `OrderManager::update_order(FIXMessage*)` |

**`main::{lambda()#6}` at 40.85% dcache self is the FIX iteration lambda** — the per-iteration loop that pushes the 3 FIX messages and calls `store()`. Its high self-time relative to `store()` reflects the `parser_to_store_` queue push cost (writing the FIX message variant into the ring buffer) being attributed to the caller rather than the callee.

`OrderManager::store()` at 33.58% dcache self is the dispatch and `update_order(FIXMessage*)` call chain. `update_order` at 7.81% self carries the FIX exec report processing, symbol lookup (`sympack_symid_.find`), `add_our_order`, and `awaitingAck_orders_.erase` cost.

#### ICache Profile

| Overhead | Self | Symbol |
|---|---|---|
| 72.73% | 5.39% | `main` |
| 66.30% | 7.11% | `main::{lambda()#6}` |
| 30.60% | 4.29% | `OrderManager::add_awaitingAck_order` |
| 28.76% | 6.26% | `OrderManager::store()` |
| 19.67% | 8.99% | `folly::ConcurrentHashMap::emplace` |
| 17.92% | 6.03% | `OrderManager::update_order(FIXMessage*)` |
| 11.38% | 8.97% | `operator new` |

**`folly::ConcurrentHashMap::emplace` at 8.99% icache self** is the `add_awaitingAck_order` path — on every iteration, the test calls `add_awaitingAck_order` which does `awaitingAck_orders_.emplace(order->client_order_id, &order)`. This is the folly insert path including hazard pointer setup, bucket lock, and node allocation. `operator new` at 8.97% icache self is the node allocation within folly's insert — every `emplace` into `ConcurrentHashMap` allocates a heap node, confirmed by the `operator new` footprint. In the FIX isolated test this is particularly visible because the `emplace` / `erase` cycle runs on every iteration with only 3 FIX messages, making the ack map operations a large fraction of total work.

**The combination of `folly::ConcurrentHashMap::emplace` (8.99%) + `add_awaitingAck_order` (4.29%) + `operator new` (8.97%) = ~22% icache self collectively attributable to `awaitingAck_orders_` map operations.** This directly confirms finding 2 from Dispatch: replacing `folly::ConcurrentHashMap` with `custom map on this single-threaded path would eliminate all three contributors.

---

### Cache & Memory

| Metric | Value | Assessment |
|---|---|---|
| L1-D miss rate | 0.02% | Excellent |
| LLC miss rate | ~0% absolute | Startup artifact |
| L1-I misses | 63,828 | Very low — narrow FIX instruction footprint |
| dTLB misses | 1,124 | Excellent |
| iTLB misses | 3,626 | Very low |

All cache and TLB metrics are excellent. The FIX path in isolation has the smallest instruction and data footprint of all OrderManager scenarios — it only exercises the FIX exec report handler, `awaitingAck_orders_`, `our_orders_`, and three queue pushes.

---

### False Sharing (C2C)

| Metric | Value |
|---|---|
| Total shared cache lines | 1 |
| Local HITM | 1 |

Effectively zero false sharing. The single HITM event is a noise-level artifact. The FIX path generates no meaningful cross-thread contention — the consumer threads drain queues that are populated sparsely at this message rate.

---

### Flamegraph

[OrderManager FIX Flamegraph](../assets/flamegraph_order_manager_fix.svg){target="_blank"}

---

### Findings & Action Items

The FIX isolated test confirms and quantifies the `awaitingAck_orders_` finding from Dispatch:

- IPC 1.19 — FIX path is compute-bound and clean in isolation. The Dispatch IPC of 0.35 is entirely driven by the multi-thread spin-wait pattern, not FIX processing cost.
- `folly::ConcurrentHashMap::emplace` + `operator new` ~18% combined icache self — node allocation on every ack map insert is the dominant overhead item. See Dispatch finding 2.
- L1-D miss 0.02% — FIX working set (Order struct, FIX message fields, three queue entries) fits entirely in L1. No data pressure.

---

## Results (OrderManager ITCH BIST)

BIST ITCH market data processing in isolation — `bist_add`, `bist_exec`, `bist_cancel` pushed per iteration through `parser_to_store_`, 5M iterations. No OUCH or FIX messages. Three consumer threads drain downstream queues but accumulate negligible samples — CPUs at ~1.0 confirms the store thread is the sole meaningful worker.

| Field | Value |
|---|---|
| Wall time | 0.68 s |
| User time | 0.67 s |
| Sys time | 0.01 s |
| CPUs utilized | 0.983 |

Single-threaded in practice. User ≈ Wall confirms the store thread is the only active CPU consumer. The consumer threads drain empty queues with near-zero cost.

---

### Key Metrics

| Metric | P-Core | E-Core | Note |
|---|---|---|---|
| Instructions | 8.63B | — (not counted) | E-core idle |
| Cycles | 3.28B @ 4.84 GHz | — | P-core near boost |
| IPC | 2.63 | — | See note below |
| Branch miss rate | 0.00% | — | Excellent |
| L1-D miss rate | 0.01% | — | Exceptional |
| LLC miss rate | 40.73% of 40,466 loads | — | Startup artifact |

**IPC at 2.63 confirms the BIST ITCH path is compute-bound with a hot working set.** The ITCH processing path is a tight loop: `store()` pops one message, visits the BIST ITCH variant, calls the appropriate filler, updates MarketBook, and pushes to three downstream queues. With L1-D miss at 0.01% and branch miss at 0.00%, the CPU executes the ITCH message sequence with near-optimal efficiency. This is the clean baseline for what BIST ITCH processing looks like without cross-protocol interference.

---

### Hotspot Analysis

#### P-Core (CPU+Branch)

| Overhead | Self | Symbol |
|---|---|---|
| 98.66% | 1.58% | `main` |
| 94.44% | 18.20% | `OrderManager::store()` |
| 35.28% | 4.79% | `std::__gen_vtable_impl` — BIST ITCH dispatch (seq 7) |
| 31.41% | 31.37% | `absl::flat_hash_map::find` |
| 22.03% | 3.79% | `std::__gen_vtable_impl` — BIST ITCH dispatch (seq 10) |
| 18.70% | 4.03% | `std::__gen_vtable_impl` — BIST ITCH dispatch (seq 8) |
| 12.74% | 12.74% | `FillerITCH_BIST::fill_itch_add` |
| 8.25% | 5.01% | `MarketBook::delete_order` |
| 7.72% | 4.67% | `MarketBook::add_order` |
| 4.30% | 4.30% | `MarketBook::exec_order` |
| 3.05% | 1.79% | `absl::btree::insert_unique` |
| 1.50% | 1.50% | `absl::btree::erase` |

**`absl::flat_hash_map::find` at 31.37% self is the single largest CPU cost in the ITCH BIST path.** Every ITCH exec, cancel, and delete message requires an order lookup — `get_order_from_market_map(OrderKey{order_id, order_book_id, venue, side})`. With 3 messages per iteration (add, exec, cancel), the exec and cancel both call `find`, and the add path calls `add_market_order` which also touches the map. The flat_hash_map is the correct data structure for this pattern (O(1) average, cache-friendly open addressing), and 31% self at IPC 2.63 indicates the hash map is executing efficiently — this is expected cost, not a defect.

**`OrderManager::store()` at 18.20% self** carries the `parser_to_store_` pop, variant type dispatch preamble, and the outer `std::visit` call overhead. The `__gen_vtable_impl` entries (4.79% + 3.79% + 4.03% = 12.61% combined self) are the vtable dispatch cost for the three BIST ITCH message alternatives encountered in the test sequence (AddOrder, OrderExecuted, OrderDelete correspond to variant indices 7, 10, 8 in the BIST ITCH variant).

**`FillerITCH_BIST::fill_itch_add` at 12.74% self** — the struct-fill for new ITCH add orders is the most expensive filler call in the BIST path. The add message carries the most fields (order_id, price, quantity, side, symbol, venue) and triggers both `add_market_order` and `MarketBook::add_order`. `MarketBook::delete_order` (5.01%), `add_order` (4.67%), and `exec_order` (4.30%) together represent the book maintenance cost.

#### TLB Profile

The TLB record shows `FillerITCH_BIST::fill_itch_add` at 31.34% self and `absl::btree::erase` at 17.31% self as the top TLB consumers. The filler generates TLB pressure by writing to Order struct fields across a pool spanning a large virtual address range. The btree erase (from `MarketBook::delete_order` via the price-level btree) traverses a tree whose nodes span disjoint allocations, generating iTLB pressure on each node access. dTLB at 25,696 absolute is low — acceptable for this working set size.

---

### Cache & Memory

| Metric | Value | Assessment |
|---|---|---|
| L1-D miss rate | 0.01% | Exceptional |
| LLC miss rate | ~0% absolute | Startup artifact |
| L1-I misses | 252,264 | Low |
| dTLB misses | 25,696 | Low |
| iTLB misses | 2,209 | Very low |

**L1-D miss at 0.01% is exceptional** and confirms the BIST ITCH path executes against a fully L1-resident working set. The 3-message sequence (add → exec → cancel for the same order book) repeatedly accesses the same Order slot, map buckets, and MarketBook price-level entry — these stay hot across iterations.

---

### False Sharing (C2C)

| Metric | Value |
|---|---|
| Total shared cache lines | 0 |
| Local HITM | 0 |

Zero false sharing. The ITCH path does not use `awaitingAck_orders_` (no OUCH/FIX messages), so `folly::ConcurrentHashMap` is never accessed. The consumer threads drain an effectively empty queue. No contention possible.

---

### Flamegraph

[OrderManager ITCH BIST Flamegraph](../assets/flamegraph_order_manager_itch_bist.svg){target="_blank"}

---

### Findings & Action Items

| # | Finding | Impact | Status |
|---|---|---|---|
| 1 | IPC 2.63 — ITCH BIST path is compute-bound and well-optimised in isolation | Expected characteristic | ✅ No action needed |
| 2 | `absl::flat_hash_map::find` 31.37% CPU self — order lookup dominant cost | Expected; correct data structure for this pattern | ✅ No action needed |
| 3 | `__gen_vtable_impl` dispatch 12.61% combined CPU self — BIST variant dispatch overhead | Expected for `std::visit` pattern; reducible only via manual dispatch redesign | ✅ No action needed |
| 4 | `FillerITCH_BIST::fill_itch_add` 12.74% CPU self — add message carries most fields | Expected; add path is inherently heaviest per ITCH message | ✅ No action needed |
| 5 | Zero HITM — no `awaitingAck_orders_` access, no cross-thread contention | Confirms ITCH path is contention-free in isolation | ✅ No action needed |
| 6 | L1-D miss 0.01% — full working set L1-resident | Excellent | ✅ No action needed |

---

## Results (OrderManager ITCH NASDAQ)

NASDAQ ITCH market data processing in isolation — `nasdaq_add`, `nasdaq_exec`, `nasdaq_delete` pushed per iteration, 5M iterations. Structurally equivalent to ITCH BIST but with the NASDAQ variant chain and the additional `nq_itch_refnum_ordkey_` map for order reference number → OrderKey translation.

| Field | Value |
|---|---|
| Wall time | 0.57 s |
| User time | 0.55 s |
| Sys time | 0.01 s |
| CPUs utilized | 0.979 |

Single-threaded in practice, slightly faster than ITCH BIST (0.57s vs 0.68s wall). The higher IPC (3.78 vs 2.63) suggests the NASDAQ ITCH sequence has a more compact execution profile — discussed below.

---

### Key Metrics

| Metric | P-Core | E-Core | Note |
|---|---|---|---|
| Instructions | 10.16B | 1.97B | E-core: startup |
| Cycles | 2.69B @ 4.84 GHz | 1.80B @ 3.24 GHz | P-core near boost |
| IPC | 3.78 | 1.09 | Highest IPC across all OrderManager scenarios |
| Branch miss rate | 0.00% | 2.00% | P-core: excellent |
| L1-D miss rate | 0.01% | — | Exceptional |
| LLC miss rate | 11.51% of 68,853 loads | — | Startup artifact |

**IPC at 3.78 is the highest across all OrderManager scenarios.** The NASDAQ ITCH path processes `add → exec → delete` with a single-venue, single-instrument-type sequence that produces a highly predictable branch pattern. The `nq_itch_refnum_ordkey_` map lookup (used for exec and delete to translate `order_ref` to `OrderKey`) adds an additional hash map access vs BIST, but the overall IPC is higher — this is consistent with the NASDAQ message format being slightly more compact and the `std::__do_visit` dispatch for NASDAQ ITCH covering a smaller variant index range in this test than the BIST `__gen_vtable_impl` dispatch.

---

### Hotspot Analysis

#### P-Core (CPU+Branch)

| Overhead | Self | Symbol |
|---|---|---|
| 98.74% | 1.72% | `main` |
| 96.25% | 18.04% | `OrderManager::store()` |
| 68.17% | 11.72% | `std::__do_visit` — NASDAQ ITCH variant dispatch |
| 24.18% | 24.18% | `absl::flat_hash_map::find` |
| 12.78% | 10.72% | `OrderManager::add_market_order` |
| 8.27% | 4.76% | `MarketBook::delete_order` |
| 5.01% | 1.50% | `MarketBook::add_order` |
| 4.84% | 4.84% | `MarketBook::exec_order` |
| 3.64% | 2.15% | `absl::btree::insert_unique` |
| 3.60% | 3.60% | `boost::lockfree::ringbuffer_base<Order*>::push` |
| 2.78% | 2.78% | `FillerITCH_NASDAQ::fill_itch_add` |

**`absl::flat_hash_map::find` at 24.18% self** is again the dominant lookup cost — same pattern as BIST but slightly lower self-time fraction (24% vs 31%), consistent with the higher IPC diluting the relative sample count.

**`OrderManager::add_market_order` at 10.72% self** is more visible here than in BIST — the NASDAQ path allocates a new market order on every `add` message and also writes the `nq_itch_refnum_ordkey_` emplace. This double-write (pool slot + reference map) explains why `add_market_order` is proportionally larger in the NASDAQ profile.

**`std::__do_visit` at 11.72% self** is the NASDAQ ITCH variant dispatch — higher than the BIST per-frame entries but in a single frame rather than three `__gen_vtable_impl` entries. The structural dispatch cost is comparable.

`FillerITCH_NASDAQ::fill_itch_add` at 2.78% self is much lower than the BIST counterpart (12.74%). This is consistent with the NASDAQ add message format being more compact and the filler having fewer field writes.

#### TLB Profile

`OrderManager::store()` at 59.15% and `std::__do_visit` NASDAQ ITCH at 30.53% are the top TLB consumers. `OrderManager::add_market_order` at 22.25% self confirms the pool allocation write (`order_pool_[market_next_slot]`) is the primary source of dTLB pressure — each new order occupies a different pool slot across the large array, generating TLB misses as the pool index advances. dTLB at 9,832 absolute is very low — lower than BIST's 25,696, consistent with the pool allocation pattern for the 3-message NASDAQ sequence touching fewer distinct pool regions.

---

### Cache & Memory

| Metric | Value | vs ITCH BIST |
|---|---|---|
| L1-D miss rate | 0.01% | Identical |
| L1-I misses | 242,215 | Slightly lower than BIST (252,264) |
| dTLB misses | 9,832 | Lower than BIST (25,696) |
| iTLB misses | 1,407 | Lower than BIST (2,209) |

All metrics confirm parity with ITCH BIST within sampling variance. The NASDAQ path is marginally more efficient in TLB terms, consistent with its simpler filler and shorter instruction sequence per message.

---

### False Sharing (C2C)

| Metric | Value |
|---|---|
| Total shared cache lines | 0 |
| Local HITM | 0 |

Zero false sharing. Same rationale as ITCH BIST — no `awaitingAck_orders_` access, consumer threads drain empty queues.

---

### Flamegraph

[OrderManager ITCH NASDAQ Flamegraph](../assets/flamegraph_order_manager_itch_nasdaq.svg){target="_blank"}

---

### Findings & Action Items

| # | Finding | Impact | Status |
|---|---|---|---|
| 1 | IPC 3.78 — highest across all OrderManager scenarios; NASDAQ ITCH path is highly efficient | Expected characteristic | ✅ No action needed |
| 2 | `absl::flat_hash_map::find` 24.18% CPU self — order lookup dominant, lower fraction than BIST due to higher IPC | Expected | ✅ No action needed |
| 3 | `add_market_order` 10.72% CPU self — NASDAQ add path writes both pool slot and reference number map | Expected for NASDAQ's two-map order tracking | ✅ No action needed |
| 4 | Zero HITM, L1-D miss 0.01% — parity with ITCH BIST confirmed | Expected | ✅ No action needed |

---

## Results (OrderManager OUCH BIST)

BIST OUCH execution report processing in isolation — `bist_acc`, `bist_execO`, `bist_canO` pushed per iteration, 5M iterations. One awaiting-ack order added per iteration via `add_awaitingAck_order`. Three consumer threads active. The OUCH path is the first scenario where `awaitingAck_orders_` (folly `ConcurrentHashMap`) is fully exercised on every iteration — insert on ack-add, lookup on accept, erase on cancel/exec.

| Field | Value |
|---|---|
| Wall time | 2.70 s |
| User time | 10.61 s |
| Sys time | 0.01 s |
| CPUs utilized | 3.936 |

Four threads active — same structure as Dispatch. Wall time at 2.70s for 5M iterations (vs 0.68s for ITCH BIST at the same N) immediately signals that the OUCH path is ~4× slower per iteration than the ITCH path.

---

### Key Metrics

| Metric | P-Core | E-Core | Note |
|---|---|---|---|
| Instructions | 18.72B | 9.64B | E-core: significant spin consumer activity |
| Cycles | 44.06B @ 4.15 GHz | 9.28B @ 0.87 GHz | P-core near boost; E-core spin-wait clock |
| IPC | 0.42 | 1.04 | P-core memory/contention-bound |
| Branch miss rate | 0.94% | 3.18% | P-core elevated — OUCH variant dispatch |
| L1-D miss rate | 3.03% | — | Elevated — folly hazptr + SPSC contention |
| LLC miss rate | 0.01% | 15.22% | P-core: LLC-resident; E-core startup noise |

**IPC at 0.42 on P-core is structurally similar to Dispatch (0.35) and OUCH NASDAQ (0.35)** — the spin consumer threads and folly `ConcurrentHashMap` overhead together depress the observed IPC regardless of the actual store-path work. The OUCH processing itself is heavier than ITCH (map insert + hazptr emplace + map lookup + hazptr erase on every accepted message) which compounds the IPC depression.

**L1-D miss rate at 3.03%** reflects both cross-thread SPSC invalidation and the folly hazard pointer table accesses (hazard pointer slots are shared across threads, generating additional cache line bouncing on every `ConcurrentHashMap` operation).

**L1-I misses at 3.99M** is dramatically elevated relative to ITCH BIST (252K). This 16× increase is attributable almost entirely to the folly hazard pointer machinery — `hazptr_domain::push_list`, `hazptr_inline_executor_add`, `hazptr_obj_cohort::reclaim_list` — which brings in multiple instruction pages on every `ConcurrentHashMap` insert and erase.

---

### Hotspot Analysis

#### P-Core (CPU+Branch)

| Overhead | Self | Symbol |
|---|---|---|
| ~75% | ~25% each | Three consumer spin lambdas |
| 15.18% | 2.57% | `OrderManager::store()` |
| 6.27% | 6.19% | BIST OUCH cancelled message handler lambda |
| 4.77% | 4.77% | `boost::lockfree::ringbuffer_base::push` (SPSC) |
| 3.18% | 0.83% | `folly::ConcurrentHashMap::eraseImpl` |
| 2.30% | 2.30% | `boost::lockfree::ringbuffer_base<Order*>::push` |
| 2.17% | 0.07% | `OrderManager::add_awaitingAck_order` |
| 2.04% | 0.19% | `folly::ConcurrentHashMap::emplace` |
| 1.77% | 1.77% | `pthread_mutex_lock` |

**The BIST OUCH cancelled message handler at 6.19% CPU self is the largest application leaf in this profile.** The cancelled handler (`OUCHOrderCancelledMessage`) is the most complex OUCH response: it calls `get_order_from_our_map` (hash map lookup by OrderKey), `fill_ouch_cancelled`, and pushes to three downstream queues plus the db queue. Its prominence as a leaf — no further callee cost attributed to it — confirms the cancel processing instructions are dense and in-lined.

`folly::ConcurrentHashMap::eraseImpl` at 0.83% CPU self and `emplace` at 0.19% appear modest in the CPU+Branch profile but are the dominant icache contributors (see below) — the hazard pointer overhead is instruction-count heavy but not instruction-retire heavy, causing it to show up more strongly in icache than CPU profiles.

`pthread_mutex_lock` at 1.77% self confirms folly's internal bucket mutex is being acquired on the store thread on every OUCH accepted/cancel processing. This is structurally unavoidable with `ConcurrentHashMap` regardless of actual concurrent access.

#### ICache Profile

| Overhead | Self | Symbol |
|---|---|---|
| 46.13% | 1.80% | `OrderManager::store()` |
| 37.92% | 1.01% | `asm_sysvec_apic_timer_interrupt` (kernel timer) |
| 32.49% | 3.54% | `folly::ConcurrentHashMap::eraseImpl` |
| 22.32% | 1.00% | `folly::hazptr_domain::push_list` |
| 20.06% | 0.16% | `folly::detail::hazptr_inline_executor_add` |

**`folly::ConcurrentHashMap::eraseImpl` at 3.54% icache self combined with `hazptr_domain::push_list` (1.00%) and `hazptr_inline_executor_add` (0.16%)** — together 4.70% of icache self — confirm that the folly hazard pointer reclamation machinery is the dominant instruction footprint cost on the OUCH path. Each `eraseImpl` call traverses the bucket list, unlinks the node, and enqueues it for deferred reclamation through the hazard pointer domain. The kernel timer interrupt at 1.01% icache self reflects the elevated timer interrupt rate at 2.70s wall time — not an application concern.

---

### Cache & Memory

| Metric | Value | Assessment |
|---|---|---|
| L1-D miss rate | 3.03% | Elevated — SPSC contention + folly hazptr table access |
| LLC miss rate | 0.01% | Excellent — LLC-resident |
| L1-I misses | 3,987,816 | Elevated — folly hazptr machinery dominant |
| dTLB misses | 102,950 | Moderate — `ConcurrentHashMap` node allocation |
| iTLB misses | 144,293 | Elevated — folly hazptr instruction pages |

**L1-I misses at 3.99M** vs ITCH BIST's 252K is a 16× increase driven by the folly hazard pointer infrastructure. iTLB at 144K vs ITCH BIST's 2.2K (66×) confirms that the folly hazptr and reclamation code pages span multiple iTLB entries that are not cached between calls. The `operator new` / `cfree` calls within `ConcurrentHashMap` also contribute to dTLB pressure (102K vs ITCH BIST's 25K) through heap allocator metadata accesses.

---

### False Sharing (C2C)

| Metric | Value |
|---|---|
| Total shared cache lines | 11 |
| Load HITs on shared lines | 22,948 |
| Local HITM | 8,400 |
| Blocked accesses on shared lines | 275 |
| Store HITs on shared lines | 17,462 |

**8,400 Local HITM across 11 shared cache lines.** The 11 lines include the SPSC queue ring buffers (same as Dispatch) and additional lines within the `ConcurrentHashMap` internal structure — the hazard pointer table slots and the bucket lock words are written by the store thread during insert/erase and read by the folly background reclamation path (which may run on a different core). Unlike the Dispatch HITM which is purely SPSC hand-off, the OUCH HITM has a structural contribution from folly's internal state, making it partially non-test-harness in character.

---

### Flamegraph

[OrderManager OUCH BIST Flamegraph](../assets/flamegraph_order_manager_ouch_bist.svg){target="_blank"}

---

### Findings & Action Items

| # | Finding | Impact | Status |
|---|---|---|---|
| 1 | IPC 0.42 — spin consumer threads + folly overhead depress IPC; OUCH path ~4× slower per iteration than ITCH | `folly::ConcurrentHashMap` is the primary cause | ⚠️ See finding 2 |
| 2 | `folly::ConcurrentHashMap` emplace + eraseImpl + hazptr + mutex on every OUCH accepted/cancel — 4.70% icache self, L1-I 3.99M (16× ITCH baseline) | Per-iteration heap allocation + hazard pointer reclamation on a single-threaded path | ⚠️ Replace folly::concurrenthashmap with custom map <br>**(handled - see Opt-1 in Recent Optimizations Page)** |
| 3 | BIST OUCH cancelled handler 6.19% CPU self — largest application leaf | Expected; cancelled is the most field-rich OUCH response | ✅ No action needed |
| 4 | 8,400 HITM across 11 lines — SPSC queues + folly ConcurrentHashMap internal state | Partially structural (folly internal), partially test harness | ⚠️ Partially resolved by finding 2 |
| 5 | dTLB 102,950 / iTLB 144,293 — folly hazptr instruction pages + node heap allocation | Directly caused by finding 2 | ⚠️ Resolved by finding 2 |

---

## Results (OrderManager OUCH NASDAQ)

NASDAQ OUCH execution report processing in isolation — `nasdaq_acc`, `nasdaq_execO`, `nasdaq_canO` pushed per iteration, 5M iterations. Structurally equivalent to OUCH BIST but with the NASDAQ OUCH variant chain and the additional `nq_ouch_refnum_ordkey_` map. The NASDAQ OUCH path is the most complex per-message: it requires `awaitingAck_orders_` lookup, `add_our_order` (pool slot + `our_orders_` insert + `orderhistory` push), `nq_ouch_refnum_ordkey_` insert/erase, and `sympack_symid_` lookup on replaced/non-accepted messages.

| Field | Value |
|---|---|
| Wall time | 3.44 s |
| User time | 13.59 s |
| Sys time | 0.01 s |
| CPUs utilized | 3.949 |

Four threads active. Wall time at 3.44s vs OUCH BIST's 2.70s — NASDAQ OUCH is ~27% slower per iteration, attributable to the additional `nq_ouch_refnum_ordkey_` and `sympack_symid_` map operations on the exec/cancel paths.

---

### Key Metrics

| Metric | P-Core | E-Core | Note |
|---|---|---|---|
| Instructions | 19.65B | 11.53B | E-core: consumer spin activity |
| Cycles | 56.53B @ 4.16 GHz | 10.70B @ 0.79 GHz | P-core near boost; E-core spin-wait |
| IPC | 0.35 | 1.08 | Lowest P-core IPC across all scenarios |
| Branch miss rate | 1.07% | 1.96% | Elevated — NASDAQ OUCH variant has more alternatives |
| L1-D miss rate | 4.08% | — | Highest across all OrderManager scenarios |
| LLC miss rate | 0.00% | 13.72% | P-core: LLC-resident |

**IPC at 0.35 matches the Dispatch figure** — the NASDAQ OUCH path is effectively as contention-bound as the full mixed-traffic run. **L1-D miss rate at 4.08% is the highest across all scenarios**, reflecting the additional map accesses (5 maps potentially touched per iteration: `awaitingAck_orders_`, `our_orders_`, `our_orders_wtokenkey_`, `nq_ouch_refnum_ordkey_`, `sympack_symid_`) combined with SPSC contention. All of these are contiguous with the folly hazptr access pattern on every `ConcurrentHashMap` operation.

**L1-I misses at 4.23M** slightly exceeds OUCH BIST (3.99M), consistent with the NASDAQ OUCH variant having more alternative message types (11 vs 5 in BIST OUCH) and the additional map access instruction sequences.

---

### Hotspot Analysis

#### P-Core (CPU+Branch)

The CPU+Branch perf record at this wall time is dominated by `ld-linux` dynamic linker frames — the NASDAQ OUCH binary has a larger dynamic linking footprint and the profiler accumulates insufficient application-level samples to produce a clean breakdown above 1% for store-path functions. This is a sampling artifact at the ~35% P-core IPC level where the store thread spends more cycles stalled than retiring, making it harder for the sampler to catch it in application code.

#### DCache Profile

| Overhead | Self | Symbol |
|---|---|---|
| 44.86% | 1.69% | `OrderManager::store()` |
| 20.76% | 20.70% | `std::thread` lambda#4 (consumer spin) |
| 17.66% | 1.51% | `std::__do_visit` — NASDAQ OUCH variant dispatch |
| 16.86% | 16.79% | `std::thread` lambda#2 (consumer spin) |
| 16.54% | 16.49% | `std::thread` lambda#3 (consumer spin) |
| 15.53% | 15.42% | `boost::lockfree::ringbuffer_base::push` (SPSC) |
| 14.26% | 9.00% | `folly::ConcurrentHashMap::eraseImpl` |
| 8.98% | 8.95% | `boost::lockfree::ringbuffer_base<Order*>::push` |

**`folly::ConcurrentHashMap::eraseImpl` at 9.00% dcache self** is substantially higher here than in OUCH BIST (4.21% in Dispatch, not separately visible in OUCH BIST dcache). The NASDAQ OUCH cancel path calls `awaitingAck_orders_.erase` and additionally touches `nq_ouch_refnum_ordkey_` — the combined map erase cost is higher per iteration. The three consumer lambdas at ~53% combined dcache self confirm the SPSC contention pattern is identical to OUCH BIST.

#### TLB Profile

| Overhead | Self | Symbol |
|---|---|---|
| 65.76% | 30.53% | `std::__do_visit` — NASDAQ OUCH variant dispatch |
| 22.98% | 2.03% | `folly::ConcurrentHashMap::eraseImpl` |
| 21.24% | 21.13% | `boost::lockfree::ringbuffer_base::push` (SPSC) |
| 12.24% | 0.18% | `std::_Deque_base` — folly hazptr deque init |
| 10.51% | 10.51% | `OrderManager::add_our_order` |
| 7.72% | 7.72% | `pthread_mutex_lock` |

**`std::__do_visit` NASDAQ OUCH at 30.53% TLB self** is striking — the NASDAQ OUCH variant dispatch (`std::variant<OUCHSystemEvent*, OUCHOrderAccepted*, ..., OUCHAccountQueryResponse*>` with 11 alternatives) generates significant iTLB pressure because the vtable for 11 alternatives spans more instruction pages than the 5-alternative BIST OUCH variant. This is the primary source of iTLB misses (32,321) vs OUCH BIST (144,293 — note BIST is higher in absolute terms due to the hazptr reclamation page pressure being more dominant there).

**`pthread_mutex_lock` at 7.72% TLB self** confirms the folly internal mutex is generating TLB pressure through its internal state accesses — consistent with OUCH BIST findings.

**`OrderManager::add_our_order` at 10.51% TLB self** — the accepted OUCH message calls `add_our_order` which writes to the order pool slot, `our_orders_` map, and `orderhistory` — spanning three distinct memory regions and generating dTLB misses as pool and map addresses diverge across iterations.

---

### Cache & Memory

| Metric | Value | vs OUCH BIST |
|---|---|---|
| L1-D miss rate | 4.08% | Higher (3.03%) — more map accesses per iteration |
| LLC miss rate | 0.00% | Identical |
| L1-I misses | 4,226,950 | Slightly higher (3,987,816) |
| dTLB misses | 244,587 | Much higher (102,950) — additional NASDAQ maps |
| iTLB misses | 32,321 | Lower (144,293) — BIST hazptr pages dominate BIST iTLB |

The dTLB difference (244K vs 103K) is the most notable delta between the two OUCH scenarios. NASDAQ OUCH touches `nq_ouch_refnum_ordkey_` (additional `absl::flat_hash_map`) and `sympack_symid_` on non-accepted messages, adding two more map regions to the dTLB working set. The L1-D miss rate increase from 3.03% to 4.08% is consistent with this wider data footprint.

---

### False Sharing (C2C)

| Metric | Value |
|---|---|
| Total shared cache lines | 13 |
| Local HITM | 11,270 |
| Blocked accesses on shared lines | 389 |
| Store HITs on shared lines | 21,383 |

**11,270 HITM across 13 shared cache lines** — higher than OUCH BIST (8,400 / 11 lines). The additional 2 shared lines likely correspond to `nq_ouch_refnum_ordkey_` bucket metadata and an additional `our_orders_` map control line that the NASDAQ OUCH accepted path writes and the consumer threads transitively read through the order pointer chain. The structural pattern is the same as OUCH BIST — SPSC queues + folly internal state + the NASDAQ OUCH path's additional map writes increase both the shared line count and HITM intensity.

---

### Flamegraph

[OrderManager OUCH NASDAQ Flamegraph](../assets/flamegraph_order_manager_ouch_nasdaq.svg){target="_blank"}

---

### Findings & Action Items

| # | Finding | Impact | Status |
|---|---|---|---|
| 1 | IPC 0.35, L1-D miss 4.08% — lowest IPC and highest L1-D miss across all scenarios | `folly::ConcurrentHashMap` + wide map footprint | ⚠️ See finding 2 |
| 2 | `folly::ConcurrentHashMap::eraseImpl` 9.00% dcache self — highest erase cost across all OUCH scenarios; NASDAQ OUCH calls erase more per iteration than BIST OUCH | Direct consequence of using concurrent map on single-threaded path | ⚠️ Replace folly::concurrenthashmap with custom map <br>**(handled - see Opt-1 in Recent Optimizations Page)** |
| 3 | dTLB 244,587 — 2.4× OUCH BIST (102K); additional `nq_ouch_refnum_ordkey_` and `sympack_symid_` map regions widen TLB footprint | Expected for NASDAQ's two additional maps; not reducible without redesign | ✅ No action needed |
| 4 | `std::__do_visit` NASDAQ OUCH 30.53% TLB self — 11-alternative variant generates more iTLB pressure than 5-alternative BIST OUCH | Inherent to the NASDAQ OUCH message type count | ✅ No action needed |
| 5 | `pthread_mutex_lock` 7.72% TLB self — folly internal mutex on single-threaded path | Resolved by finding 2 | ⚠️ Resolved by finding 2 |
| 6 | 11,270 HITM across 13 shared lines — SPSC + folly internal state + additional NASDAQ OUCH map writes | Partially resolved by finding 2 (folly lines eliminated) | ⚠️ Partially resolved by finding 2 |
| 7 | LLC miss 0.00% — despite 5 map accesses per iteration, full working set LLC-resident | Pool allocation strategy effective | ✅ No action needed |

---

## Raw Output

### OrderManager RAW

```
task-clock:         54,476,106,755        3.996 CPUs utilized
context-switches:              213        3.910 /sec
cpu-migrations:                  4        0.073 /sec
page-faults:                   788       14.465 /sec

P-Core (cpu_core):
  instructions:    79,835,684,978        0.35  insn/cycle
  cycles:         226,953,465,798        4.166 GHz
  branches:        15,370,733,928      282.156 M/sec
  branch-misses:      153,192,457        1.00%
  L1-dcache-loads:  22,850,915,233     419.467 M/sec
  L1-dcache-misses:    794,465,469        3.48%
  LLC-loads:           770,076,933       14.136 M/sec
  LLC-load-misses:          16,126        0.00%

E-Core (cpu_atom):
  instructions:    59,232,384,866        1.23  insn/cycle
  cycles:          48,211,009,369        0.885 GHz
  branches:        11,126,888,137      204.253 M/sec
  branch-misses:      196,575,914        1.77%
  L1-dcache-loads:  14,421,291,910      264.727 M/sec
  LLC-loads:           58,287,178        1.070 M/sec
  LLC-load-misses:       8,901,224       15.27%

Wall: 13.63s  User: 54.46s  Sys: 0.02s
```

```
P-Core L1-icache-load-misses:  372,284,365
P-Core dTLB-load-misses:        1,495,210
P-Core iTLB-load-misses:          132,210
```

CPU+Branch top application frames:
```
21.36%  2.95%  order_manager_perf  OrderManager::store()
 6.93%  6.93%  order_manager_perf  boost::lockfree::ringbuffer_base::push (SPSC message queue)
 2.19%  2.19%  order_manager_perf  boost::lockfree::ringbuffer_base<Order*>::push
 1.17%  1.17%  order_manager_perf  absl::flat_hash_map::find (order lookup)
 1.10%  0.06%  order_manager_perf  OrderManager::add_awaitingAck_order
```

ICache top application frames:
```
83.72%  2.81%  order_manager_perf  OrderManager::store()
16.99% 11.02%  order_manager_perf  folly::ConcurrentHashMap::eraseImpl
13.18% 13.13%  order_manager_perf  HashTables::wyhash_small
 7.39%  7.34%  order_manager_perf  absl::flat_hash_map::find
 5.13%  5.11%  libc.so.6           pthread_mutex_lock
 5.29%  2.21%  order_manager_perf  OrderManager::add_market_order
```

DCache top application frames:
```
42.89%  1.14%  order_manager_perf  OrderManager::store()
13.84% 13.74%  order_manager_perf  boost::lockfree::ringbuffer_base::push
 8.02%  4.21%  order_manager_perf  folly::ConcurrentHashMap::eraseImpl
 6.56%  6.54%  order_manager_perf  boost::lockfree::ringbuffer_base<Order*>::push
```

C2C:
```
Total Shared Cache Lines:  19
Load HITs on shared lines: 112,642
Load Local HITM:           45,310
Blocked Access on shared:   1,653
Store HITs on shared:      83,941
```

---

### OrderManager FIX RAW

```
task-clock:          2,291,169,359        3.894 CPUs utilized
page-faults:                   759      331.272 /sec

P-Core (cpu_core):
  instructions:    11,097,425,752        1.19  insn/cycle
  cycles:           9,340,793,201        4.077 GHz
  branches:         1,950,050,405      851.116 M/sec
  branch-misses:           17,613        0.00%
  L1-dcache-loads:  2,836,739,377        1.238 G/sec
  L1-dcache-misses:       685,883        0.02%
  LLC-loads:               34,628       15.114 K/sec
  LLC-load-misses:          2,127        6.14%

Wall: 0.588s  User: 2.288s  Sys: 0.005s
```

```
P-Core L1-icache-load-misses:     63,828
P-Core dTLB-load-misses:           1,124
P-Core iTLB-load-misses:           3,626
```

DCache top frames:
```
73.80% 40.85%  order_manager_fix_perf  main::{lambda()#6} (FIX iteration loop)
43.35% 33.58%  order_manager_fix_perf  OrderManager::store()
 9.78%  7.81%  order_manager_fix_perf  OrderManager::update_order(FIXMessage*)
```

ICache top frames:
```
66.30%  7.11%  order_manager_fix_perf  main::{lambda()#6}
30.60%  4.29%  order_manager_fix_perf  OrderManager::add_awaitingAck_order
28.76%  6.26%  order_manager_fix_perf  OrderManager::store()
19.67%  8.99%  order_manager_fix_perf  folly::ConcurrentHashMap::emplace
17.92%  6.03%  order_manager_fix_perf  OrderManager::update_order(FIXMessage*)
11.38%  8.97%  libstdc++.so.6.0.33     operator new
```

C2C:
```
Total Shared Cache Lines:  1
Load Local HITM:           1
```

---

### OrderManager ITCH BIST RAW

```
task-clock:            678,118,647        0.983 CPUs utilized
page-faults:                   760        1.121 K/sec

P-Core (cpu_core):
  instructions:     8,634,286,667        2.63  insn/cycle
  cycles:           3,280,481,993        4.838 GHz
  branches:         1,430,678,024        2.110 G/sec
  branch-misses:           67,114        0.00%
  L1-dcache-loads:  2,398,842,012        3.537 G/sec
  L1-dcache-misses:       178,664        0.01%
  LLC-loads:               40,466       59.674 K/sec
  LLC-load-misses:         16,482       40.73%

Wall: 0.678s  User: 0.674s  Sys: 0.005s
```

```
P-Core L1-icache-load-misses:   252,264
P-Core dTLB-load-misses:         25,696
P-Core iTLB-load-misses:          2,209
```

CPU+Branch top frames:
```
94.44% 18.20%  order_manager_itch_bist_perf  OrderManager::store()
31.41% 31.37%  order_manager_itch_bist_perf  absl::flat_hash_map::find
12.74% 12.74%  order_manager_itch_bist_perf  FillerITCH_BIST::fill_itch_add
 8.25%  5.01%  order_manager_itch_bist_perf  MarketBook::delete_order
 7.72%  4.67%  order_manager_itch_bist_perf  MarketBook::add_order
 4.30%  4.30%  order_manager_itch_bist_perf  MarketBook::exec_order
 3.05%  1.79%  order_manager_itch_bist_perf  absl::btree::insert_unique
 1.50%  1.50%  order_manager_itch_bist_perf  absl::btree::erase
```

C2C:
```
Total Shared Cache Lines:  0
Load Local HITM:           0
```

---

### OrderManager ITCH NASDAQ RAW

```
task-clock:            555,258,414        0.979 CPUs utilized
page-faults:                   764        1.376 K/sec

P-Core (cpu_core):
  instructions:    10,160,230,323        3.78  insn/cycle
  cycles:           2,689,839,328        4.844 GHz
  branches:         1,718,584,176        3.095 G/sec
  branch-misses:           25,854        0.00%
  L1-dcache-loads:  2,751,705,185        4.956 G/sec
  L1-dcache-misses:       338,048        0.01%
  LLC-loads:               68,853      124.002 K/sec
  LLC-load-misses:          7,923       11.51%

Wall: 0.555s  User: 0.551s  Sys: 0.006s
```

```
P-Core L1-icache-load-misses:   242,215
P-Core dTLB-load-misses:          9,832
P-Core iTLB-load-misses:          1,407
```

CPU+Branch top frames:
```
96.25% 18.04%  order_manager_itch_nasdaq_perf  OrderManager::store()
68.17% 11.72%  order_manager_itch_nasdaq_perf  std::__do_visit (NASDAQ ITCH variant)
24.18% 24.18%  order_manager_itch_nasdaq_perf  absl::flat_hash_map::find
12.78% 10.72%  order_manager_itch_nasdaq_perf  OrderManager::add_market_order
 8.27%  4.76%  order_manager_itch_nasdaq_perf  MarketBook::delete_order
 4.84%  4.84%  order_manager_itch_nasdaq_perf  MarketBook::exec_order
 3.60%  3.60%  order_manager_itch_nasdaq_perf  boost::lockfree::ringbuffer_base<Order*>::push
 2.78%  2.78%  order_manager_itch_nasdaq_perf  FillerITCH_NASDAQ::fill_itch_add
```

C2C:
```
Total Shared Cache Lines:  0
Load Local HITM:           0
```

---

### OrderManager OUCH BIST RAW

```
task-clock:         10,615,693,445        3.936 CPUs utilized
page-faults:                   782       73.665 /sec

P-Core (cpu_core):
  instructions:    18,722,426,500        0.42  insn/cycle
  cycles:          44,064,282,710        4.151 GHz
  branches:         3,594,106,691      338.565 M/sec
  branch-misses:       33,726,535        0.94%
  L1-dcache-loads:  5,411,392,951      509.754 M/sec
  L1-dcache-misses:    163,949,377        3.03%
  LLC-loads:           153,728,295       14.481 M/sec
  LLC-load-misses:          13,536        0.01%

E-Core (cpu_atom):
  instructions:     9,635,484,373        1.04  insn/cycle
  cycles:           9,275,813,082        0.874 GHz

Wall: 2.697s  User: 10.609s  Sys: 0.009s
```

```
P-Core L1-icache-load-misses:   3,987,816
P-Core dTLB-load-misses:          102,950
P-Core iTLB-load-misses:          144,293
```

CPU+Branch top application frames:
```
15.18%  2.57%  order_manager_ouch_bist_perf  OrderManager::store()
 6.27%  6.19%  order_manager_ouch_bist_perf  BIST::OUCHOrderCancelledMessage handler
 4.77%  4.77%  order_manager_ouch_bist_perf  boost::lockfree::ringbuffer_base::push (SPSC)
 3.18%  0.83%  order_manager_ouch_bist_perf  folly::ConcurrentHashMap::eraseImpl
 1.77%  1.77%  libc.so.6                     pthread_mutex_lock
```

ICache top application frames:
```
46.13%  1.80%  order_manager_ouch_bist_perf  OrderManager::store()
32.49%  3.54%  order_manager_ouch_bist_perf  folly::ConcurrentHashMap::eraseImpl
22.32%  1.00%  order_manager_ouch_bist_perf  folly::hazptr_domain::push_list
20.06%  0.16%  order_manager_ouch_bist_perf  folly::detail::hazptr_inline_executor_add
```

TLB top application frames:
```
85.27% 18.77%  order_manager_ouch_bist_perf  OrderManager::store()
58.48% 11.46%  order_manager_ouch_bist_perf  folly::ConcurrentHashMap::eraseImpl
 5.92%  5.92%  order_manager_ouch_bist_perf  OrderManager::add_our_order
 3.84%  3.84%  libc.so.6                     pthread_mutex_lock
```

C2C:
```
Total Shared Cache Lines:  11
Load HITs on shared lines: 22,948
Load Local HITM:            8,400
Blocked Access on shared:     275
Store HITs on shared:      17,462
```

---

### OrderManager OUCH NASDAQ RAW

```
task-clock:         13,600,684,916        3.949 CPUs utilized
page-faults:                   787       57.865 /sec

P-Core (cpu_core):
  instructions:    19,646,108,800        0.35  insn/cycle
  cycles:          56,532,798,687        4.157 GHz
  branches:         3,734,485,002      274.581 M/sec
  branch-misses:       40,082,906        1.07%
  L1-dcache-loads:  5,786,034,471      425.422 M/sec
  L1-dcache-misses:    236,145,658        4.08%
  LLC-loads:           225,340,959       16.568 M/sec
  LLC-load-misses:           9,772        0.00%

E-Core (cpu_atom):
  instructions:    11,530,354,333        1.08  insn/cycle
  cycles:          10,696,800,098        0.786 GHz

Wall: 3.444s  User: 13.593s  Sys: 0.009s
```

```
P-Core L1-icache-load-misses:   4,226,950
P-Core dTLB-load-misses:          244,587
P-Core iTLB-load-misses:           32,321
```

DCache top application frames:
```
44.86%  1.69%  order_manager_ouch_nasdaq_perf  OrderManager::store()
17.66%  1.51%  order_manager_ouch_nasdaq_perf  std::__do_visit (NASDAQ OUCH variant)
15.53% 15.42%  order_manager_ouch_nasdaq_perf  boost::lockfree::ringbuffer_base::push (SPSC)
14.26%  9.00%  order_manager_ouch_nasdaq_perf  folly::ConcurrentHashMap::eraseImpl
 8.98%  8.95%  order_manager_ouch_nasdaq_perf  boost::lockfree::ringbuffer_base<Order*>::push
```

TLB top application frames:
```
98.10%  2.99%  order_manager_ouch_nasdaq_perf  OrderManager::store()
65.76% 30.53%  order_manager_ouch_nasdaq_perf  std::__do_visit (NASDAQ OUCH variant)
22.98%  2.03%  order_manager_ouch_nasdaq_perf  folly::ConcurrentHashMap::eraseImpl
10.51% 10.51%  order_manager_ouch_nasdaq_perf  OrderManager::add_our_order
 7.72%  7.72%  libc.so.6                       pthread_mutex_lock
```

C2C:
```
Total Shared Cache Lines:  13
Load HITs on shared lines: 29,097
Load Local HITM:           11,270
Blocked Access on shared:     389
Store HITs on shared:      21,383
```
