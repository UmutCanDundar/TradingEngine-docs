# TradingEngine

A low-latency algorithmic trading engine written in C++23, designed for direct exchange connectivity across multiple venues and protocols.

---

## Overview

TradingEngine is a multi-venue, multi-protocol order execution engine built for algorithmic trading. It handles the full lifecycle of an order — from market data ingestion and order book maintenance, through risk validation, to order construction and transmission to the exchange.

The engine is designed with a focus on predictable, low-latency execution. Every component on the hot path avoids dynamic allocation, uses lock-free data structures, and applies explicit cache-line isolation to minimize false sharing between threads.

!!! note "Work in Progress"
The engine is currently in its Core Performance Phase. The primary focus is achieving deterministic, low-latency execution on the "Happy Path." The following areas are under active development:

**Strategy Engine** – Implementation of the main decision-making logic is pending.

**Database Integration** – ClickHouse connectivity for post-trade analysis is 
partially implemented.

**System Monitoring** – A dedicated telemetry unit is planned to monitor pipeline 
health, thread-heartbeats, and latency histograms in real-time.

---

## Key Properties

| Property | Detail |
|----------|--------|
| Language | C++23, (Python, Shell — tooling)|
| Venues | BIST, NASDAQ |
| Inbound protocols | FIX (BIST), OUCH (BIST, NASDAQ), ITCH (BIST, NASDAQ) |
| Outbound protocols | FIX (BIST), OUCH (BIST, NASDAQ) |
| Account model | Single account per venue |
| Concurrency model | Lock-free, SPSC queues between pipeline stages |
| Memory model | Preallocated pools, no heap allocation on hot path |
| Kernel bypass | Not used (hardware limitations) |

---

## Pipeline

The engine processes two concurrent pipelines:

**Rx (Inbound)** – `Exchange → NetworkIO → Parser → OrderManager → RiskEngine` 

**Tx (Outbound)** – `Strategy → RiskEngine → Builder → NetworkIO → Exchange`


Each stage communicates via lock-free SPSC queues. Stages run on pinned CPU cores to eliminate scheduler interference.

---

## Components

### Core
- **MarketBook** — maintains aggregated best bid/ask per symbol using a B-tree price level structure. Publishes atomic best prices for consumption by the risk engine.
- **HashTables** — custom hash tables for symbol lookup and exec ID dedup.

### Main 
- **NetworkIO** — epoll-based non-blocking socket I/O. Manages TCP (FIX, OUCH) and UDP (ITCH) connections. Handles login/session lifecycle via SoupBinTCP.
- **Parser** — protocol-specific parsers for FIX, OUCH, and ITCH. Dispatches parsed messages downstream.
- **OrderManager** — maintains the internal order state. Matches execution reports against pending orders.
- **RiskEngine** — dual-thread risk evaluation. One thread updates risk state from execution reports and market data; another validates outbound orders pre-trade.
- **Builder** — constructs outbound protocol messages (FIX NewOrderSingle, OUCH Enter Order) from internal order structures.

### Session
- **SessionManager** — tracks connection and login state per session.
- **LoginController** — manages login sequences for SoupBinTCP and FIX sessions.
- **SoupBinTCP** — SBT session layer handler (login, logout, heartbeat, sequencing).

### Protocol
- **Builder_FIX / Builder_OUCH** — venue and protocol-specific message serializers.
- **Parser_FIX / Parser_OUCH / Parser_ITCH** — venue and protocol-specific message deserializers.

### Database
- **ClickhouseWriter** — asynchronous trade and order event logging to ClickHouse (partially implemented).

---

## Design Highlights

- **Zero-copy on hot path** – Messages are written directly into preallocated packet buffers with no intermediate copies, eliminating heap allocation overhead during execution.

- **Lock-free pipeline** – All inter-thread communication is handled via SPSC queues (boost::lockfree::spsc_queue). No mutexes or condition variables exist on production.

- **Atomic State & Memory Ordering** – The pipeline utilizes std::atomic with explicit Memory Barriers (Acquire/Release semantics) for cross-thread synchronization. This ensures maximum throughput by avoiding expensive sequentially consistent (memory_order_seq_cst) operations where possible.

- **Cache-line isolation** – Data structures are grouped by access pattern and isolated with alignas(64) to eliminate false sharing and optimize L1/L2 cache hits.

- **Code generation** – ITCH/OUCH message types and symbol tables are strictly generated from exchange specifications at build time, ensuring binary-level accuracy.

- **Static Dispatch** – The engine utilizes function pointer tables, switch-case, and tag dispatch to achieve high-performance routing while avoiding runtime vtable overhead.

- **Busy-Wait Polling** – Network and Flow threads employ a 100% duty-cycle polling strategy. Spin-loops are optimized with _mm_pause() intrinsics to prevent CPU pipeline flushes and reduce power consumption while maintaining sub-microsecond responsiveness.

- **SIMD-Accelerated Parsing** – The FIX parser utilizes SIMD intrinsics to batch-process tag/value delimiters, significantly reducing the clock cycles required for string-to-binary conversion.

- **Design Patterns** – The engine applies several well-known design patterns across its pipeline. See [Design Patterns](design.md) for a detailed breakdown.

---

## Roadmap & Resilience Engineering

Current development ensures that the engine performs at peak speed when data is valid. The next phases will introduce robustness against "Real World" anomalies:

Phase 2: Production Readiness & Safety

- **Binary Journaling** – Implementing a high-speed, sequential disk-logger to 
persist the last seen Sequence Numbers and critical state for instant recovery 
after crashes.

- **Validation & Boundary Checks** – Adding Checksum validation and rigorous 
protection against malformed packets or protocol violations from the exchange 
side.

- **Secure Config Management** – Transitioning from plain-text JSON credentials to 
**AES-256 encrypted secrets** with environment-based master keys.

- **Transport Layer Security (TLS/SSL)** – Integrating **OpenSSL** for FIX 
connectivity to ensure encrypted communication with exchange gateways.

Phase 3: Micro-Optimizations

- **Warm-up Sequences** – Pre-market dummy-load injection to prime the Instruction 
Cache and Branch Predictor for "first-packet" stability.

- **Latency Telemetry** – Integrating a non-intrusive monitoring unit to track 
nanosecond-level jitter across each pipeline stage without impacting the hot path.

---
    
## Repository Structure

```
TradingEngine/
├── config/                # Risk limits and session configuration
├── generated/             # Code-generated message types and symbol tables
├── logs/
├── scripts/
│   ├── test/              # Unit, benchmark, perf and pipeline test runners
│   └── tools/             # Code generation scripts
├── src/
│   ├── core/              # Order, HashTables, MessagePool, common types
│   ├── database/          # ClickhouseWriter
│   ├── flows/             # Thread loop definitions per pipeline stage
│   ├── market/            # MarketBook
│   ├── network/           # NetworkIO, packet pools
│   ├── order_management/  # OrderManager and fillers
│   ├── protocol/          # Builders, parsers, dispatch (FIX, OUCH, ITCH)
│   ├── risk/              # RiskEngine, Limits
│   ├── session/           # SessionManager, LoginController, SoupBinTCP
│   ├── strategy/
│   └── utils/             # Logger, ErrorHandler, Endian, common.h
└── test/                  # Unit tests, benchmarks, perf targets
```

---

## Build & Source Code

This project is closed-source. The source code is available for review upon request during the interview process — please reach out directly.

If you have been granted access, the following will get you up and running:

### Prerequisites

- **Compiler:** g++ 13+ (C++23)
- **Build System:** CMake 3.16+, Ninja
- **Python:** Python3 (for code generation scripts)
- **Libraries:** Boost, Abseil, nlohmann/json, ClickHouse C++ Client, spdlog
- **Environment:** Linux x86\_64 

### Quick Start

```bash
./setup.sh    # fetches any missing third-party dependencies automatically. 
              # If a library is already installed system-wide, it is used 
              # directly — no duplicate downloads.
./build.sh    # build Release  
./test.sh     # run all tests
```

### Build Modes

| Command | Description |
|---------|-------------|
| `./build.sh` | Release (default) |
| `./build.sh debug` | Debug |
| `./build.sh relwithdebinfo` | RelWithDebInfo |
| `./build.sh asan` | Debug + AddressSanitizer |
| `./build.sh tsan` | RelWithDebInfo + ThreadSanitizer |

ASan and TSan builds are intended for development and debugging only.

### Test Modes

| Command | Description |
|---------|-------------|
| `./test.sh` | All test below |
| `./test.sh unit` | All unit test |
| `./test.sh bench` | All benchmark test |
| `./test.sh perf` | All perf test |
| `./test.sh pipeline` | Engine Rx-Tx benchmark tests + Engine RxTx perf test |

## Protocol Source Material

### Borsa İstanbul
| Protocol | Category | Spesification Document |
|----------|----------|------------------------|
| BIST-FIX | Order Entry | [Genium INET FIX Protocol Specification](https://www.borsaistanbul.com/files/genium-inet-fix-protocol-specification.pdf) |
| BIST-ITCH | Market Data | [BISTECH ITCH Protocol Specification](https://www.borsaistanbul.com/files/bistech-itch-protocol-specification.pdf) |
| BIST-OUCH | Order Entry | [BISTECH OUCH Protocol Specification](https://www.borsaistanbul.com/files/bistech-ouch-protocol-specification.pdf) |
| SoupBinTCP | Session Layer | [SoupBinTCP Protocol Specification](https://www.borsaistanbul.com/files/soupbintcp-protocol-specification24F25E31E16B2FE7E78531A3.pdf) |

### Nasdaq
| Protocol | Category | Spesification Document |
|----------|----------|------------------------|
| NASDAQ-ITCH | Market Data | [Nasdaq TotalView-ITCH 5.0](https://www.nasdaqtrader.com/content/technicalsupport/specifications/dataproducts/NQTVITCHSpecification.pdf) |
| NASDAQ-OUCH | Order Entry | [OUCH 5.0 Order Entry Specification](https://nasdaqtrader.com/content/technicalsupport/specifications/TradingProducts/Ouch5.0.pdf) |
| SoupBinTCP | Session Layer | [SoupBinTCP Version 4.00](https://www.nasdaq.com/docs/SoupBinTCP%204.0.pdf) |

---
