# Parser — Unit Tests

## Overview

Unit tests for `Parser_Dispatch` — the component responsible for routing inbound `RxPacket`s to the correct protocol parser (FIX, OUCH BIST, OUCH NASDAQ, ITCH BIST, ITCH NASDAQ), reassembling fragmented messages, and pushing parsed `MessageWithVenue` structs to the downstream queue.

Tests are written with **Google Test (gtest)**. The test feeds raw `RxPacket` objects directly into the parser queue, bypassing `NetworkIO`, to test parsing logic in isolation.

---

## Test Environment

| Component | Role |
|---|---|
| `Parser_Dispatch` | System under test |
| `NetworkIO::SBTPacketHandler` | Used for OUCH SoupBinTCP framing before dispatch |
| `receiver_to_parser` | Input queue — `RxPacket*` pushed by test |
| `parser_to_store` | Output queue — parsed `MessageWithVenue` popped for verification |
| `SessionManager` | Tracks session and sequence state |

---

## Test Cases

### ParserDispatchTest.MixedProtocolTraffic

End-to-end parsing of interleaved multi-protocol traffic across OUCH BIST, ITCH BIST, FIX, and ITCH NASDAQ. Tests correct fragmentation reassembly, protocol routing, field extraction, and output ordering under realistic mixed-protocol conditions.

---

#### Input Dataset

13 raw `RxPacket`s are injected in the following order — deliberately interleaved to stress protocol boundary handling:

| # | Packet | Protocol | Type |
|---|---|---|---|
| 1 | `ouch_bist_RxPacket_partial_1` | OUCH BIST | Fragment 1/5 |
| 2 | `ouch_bist_RxPacket_partial_2` | OUCH BIST | Fragment 2/5 |
| 3 | `itch_bist_RxPacket_single_1` | ITCH BIST | Complete |
| 4 | `fix_RxPacket_partial_1` | FIX | Fragment 1/6 |
| 5 | `fix_RxPacket_partial_2` | FIX | Fragment 2/6 |
| 6 | `ouch_bist_RxPacket_partial_3` | OUCH BIST | Fragment 3/5 |
| 7 | `ouch_bist_RxPacket_partial_4` | OUCH BIST | Fragment 4/5 |
| 8 | `fix_RxPacket_partial_3` | FIX | Fragment 3/6 |
| 9 | `fix_RxPacket_partial_4` | FIX | Fragment 4/6 |
| 10 | `ouch_bist_RxPacket_partial_5` | OUCH BIST | Fragment 5/5 |
| 11 | `itch_nasdaq_RxPacket_single_1` | ITCH NASDAQ | Complete |
| 12 | `fix_RxPacket_partial_5` | FIX | Fragment 5/6 |
| 13 | `fix_RxPacket_partial_6` | FIX | Fragment 6/6 |

OUCH packets pass through `SBTPacketHandler` for SoupBinTCP framing before entering the parser queue. FIX and ITCH packets are pushed directly. `dispatch()` is called 12 times to process all input.

---

#### Expected Output — 12 Parsed Messages

Messages are verified in the order they arrive on `parser_to_store`:

---

**Message 1 — OUCH BIST: Order Accepted (`'A'`)**

Reassembled from 5 fragments. BIST OUCH `OUCHOrderAcceptedMessage`.

| Field | Expected | Notes |
|---|---|---|
| `message_type` | `'A'` | Accepted |
| `timestamp` | 1 | |
| `order_book_id` | 2 | |
| `side` | `'B'` | Buy |
| `order_id` | 1 | |
| `quantity` | 2000 | |
| `price` | 10000 | |
| `order_token` | `"TOKEN000000002"` | 14 chars |
| `time_in_force` | 0 | |
| `order_state` | 1 | |
| `client_account` | 16× `'\0'` | Empty |
| `customer_info` | 15× `'\0'` | Empty |
| `exchange_info` | 32× `'\0'` | Empty |

---

**Message 2 — ITCH BIST: Order Book Directory (`'R'`)**

Single complete packet. BIST ITCH `ITCHOrderBookDirectoryMessage`.

| Field | Expected | Notes |
|---|---|---|
| `message_type` | `'R'` | |
| `timestamp_ns` | 48 | |
| `order_book_id` | 3 | |
| `financial_product` | 5 | |
| `decimals_in_price` | 2 | |
| `round_lot_size` | 1 | |
| `trading_currency` | `"TRY"` | |
| `symbol` | `"GARAN"` | bytes 0–4 |
| `isin` | starts `"TRG"` | bytes 0–2 |
| `ranking_type` | 1 | |

---

**Message 3 — FIX: New Order (`'D'`)**

Reassembled from FIX fragments (partial 1–2 contribute). Sequence number 1.

| Field | Expected | Notes |
|---|---|---|
| `msg_type` | `'D'` | New Order Single |
| `seqnum` | 1 | |
| `quantity` | 100 | |
| `price` | 10000 | |
| `side` | 1 | Buy |
| `symbol` | `"GARAN"` | |
| `cl_ord_id` | `"O1"` | |
| `order_id` | `""` | Not yet assigned |
| `last_qty` | 0 | |
| `cxl_rej_reason` | 255 | Default unset |

---

**Message 4 — FIX: Cancel Request (`'F'`)**

Continuation of FIX fragment reassembly. Sequence number 2.

| Field | Expected | Notes |
|---|---|---|
| `msg_type` | `'F'` | Order Cancel Request |
| `seqnum` | 2 | |
| `symbol` | `"TUPRS"` | Different symbol |
| `cl_ord_id` | `"O2"` | |
| `orig_cl_ord_id` | `"O1"` | References previous order |
| `price` | 0 | Not set on cancel |
| `side` | 0 | Not set on cancel |

---

**Message 5 — OUCH BIST: Order Cancelled (`'C'`)**

Fragments 3–5 complete the OUCH sequence.

| Field | Expected | Notes |
|---|---|---|
| `message_type` | `'C'` | Cancelled |
| `timestamp` | 2 | |
| `order_book_id` | 2 | |
| `side` | `'B'` | Buy |
| `order_id` | 1 | Same order as msg1 |
| `order_token` | `"TOKEN000000002"` | |
| `reason` | 0 | |

---

**Message 6 — OUCH BIST: Order Executed (`'E'`)**

| Field | Expected | Notes |
|---|---|---|
| `message_type` | `'E'` | Executed |
| `timestamp` | 3 | |
| `order_book_id` | 2 | |
| `traded_quantity` | 1000 | Partial fill |
| `trade_price` | 10000 | |
| `order_token` | `"TOKEN000000002"` | |
| `match_id` | 12× `'\0'` | Empty |
| `reserved` | 16× `'\0'` | Empty |

---

**Message 7 — FIX: Execution Report (`'8'`)**

FIX fragments 3–4 produce this message. Sequence number 3.

| Field | Expected | Notes |
|---|---|---|
| `msg_type` | `'8'` | Execution Report |
| `seqnum` | 3 | |
| `last_qty` | 100 | Fill quantity |
| `last_price` | 10000 | Fill price |
| `cl_ord_id` | `"O1"` | |
| `ord_status` | `'2'` | Filled |
| `exec_type` | `'F'` | Fill |

---

**Message 8 — OUCH BIST: Order Executed (`'E'`) — duplicate**

Second execution event on the same order. Fields identical to message 6 — confirms parser correctly handles repeated execution messages on the same order token.

---

**Message 9 — ITCH NASDAQ: Stock Directory (`'R'`)**

Single complete NASDAQ ITCH packet. `ITCHStockDirectoryMessage`.

| Field | Expected | Notes |
|---|---|---|
| `message_type` | `'R'` | |
| `stock_locate` | 1 | |
| `tracking_number` | 2 | |
| `timestamp` | 48 | |
| `stock` | `"AAPL"` | bytes 0–3 |
| `round_lot_size` | 100 | |
| `market_category` | `'Q'` | NASDAQ Global Select |
| `financial_status_indicator` | `'N'` | Normal |
| `round_lots_only` | `'N'` | |
| `issue_classification` | `'C'` | |
| `authenticity` | `'P'` | Production |
| `etp_leverage_factor` | 1 | |

---

**Message 10 - 11 - 12  — FIX: Execution Report (`'8'`) — FIX: New Order (`'D'`)  — FIX: Cancel Request (`'F'`) **

FIX fragments 4–5–6 produce this. Same fill fields as previous FIX messages but in not sequentially order — confirms the receiving out of sequence messages are delivered to downstream in correct order.

---

## Results

```
Running main() from googletest/src/gtest_main.cc
[==========] Running 1 test from 1 test suite.
[----------] Global test environment set-up.
[----------] 1 test from ParserDispatchTest
[ RUN      ] ParserDispatchTest.MixedProtocolTraffic
[       OK ] ParserDispatchTest.MixedProtocolTraffic (223 ms)
[----------] 1 test from ParserDispatchTest (223 ms total)
[----------] Global test environment tear-down
[==========] 1 test from 1 test suite ran. (223 ms total)
[  PASSED  ] 1 test.
```

---

## Notes

- The 13-packet input is deliberately interleaved — OUCH fragments are split across FIX and ITCH packets to verify that each protocol parser maintains independent reassembly state.

