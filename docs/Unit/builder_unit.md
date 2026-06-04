# Builder — Unit Tests

## Overview

Unit tests for `Builder_Dispatch` — the component responsible for routing outbound orders to the correct protocol builder (FIX, OUCH BIST, OUCH NASDAQ), constructing wire-format `TxPacket`s, and pushing them to the sender queue.

Tests are written with **Google Test (gtest)**. The test pushes `Order*` objects directly to the `risk_to_builder` queue and verifies the resulting `TxPacket` bytes and metadata.

---

## Test Environment

| Component | Role |
|---|---|
| `Builder_Dispatch` | System under test |
| `risk_to_builder` | Input queue — `Order*` pushed by test |
| `builder_to_sender` | Output queue — `TxPacket*` popped for verification |
| `SessionManager` | Provides session state per venue/protocol |
| `Builder_FIX` | FIX packet builder, injected dependency |
| `TxPacketPoolManager` | Manages `TxPacket` memory pool |

---

## Test Cases

### BuilderDispatchTest.MixedMessageTraffic

Verifies that `Builder_Dispatch` correctly routes three orders across all three outbound protocols, builds wire-format packets with correct byte layout, and tags each `TxPacket` with the right socket index.

---

#### Input Dataset

Three test orders are pushed to the `risk_to_builder` queue, one per dispatch call:

| # | Order | Venue | Protocol |
|---|---|---|---|
| 1 | `fix_new_order` | BIST | FIX |
| 2 | `ouch_new_order` | BIST | OUCH |
| 3 | `NQouch_new_order` | NASDAQ | OUCH |

---

#### Expected Output — 3 TxPackets

---

**TxPacket 1 — FIX New Order**

| Field | Expected | Notes |
|---|---|---|
| `data[0]` | `'8'` | SoupBinTCP message type — FIX payload start |
| `len` | 177 | Total packet length including SoupBinTCP header |
| `sock_index` | 0 | FIX BIST session socket |
| `is_login_msg` | `false` | Order packet, not login |

---

**TxPacket 2 — OUCH BIST New Order**

| Field | Expected | Notes |
|---|---|---|
| `data[33]` | `0x27` | Price high byte — price = 10000 (0x2710) |
| `len` | 117 | Total packet length including SoupBinTCP header |
| `sock_index` | 1 | OUCH BIST session socket |
| `is_login_msg` | `false` | Order packet, not login |

---

**TxPacket 3 — OUCH NASDAQ New Order**

| Field | Expected | Notes |
|---|---|---|
| `data[12]` | `0x64` | Quantity byte — quantity = 100 (0x64) |
| `len` | 50 | Total packet length including SoupBinTCP header |
| `sock_index` | 2 | OUCH NASDAQ session socket |
| `is_login_msg` | `false` | Order packet, not login |

---

## Results

```
Running main() from googletest/src/gtest_main.cc
[==========] Running 1 test from 1 test suite.
[----------] Global test environment set-up.
[----------] 1 test from BuilderDispatchTest
[ RUN      ] BuilderDispatchTest.MixedMessageTraffic
[       OK ] BuilderDispatchTest.MixedMessageTraffic (238 ms)
[----------] 1 test from BuilderDispatchTest (238 ms total)
[----------] Global test environment tear-down
[==========] 1 test from 1 test suite ran. (239 ms total)
[  PASSED  ] 1 test.
```

---

## Notes

- Each order is dispatched individually — `risk_to_builder.push()` followed by `builder_dispatch->dispatch()` — to verify that routing and building are correct per call without batching effects.
- Socket index assignment (`sock_index` 0, 1, 2) maps directly to the session order registered in `SessionManager` — FIX BIST, OUCH BIST, OUCH NASDAQ respectively.
- Packet length differences (177, 117, 50) reflect protocol overhead: FIX carries tag-value field text, OUCH BIST carries full order book fields, OUCH NASDAQ has a more compact format.
- `is_login_msg = false` on all three confirms the builder correctly distinguishes order packets from login/session management packets which share the same sender queue.
