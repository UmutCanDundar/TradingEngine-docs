# Order Manager — Unit Tests

## Overview

Unit tests for `OrderManager` — the component responsible for matching incoming market messages (FIX, OUCH, ITCH) to pending orders, updating `Order` state (status, quantities, timestamps), maintaining the `MarketBook`, and forwarding updated orders to the risk queue.

Tests are written with **Google Test (gtest)**. The test pushes `MessageWithVenue` objects directly to the `parser_to_store` queue and verifies the resulting `Order*` state on `store_to_risk` after each `store()` call.

---

## Test Environment

| Component | Role |
|---|---|
| `OrderManager` | System under test |
| `parser_to_store` | Input queue — `MessageWithVenue` pushed by test |
| `store_to_risk` | Output queue — `Order*` popped for verification |
| `MarketBook` | Updated by ITCH messages, queried to verify best bid/ask |
| `HashTables` | Symbol metadata lookup |

Three orders are pre-registered via `add_awaitingAck_order()` before traffic begins:

| Order | Venue | Protocol | Symbol | Side | Price | Qty | Token |
|---|---|---|---|---|---|---|---|
| FIX | BIST | FIX | GARAN | Buy | 15000 | 500 | `"F1"` / cl_ord_id `"O1"` |
| OUCH BIST | BIST | OUCH | GARAN | Buy | 10000 | 100 | `"TOKEN000000042"` |
| OUCH NASDAQ | NASDAQ | OUCH | AAPL | Buy | 10000 | 1000 | `"CLIENT0000001 "` |

---

## Test Cases

### OrderManagerTest.MixedMessageTraffic

Processes 20 messages across all protocols and venues, verifying `Order` state transitions, quantity tracking, and `MarketBook` updates at each step.

---

#### Setup — SymbolMeta Initialization

Before order traffic, two directory messages initialize symbol metadata:

- NASDAQ ITCH `StockDirectory` — AAPL (`stock_locate=1`)
- BIST ITCH `OrderBookDirectory` — GARAN (`order_book_id=3`)

These are required for instrument ID resolution in subsequent messages.

---

#### Message 1 — FIX New Order (GARAN, buy 500@15000)

FIX `ExecutionReport` with `exec_type='0'` (New) matches the pre-registered FIX order.

| Field | Expected |
|---|---|
| `price` | 15000 |
| `quantity` | 500 |
| `filled_quantity` | 0 |
| `remaining_quantity` | 500 |
| `status` | `Status::New` |
| `side` | `Side::Buy` |
| `venue` | `Venue::BIST` |
| `protocol` | `Protocol::FIX` |
| `order_type` | `OrderType::Limit` |
| `instrument_id` | 3 |
| `order_id` | hash of FIX `order_id` field |
| `exec_type` | `'0'` |
| `syncState` | `SyncState::NewSeen` |
| `client_order_token` | `"F1"` |
| `fix_org_order_id` | `"O1"` |

---

#### Message 2 — BIST OUCH Order Accepted (GARAN, order_id=42)

OUCH `OUCHOrderAcceptedMessage` matches the pre-registered OUCH BIST order.

| Field | Expected |
|---|---|
| `price` | 10000 |
| `quantity` | 100 |
| `remaining_quantity` | 100 |
| `status` | `Status::New` |
| `order_id` | 42 |
| `syncState` | `SyncState::WaitingNew` |
| `client_order_token` | `"TOKEN000000042"` |
| `fix_org_order_id` | `""` |

---

#### Message 3 — BIST ITCH Add Order (GARAN, 1000@10000 Buy)

Market order — not our order (`isOurOrder=false`). Updates `MarketBook`.

| Field | Expected |
|---|---|
| `quantity` | 1000 |
| `price` | 10000 |
| `isOurOrder` | `false` |
| `protocol` | `Protocol::ITCH` |
| `order_type` | `OrderType::Market` |
| `order_id` | 42 |
| `timestamp` | 100 |

**MarketBook after:**

| Level | Price | Qty |
|---|---|---|
| Best Bid | 10000 | 1000 |
| Best Ask | 0 | 0 |

---

#### Message 4 — NASDAQ OUCH Order Accepted (AAPL, user_ref=99)

Matches pre-registered NASDAQ OUCH order.

| Field | Expected |
|---|---|
| `price` | 10000 |
| `quantity` | 1000 |
| `order_id` | 99 |
| `user_ref_num` | 99 |
| `venue` | `Venue::NASDAQ` |
| `client_order_token` | `"CLIENT0000001 "` |

---

#### Message 5 — NASDAQ ITCH Add Order (AAPL, 100@10000 Buy)

Market order. Updates NASDAQ MarketBook.

| Field | Expected |
|---|---|
| `quantity` | 100 |
| `order_id` | 99 |
| `isOurOrder` | `false` |
| `timestamp` | 100 |

---

#### Message 6 — FIX Partial Fill (GARAN, 200 filled)

FIX `ExecutionReport` with `exec_type='F'`, `last_qty=200`.

| Field | Expected |
|---|---|
| `filled_quantity` | 200 |
| `remaining_quantity` | 300 |
| `last_exec_quantity` | 200 |
| `status` | `Status::Partial` |
| `exec_type` | `'F'` |

---

#### Message 7 — BIST ITCH Order Executed (GARAN, 600 executed)

Market order partially filled. Updates MarketBook quantity.

| Field | Expected |
|---|---|
| `filled_quantity` | 600 |
| `remaining_quantity` | 400 |
| `last_exec_quantity` | 600 |
| `status` | `Status::Partial` |
| `last_update_time` | 200 |

**MarketBook after:**

| Level | Price | Qty |
|---|---|---|
| Best Bid | 10000 | 400 |
| Best Ask | 0 | 0 |

---

#### Message 8 — BIST OUCH Order Executed (GARAN, 60 traded)

Our OUCH BIST order partially filled.

| Field | Expected |
|---|---|
| `filled_quantity` | 60 |
| `remaining_quantity` | 40 |
| `last_exec_quantity` | 60 |
| `status` | `Status::Partial` |

---

#### Message 9 — NASDAQ ITCH Order Executed (AAPL, 60 executed)

| Field | Expected |
|---|---|
| `filled_quantity` | 60 |
| `remaining_quantity` | 40 |
| `last_update_time` | 200 |

---

#### Message 10 — NASDAQ OUCH Order Executed (AAPL, 600 executed)

| Field | Expected |
|---|---|
| `filled_quantity` | 600 |
| `remaining_quantity` | 400 |
| `last_exec_quantity` | 600 |
| `status` | `Status::Partial` |

---

#### Message 11 — BIST ITCH Order Delete (GARAN, remaining 400 deleted)

Market order removed from book.

| Field | Expected |
|---|---|
| `status` | `Status::Deleted` |
| `last_update_time` | 300 |
| `cancelled_count` | 1 |

**MarketBook after:**

| Level | Price | Qty |
|---|---|---|
| Best Bid | 0 | 0 |
| Best Ask | 0 | 0 |

---

#### Message 12 — BIST OUCH Order Cancelled (GARAN)

Our OUCH BIST order cancelled. `replaced_quantity = -40` (remaining qty negated).

| Field | Expected |
|---|---|
| `remaining_quantity` | 0 |
| `replaced_quantity` | -40 |
| `status` | `Status::Deleted` |
| `cancelled_count` | 1 |

---

#### Message 13 — NASDAQ ITCH Order Delete (AAPL)

| Field | Expected |
|---|---|
| `status` | `Status::Deleted` |
| `last_update_time` | 300 |
| `cancelled_count` | 1 |

---

#### Message 14 — NASDAQ OUCH Order Cancelled (AAPL)

`replaced_quantity = -400`. Status is `Cancelled` (NASDAQ OUCH uses Cancelled, BIST OUCH uses Deleted).

| Field | Expected |
|---|---|
| `remaining_quantity` | 0 |
| `replaced_quantity` | -400 |
| `status` | `Status::Cancelled` |
| `cancelled_count` | 1 |

---

#### Message 15 — FIX Cancel Confirm (GARAN)

`exec_type='4'` (Cancelled). `replaced_quantity = -300`.

| Field | Expected |
|---|---|
| `remaining_quantity` | 0 |
| `replaced_quantity` | -300 |
| `filled_quantity` | 200 |
| `status` | `Status::Deleted` |
| `exec_type` | `'4'` |
| `cancelled_count` | 1 |

---

#### Messages 16–18 — MarketBook Progression (BIST ITCH Add Orders)

Three additional ITCH add orders test `MarketBook` best bid/ask update logic:

| Step | Message | Expected Best Bid | Expected Best Ask |
|---|---|---|---|
| 16 | GARAN 1000@100 Buy | 100 | 0 |
| 17 | GARAN 1000@1000 Buy | 1000 | 0 |
| 18 | GARAN 1000@1000 Sell | 1000 | 1000 |

Message 17 replaces message 16 as best bid (higher price). Message 18 sets best ask at the same price level — verifies bid/ask independence.

---

## Results

```
Running main() from googletest/src/gtest_main.cc
[==========] Running 1 test from 1 test suite.
[----------] Global test environment set-up.
[----------] 1 test from OrderManagerTest
[ RUN      ] OrderManagerTest.MixedMessageTraffic
[       OK ] OrderManagerTest.MixedMessageTraffic (13 ms)
[----------] 1 test from OrderManagerTest (13 ms total)
[----------] Global test environment tear-down
[==========] 1 test from 1 test suite ran. (13 ms total)
[  PASSED  ] 1 test.
```

---

## Notes

- The 13ms runtime reflects in-process queue operations with no I/O.
- BIST OUCH and NASDAQ OUCH use different terminal statuses for cancellation (`Deleted(remaining qty = 0)` vs `Cancelled(remaning qty >= 0)`) — both are verified explicitly.
- MarketBook state is verified at four checkpoints (after add, after partial exec, after delete, and through the final three adds) to confirm incremental update correctness.
