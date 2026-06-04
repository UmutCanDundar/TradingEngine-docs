# Risk Engine — Unit Tests

## Overview

Unit tests for `RiskEngine` — the component responsible for two distinct paths:

- **Update path** (`update_risk()`) — processes incoming order lifecycle events (new, partial fill, execution, cancel) from `store_to_risk`, updating `AccountRisk`, `SymbolRisk`, and `OrderRisk` structs with exposure, position, PnL, and margin calculations.
- **Check path** (`check_risk()`) — validates outbound orders from strategy against all configured risk limits before forwarding to the builder, returning structured reject reasons for any violation.

Tests are written with **Google Test (gtest)**. The test is split into two sequential sections within a single test case, sharing the same `RiskEngine` instance.

---

## Test Environment

| Component | Role |
|---|---|
| `RiskEngine` | System under test |
| `OrderManager` | Used for symbol metadata initialization |
| `MarketBook` | Updated alongside risk engine for ITCH messages |
| `Limits` | Provides account and symbol limit configurations |
| `store_to_risk` | Input queue for update path — `Order*` pushed by test |
| `strategy_to_risk` | Input queue for check path — `Order*` pushed by test |
| `risk_to_strategy` | Output queue — `OrderWithRejectReason` popped for verification |
| `risk_to_builder` | Output queue — approved orders forwarded here |

**Risk struct layout verified per update:**

| Struct | Fields tracked |
|---|---|
| `AccountRisk` | `current_exposure`, `positional_exposure`, `used_margin`, `balance`, `current_leverage`, `daily_realized_pnl`, `total_unrealized_pnl`, `open_orders_count` |
| `SymbolRisk` | `open_orders_count`, `pending_notional_scaled`, `net_position`, `cost_basis_scaled`, `unrealized_pnl`, `best_bid`, `best_ask` |
| `OrderRisk` | `price`, `remaining_qty`, `status`, `symbol_index`, `side`, `active` |

---

## Test Cases

### RiskEngineTest.MixedMessageTraffic

End-to-end test of both risk paths across BIST and NASDAQ, covering full order lifecycle and all configured risk limit checks.

---

## Part 1 — Update Risk

15 orders processed through `update_risk()` in sequence. BIST (GARAN) and NASDAQ (AAPL) run in parallel — each venue has independent `AccountRisk` and `SymbolRisk` state.

**Scaling notes:** `remaining_qty` in `OrderRisk` is stored in round lots (quantity / round_lot_size). `current_leverage` uses a 10,000 scale factor. Fee deductions use a 1,000,000 scale factor.

---

#### Message 1 — FIX New Order (GARAN, buy 500@15000)

`update_risk()` registers a new pending order. No position yet — only exposure and margin.

**AccountRisk (BIST):**

| Field | Expected | Notes |
|---|---|---|
| `current_exposure` | 7,500,000 | price × qty × side_mult |
| `positional_exposure` | 0 | No fill yet |
| `used_margin` | 7,500,000 | abs(notional) |
| `balance` | 10,000,000 | Unchanged |
| `current_leverage` | 7,500 | notional × 10,000 / balance |
| `open_orders_count` | 1 | |

**SymbolRisk (GARAN):**

| Field | Expected |
|---|---|
| `open_orders_count` | 1 |
| `pending_notional_scaled` | 7,500,000 |
| `net_position` | 0 |
| `best_bid` | 0 |
| `best_ask` | 0 |

**OrderRisk:** price=15000, remaining_qty=50 (round lots), status=New, active=true

---

#### Message 2 — BIST OUCH Order Accepted (GARAN, buy 100@10000)

Second pending order added. notional_bist accumulates.

**AccountRisk (BIST):**

| Field | Expected |
|---|---|
| `current_exposure` | 8,500,000 |
| `used_margin` | 8,500,000 |
| `open_orders_count` | 2 |

**SymbolRisk:** `open_orders_count`=2, `pending_notional_scaled`=8,500,000

**OrderRisk:** price=10000, remaining_qty=10 (round lots), status=New, active=true

---

#### Message 3 — BIST ITCH Add Order (GARAN, 1000@10000 Buy)

Market order — not our order. Updates `SymbolRisk.best_bid` via market data path.

**SymbolRisk:** `best_bid`=10000, `best_ask`=0 — no PnL change, no position change.

---

#### Message 4 — NASDAQ OUCH Order Accepted (AAPL, buy 1000@10000)

First NASDAQ order. Independent `AccountRisk` and `SymbolRisk` for NASDAQ venue.

**AccountRisk (NASDAQ):**

| Field | Expected |
|---|---|
| `current_exposure` | 10,000,000 |
| `used_margin` | 10,000,000 |
| `balance` | 10,000,000 |
| `current_leverage` | 10,000 |
| `open_orders_count` | 1 |

**OrderRisk:** price=10000, remaining_qty=25 (round lots), status=New, active=true

---

#### Message 5 — NASDAQ ITCH Add Order (AAPL, 100@10000 Buy)

Market order. Updates NASDAQ `best_bid`=10000. Net position and PnL remain 0 — no fill yet.

---

#### Message 6 — FIX Partial Fill (GARAN, 200 filled @15000)

First fill event. Balance decreases by filled nominal + maker fee. Position opens.

**AccountRisk (BIST):**

| Field | Expected | Notes |
|---|---|---|
| `positional_exposure` | 3,000,000 | 200 × 15000 |
| `used_margin` | 5,500,000 | notional − filled nominal |
| `balance` | < 10,000,000 | Deducted by nominal + fee |
| `open_orders_count` | 2 | FIX + OUCH still open |

**SymbolRisk:** `net_position`=200, `pending_notional_scaled`=5,500,000, `cost_basis_scaled`=3,000,000

**OrderRisk (FIX):** remaining_qty=30 (round lots), status=Partial

---

#### Message 7 — BIST ITCH Order Executed (GARAN market order, 60 executed)

Market order partial fill. Updates `MarketBook`. Unrealized PnL calculated against `best_bid`.

Since average entry price (cost_basis / net_position) is 15000 and best_bid is 10000:

`unrealized_pnl = (best_bid − avg_price) × net_position = (10000 − 15000) × 200 = −1,000,000`

**AccountRisk:** `total_unrealized_pnl` = pnl_bist (negative)

**SymbolRisk:** `unrealized_pnl` = pnl_bist, `best_bid`=10000

---

#### Message 8 — BIST OUCH Order Executed (GARAN, partial fill)

Second fill on OUCH order. Balance decreases further. net_position and cost_basis accumulate.

**OrderRisk (OUCH):** remaining_qty=4 (round lots), status=Partial, active=true

---

#### Message 9 — NASDAQ ITCH Order Executed (AAPL market order, 60 executed)

Market order execution. NASDAQ ITCH order is not our order — net_position stays 0, PnL stays 0.

**SymbolRisk (AAPL):** `net_position`=0, `unrealized_pnl`=0, `cost_basis_scaled`=0

---

#### Message 10 — NASDAQ OUCH Order Executed (AAPL, partial fill)

Our NASDAQ order partially filled.

**AccountRisk (NASDAQ):** `positional_exposure`=filled nominal, `open_orders_count`=1

**OrderRisk (NASDAQ OUCH):** remaining_qty=15 (round lots), status=Partial, active=true

---

#### Message 11 — BIST ITCH Order Delete (GARAN market order)

Market order removed from book. `best_bid` drops to 0. Unrealized PnL value preserved in SymbolRisk — risk engine does not recalculate on book delete, only on new market data.

**SymbolRisk:** `best_bid`=0, `best_ask`=0, `unrealized_pnl`=pnl_bist (unchanged)

---

#### Message 12 — BIST OUCH Order Cancelled (GARAN)

OUCH order cancelled with remaining quantity. `pending_notional` reduced by cancelled portion. `OrderRisk` entry removed from map.

**AccountRisk:** `open_orders_count`=1 (only FIX remains)

**SymbolRisk:** `pending_notional_scaled` reduced, `open_orders_count`=1

**OrderRisk:** `nullptr` — confirmed removed from map

---

#### Message 13 — NASDAQ ITCH Order Delete (AAPL market order)

Market order removed. NASDAQ `best_bid`=0. NASDAQ net_position and PnL unchanged.

---

#### Message 14 — NASDAQ OUCH Order Cancelled (AAPL)

NASDAQ OUCH order fully cancelled. All pending notional cleared.

**AccountRisk (NASDAQ):** `used_margin`=0, `open_orders_count`=0

**SymbolRisk:** `pending_notional_scaled`=0, `open_orders_count`=0

**OrderRisk:** `nullptr` — confirmed removed

---

#### Message 15 — FIX Cancel Confirm (GARAN)

FIX order cancelled. Remaining pending notional removed.

**AccountRisk (BIST):** `open_orders_count`=0

**SymbolRisk:** `open_orders_count`=0

**OrderRisk:** `nullptr` — confirmed removed

---

## Part 2 — Check Risk

Risk state reset to clean baseline before check tests begin. Tick size table initialized (GARAN: tick size 10, range 0–100M). `best_bid`=10000, `best_ask`=10000.

15 orders tested against `check_risk()`. Each order is designed to trigger a specific limit violation. One order passes all checks.

---

#### Check 1 — Pass

Valid order within all limits. `risk_to_strategy` receives `rej_order.order == nullptr` — order forwarded to builder, not rejected.

---

#### Check 2 — `MinOrderSizeExceeded`

Order quantity below the configured minimum order size for the symbol.

---

#### Check 3 — `MaxOrderSizeExceeded`

Order quantity above the configured maximum order size for the symbol.

---

#### Check 4 — `InvalidLotSize`

Order quantity is not a multiple of the symbol's round lot size.

---

#### Check 5 — `MaxOrderNotionalLimitExceeded` + `InvalidPriceRange`

`best_bid` set to 180,000 before check. Order price deviates beyond the allowed range from best bid, and the resulting notional exceeds the per-order notional limit. Both flags set simultaneously in the reject bitmask.

---

#### Check 6 — `PriceTickInvalid`

Order price is not aligned to the tick size (tick size = 10, price not divisible by 10).

---

#### Check 7 — `FatFingerCheckFailed`

Order price deviates beyond the fat finger threshold from the reference price.

---

#### Check 8 — `MaxPositionLimitExceeded`

`net_position` set to 960 before check — the incoming order would push position beyond the configured symbol position limit.

---

#### Check 9 — `MaxNotionalLimitExceeded`

`current_exposure` set to 49,900,000 before check — the incoming order's notional would push account exposure beyond the configured notional limit.

---

#### Check 10 — `AccountBalanceInsufficient` + `FreeMarginInsufficient` + `MaxLeverageExceeded`

`balance` set to 10,000 before check. Three limits triggered simultaneously — insufficient balance, free margin below required level, and resulting leverage above maximum. All three flags set in reject bitmask.

---

#### Check 11 — `MaxOpenOrdersReachedAccount`

`open_orders_count` set to 5 at account level — at the configured maximum. New order rejected.

---

#### Check 12 — `MaxOpenOrdersReachedSymbol`

`open_orders_count` set to 3 at symbol level — at the configured maximum for this symbol.

---

#### Check 13 — `SelfTradeDetected`

An `OrderRisk` entry is manually inserted at the same price, side, symbol, and venue as the incoming order before the check. Self-trade detection identifies the matching resting order.

---

#### Check 14 — `UnrealizedLossLimitExceeded`

Unrealized PnL on the position (best_bid below avg entry) has breached the configured unrealized loss limit.

---

#### Check 15 — `RealizedLossLimitExceeded`

`avg_entry_price` set to 30,000 before check. Closing at current price would realize a loss beyond the configured daily realized loss limit.

---

#### Check 16 — `MaxOrderRateLimitExceededSymbol` + `MaxOrderRateLimitExceededAccount`

Checks whether the number of orders sent within a predefined time window exceeds the allowed limit. Implemented using rate-limit token buckets.

---

## Results

```
Running main() from googletest/src/gtest_main.cc
[==========] Running 1 test from 1 test suite.
[----------] Global test environment set-up.
[----------] 1 test from RiskEngineTest
[ RUN      ] RiskEngineTest.MixedMessageTraffic
[       OK ] RiskEngineTest.MixedMessageTraffic (21 ms)
[----------] 1 test from RiskEngineTest (21 ms total)
[----------] Global test environment tear-down
[==========] 1 test from 1 test suite ran. (21 ms total)
[  PASSED  ] 1 test.
```

---

## Notes

- BIST and NASDAQ venue state is fully independent — `AccountRisk`, `SymbolRisk`, and `OrderRisk` maps are per-venue. Cross-venue leakage would be caught immediately by the sequential assertions.
- Multiple reject flags in a single bitmask (checks 5, 10) confirm that `check_risk()` evaluates all limits independently and accumulates all violations before returning — it does not short-circuit on the first failure.
- The 21ms runtime reflects purely in-process computation with no I/O or network overhead.
