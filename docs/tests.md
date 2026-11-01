## Hardware & Build

| | |
|--|--|
| CPU | Intel Core i7-13620H, 10 core (6P + 4E), 16 threads |
| RAM | 16GB |
| OS | Ubuntu 24, Kernel 6.17.0-29 |
| Compiler | g++ 13.3 |
| Build | Release (-O3 -DNDEBUG) |

## Test Philosophy

Each component is tested both in isolation and as part of 
the full pipeline. This allows measuring the cost of each 
stage independently before observing end-to-end behavior.

### Dataset Types
| Label | Description |
|-------|-------------|
| **Best** | Single-message/single-packet, ideal conditions |
| **Normal** | Full-messages/single-packet, realistic scenario |
| **Worst** | Partial-message(s)/multi-packet, stress conditions |

## Test Structure

Five levels of testing were applied across all components:

- **Unit (gtest)** — Correctness testing of each component in isolation 
- **Benchmark (google benchmark)** — Latency and cycle count measurement 
- **Perf** — Cache miss, branch prediction, TLB analysis... 
- **AddressSanitizer (ASan)** — Memory leaks, out-of-bounds accesses, and use-after-free bugs (minor leaks were successfully identified and resolved).
- **ThreadSanitizer (TSan)** — Data races and deadlocks, ensuring strict thread safety across concurrent components (executed with clean runs).

### Component Hierarchy

Where applicable, both the top-level interface and the 
underlying protocol handlers are tested independently. 
This isolates per-stage costs before measuring combined 
overhead — for example, measuring a FIX parser in isolation 
before measuring it through the parser dispatch layer.


**Note**: This section describes the initial design. For the current state see Recent Optimization and Summary.
