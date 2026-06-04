# Design Patterns

This page documents the design patterns applied across the TradingEngine pipeline. Each pattern is described briefly and mapped to its concrete usage in the codebase.

---

## Object Pool

**What:** Pre-allocate a fixed set of objects at startup and reuse them instead of allocating/deallocating on demand. Eliminates heap allocation on the hot path.

**Where:**

1. `ProtocolMessagePools` — preallocated FIX/ITCH/OUCH message objects
2. `PacketPools` — preallocated outbound and inbound network packet buffers
3. `OrderPool` — preallocated order objects
4. `OrderRiskPool` in `RiskEngine` — preallocated `OrderRisk` entries for active order tracking

**Why:** Dynamic allocation (`new`/`delete`) is non-deterministic in latency due to heap fragmentation and OS interaction. Object pools give O(1) predictable allocation with zero system calls on the hot path.

---

## Producer-Consumer (SPSC Queue)

**What:** Decouple producers and consumers of data using a shared queue. In this engine, all inter-thread communication uses lock-free Single-Producer Single-Consumer (SPSC) queues.

**Where:** Every pipeline stage boundary:

1. `receiver_to_parser` — NetworkIO → ParserDispatch
2. `builder_to_sender` — BuilderDispatch → NetworkIO
3. `parser_to_store` — ParserDispatch → OrderManager
4. `store_to_risk` — OrderManager → RiskEngine (update)
5. `strategy_to_risk` — StrategyEngine → RiskEngine (check)
6. `risk_to_builder` — RiskEngine → BuilderDispatch
7. `store_to_db` — OrderManager → ClickhouseWriter
8. `db_to_parser` — ClickhouseWriter → ParserDispatch

**Why:** SPSC queues are wait-free for both producer and consumer when they operate on separate threads. No mutexes, no contention, no blocking. `boost::lockfree::spsc_queue` is used throughout.

---

## Facade

**What:** Provide a simplified interface to a complex subsystem. Hides internal wiring and lifecycle management from the caller.

**Where:** 

1. `TradingEngine` — constructs, wires, and manages the lifetime of all subsystems (NetworkIO, ParserDispatch, BuilderDispatch, OrderManager, RiskEngine, MarketBook, HashTables, SessionManager, LoginController, ClickhouseWriter). The caller only interacts with `start()` and `stop()`.

**Why:** Without a facade, the caller would need to know the construction order, dependency graph, and thread lifecycle of every subsystem. `TradingEngine` encapsulates all of this.

---

## Strategy Pattern

**What:** Define a family of algorithms, encapsulate each one, and make them interchangeable at runtime or compile time.

**Where:**

1. `Builder_Dispatch` — selects the appropriate builder (`Builder_FIX`, `Builder_OUCH_BIST`, `Builder_OUCH_NASDAQ`) based on the order's venue and protocol at runtime.
2. `Parser_Dispatch` — routes incoming packets to the correct parser (`Parser_FIX`, `Parser_OUCH_BIST`, `Parser_ITCH_BIST`, etc.) based on protocol and venue.

**Why:** New venues or protocols can be added by implementing a new builder/parser and registering it in the dispatch — existing code is unchanged. Open/Closed Principle in practice.

---

## Chain of Responsibility

**What:** Pass a request along a chain of handlers. Each handler decides to process it or pass it further.

**Where:** `RiskEngine::check_risk()` — an incoming order passes through a sequence of independent checks:

1. `check_venue_halt_and_circuit()`
2. `check_order_rate_limit()`
3. `check_price_tick_valid()`
4. `check_quantity_bounds()`
5. `check_max_open_orders_account/symbol()`
6. `check_self_trade_order()`
7. `check_account_risk()` (which itself chains: leverage, drawdown, unrealized loss, margin, balance, notional)

Each check returns a reject reason bitmask. The chain short-circuits on certain critical failures.

**Why:** Each risk check is independent, testable, and can be added or removed without touching others. The bitmask approach allows multiple reject reasons to be accumulated in a single pass.

---

## Command

**What:** Encapsulate a request as an object, allowing it to be passed, queued, stored, and processed independently of the requester.

**Where:** 

1. `Order` struct — every order (new, cancel, replace, fill) is represented as a self-contained `Order` object carrying all necessary state (venue, protocol, price, quantity, side, status, exec info). It is created once and flows through the entire pipeline — from strategy to risk to builder to network — without being reconstructed.

**Why:** Decouples the originator of an order from its execution. Each pipeline stage reads and writes the same `Order` object in place, avoiding copies and extra allocations.

---

## Composite

**What:** Treat a group of objects the same way as a single object. A composite object delegates to its children.

**Where:**

1. `Parser_Dispatch` — owns and delegates to `Parser_ITCH_BIST`, `Parser_ITCH_NASDAQ`, `Parser_OUCH_BIST`, `Parser_OUCH_NASDAQ`, and holds a reference to `Parser_FIX`. A single `parseFIX()` or `parseITCH()` call dispatches to the correct child.
2. `Builder_Dispatch` — owns `Builder_OUCH_BIST`, `Builder_OUCH_NASDAQ`, and holds a reference to `Builder_FIX`. A single `build()` call dispatches to the correct child.

**Why:** The rest of the system does not need to know which specific parser or builder handles a given message. The dispatch layer absorbs protocol and venue complexity.

---

## Factory Method

**What:** Define an interface for creating an object, but let subclasses or specific components decide which class to instantiate.

**Where:**

1. `LoginController::LoginAttempt()` — constructs the correct login message (`SBT Login` or `FIX Logon`) depending on the session protocol.
2. `Builder_FIX` / `Builder_OUCH_BIST` / `Builder_OUCH_NASDAQ` — each constructs a protocol-specific wire message from a generic `Order` object.

**Why:** The caller does not need to know the wire format of the message. It provides an `Order` and the factory produces the correct byte sequence for the target exchange.

---

## Observer (epoll)

**What:** Define a one-to-many dependency so that when one object changes state, all dependents are notified automatically.

**Where:** 

1. `NetworkIO` — epoll is the kernel-level implementation of the Observer pattern. Each socket file descriptor is registered as a subscriber (`epoll_ctl`), and `epoll_wait()` acts as the notification mechanism. When data arrives or a socket becomes writable, the corresponding event handler is invoked.

**Why:** Kernel bypass solutions (DPDK, RDMA, Solarflare OpenOnload) were not available due to hardware limitations. epoll was chosen as the most performant alternative available on standard Linux — it scales to thousands of file descriptors with O(1) event delivery, avoids busy-polling CPU waste, and integrates naturally with non-blocking sockets. Edge-triggered mode (`EPOLLET`) is used throughout to minimize spurious wakeups and syscall overhead.

---

## Summary

| Pattern | Where |
|---------|-------|
| Object Pool | `MessagePool`, `TxPacketPoolManager`, `orderrisk_pool_`, `PacketPool` |
| Producer-Consumer | All SPSC queues between pipeline stages |
| Facade | `TradingEngine` |
| Strategy | `Builder_Dispatch`, `Parser_Dispatch` |
| Chain of Responsibility | `RiskEngine::check_risk()` |
| Command | `Order` struct flowing through the pipeline |
| Composite | `Parser_Dispatch`, `Builder_Dispatch` owning child parsers/builders |
| Factory Method | `LoginController`, `Builder_FIX`, `Builder_OUCH_*` |
| Observer | `NetworkIO` epoll-based socket event handling |
