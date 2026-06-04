# Network — Unit Tests

## Overview

Unit tests for `NetworkIO` — the component responsible for managing all socket connections, handling the epoll event loop, sending outbound packets, and delivering inbound packets to the parser queue as `RxPacket` structs.

Tests are written with **Google Test (gtest)**. The test suite spins up fake TCP servers and UDP senders on localhost to exercise the full network I/O path without requiring real exchange connectivity.

---

## Test Environment

| Component | Role |
|---|---|
| `FakeTCPServer` | Accepts connections, receives sent data, sends test packets back — one instance per TCP session |
| `FakeUDPSender` | Sends UDP datagrams to NetworkIO's bound ports |
| `NetworkIO` | System under test — runs `recv_send()` on a dedicated thread |
| `Builder_Dispatch` | Used to generate outbound `TxPacket`s from test orders |
| `SessionManager` | Tracks login state per session |

Five sessions are simulated:

| Session | Port | Transport | Venue | Protocol |
|---|---|---|---|---|
| 0 | 9876 | TCP | BIST | FIX |
| 1 | 1234 | TCP | BIST | OUCH |
| 2 | 4321 | TCP | NASDAQ | OUCH |
| 3 | 5000 | UDP | BIST | ITCH |
| 4 | 6000 | UDP | NASDAQ | ITCH |

---

## Test Cases

### NetworkIOTest.NetworkIOTestAll

End-to-end integration of the full NetworkIO path — connection establishment, session login, outbound order sending, and inbound packet delivery across all five sessions simultaneously.

---

#### 1. Socket Initialization

Verifies that `NetworkIO` correctly initializes socket state for each TCP session before the event loop starts.

**Checked fields:**

- `connection_pending` is `true` for all three TCP sessions — sockets have called `::connect()` and are waiting for the handshake (EINPROGRESS state)
- `epoll_data` encoding is correct for each session: `session_index | venue << 8 | protocol << 16`

| Session | Expected epoll_data encoding |
|---|---|
| 0 (FIX BIST) | `0 \| BIST << 8 \| FIX << 16` |
| 1 (OUCH BIST) | `1 \| BIST << 8 \| OUCH << 16` |
| 2 (OUCH NASDAQ) | `2 \| NASDAQ << 8 \| OUCH << 16` |

---

#### 2. TCP Handshake

Verifies that all three TCP connections complete successfully once `recv_send()` starts processing epoll events.

NetworkIO handles `EINPROGRESS` on the epoll loop — on `EPOLLOUT` for a pending socket it completes the handshake and transitions the socket to connected state. The test waits for all three fake servers to receive the accepted client connection before proceeding.

---

#### 3. Login Sequence

Verifies that NetworkIO sends login request messages to each server immediately after connection, and that sessions transition to logged-in state after receiving login acknowledgements.

- FIX session (0): `SessionState` is set manually — FIX login ack is handled by `Parser_FIX`, not `SBTPacketHandler`, so the test simulates this by calling `setSessionLoggedIn(0, true)` directly
- OUCH BIST (1) and OUCH NASDAQ (2): fake servers send back `sbt_login_accepted_bist` and `sbt_login_accepted_nq` — `SBTPacketHandler` processes these and updates `SessionManager`

All three sessions reach logged-in state before the traffic phase begins.

---

#### 4. Outbound Order Sending (Tx Path)

Verifies that NetworkIO correctly transmits outbound packets built by `Builder_Dispatch`.

Three test orders are pushed to the `risk_to_builder` queue and dispatched:

| Order | Venue | Protocol |
|---|---|---|
| `fix_new_order` | BIST | FIX |
| `ouch_new_order` | BIST | OUCH |
| `NQouch_new_order` | NASDAQ | OUCH |

The test waits until each fake TCP server receives exactly 2 messages — the login request sent on connection plus the order packet. Confirms the Tx path from `builder_to_sender` queue through `NetworkIO::send()` to the socket.

---

#### 5. Inbound Packet Delivery (Rx Path)

Verifies that NetworkIO correctly receives inbound market data and order acknowledgements, wraps them as `RxPacket` structs with correct venue/protocol tagging, and delivers them to the `receiver_to_parser` queue.

Five test packets are injected by the fake servers/senders:

| Packet | Transport | Source | Expected bytes |
|---|---|---|---|
| FIX order ack (ExecutionReport) | TCP | FIX server | 125 bytes |
| OUCH BIST order ack | TCP | OUCH BIST server | 140 bytes |
| OUCH NASDAQ order ack | TCP | OUCH NASDAQ server | 67 bytes |
| ITCH BIST add order | UDP | ITCH BIST sender | 45 bytes |
| ITCH NASDAQ add order | UDP | ITCH NASDAQ sender | 36 bytes |

The test collects all 5 `RxPacket*` from the queue, then validates each one by venue+protocol routing:

**FIX order ack (125 bytes):**

| Field | Byte offset | Expected | Meaning |
|---|---|---|---|
| `rxPkt->len` | — | 125 | Total packet length |
| `data[19]` | 19 | `'8'` | `35=MsgType` = ExecutionReport |
| `data[78]` | 78 | `'G'` | `55=Symbol` starts with G (GARAN) |
| `data[87]` | 87 | `'1'` | `54=Side` = Buy |
| `data[92]` | 92 | `'1'` | `38=Qty` starts with 1 (100) |
| `data[110]` | 110 | `'0'` | `39=OrdStatus` = New |
| `venue` | — | `BIST` | Correctly tagged |
| `protocol` | — | `FIX` | Correctly tagged |

**OUCH BIST order ack (140 bytes):**

| Field | Byte offset | Expected | Meaning |
|---|---|---|---|
| `rxPkt->len` | — | 140 | Total packet length |
| `data[3]` | 3 | `'A'` | message_type = Accepted |
| `data[30]` | 30 | `'B'` | side = Buy |
| `data[46]` | 46 | `0x64` | quantity last byte = 100 |
| `data[49]` | 49 | `0x27` | price high byte |
| `data[50]` | 50 | `0x10` | price = 10000 |
| `venue` | — | `BIST` | Correctly tagged |
| `protocol` | — | `OUCH` | Correctly tagged |

**OUCH NASDAQ order ack (67 bytes):**

| Field | Byte offset | Expected | Meaning |
|---|---|---|---|
| `rxPkt->len` | — | 67 | Total packet length |
| `data[3]` | 3 | `'A'` | message_type = Accepted |
| `data[16]` | 16 | `'S'` | side = Sell |
| `data[20]` | 20 | `0x64` | quantity last byte = 100 |
| `venue` | — | `NASDAQ` | Correctly tagged |
| `protocol` | — | `OUCH` | Correctly tagged |

**ITCH BIST add order (45 bytes):**

| Field | Byte offset | Expected | Meaning |
|---|---|---|---|
| `rxPkt->len` | — | 45 | Total packet length |
| `data[0]` | 0 | `'A'` | message_type = Add Order |
| `data[17]` | 17 | `'B'` | side = Buy |
| `data[29]` | 29 | `0x64` | quantity last byte = 100 |
| `data[30]` | 30 | `0x00` | price byte 0 |
| `data[33]` | 33 | `0x10` | price last byte |
| `venue` | — | `BIST` | Correctly tagged |
| `protocol` | — | `ITCH` | Correctly tagged |

**ITCH NASDAQ add order (36 bytes):**

| Field | Byte offset | Expected | Meaning |
|---|---|---|---|
| `rxPkt->len` | — | 36 | Total packet length |
| `data[0]` | 0 | `'A'` | message_type = Add Order |
| `data[19]` | 19 | `'S'` | side = Sell |
| `data[23]` | 23 | `0x64` | shares last byte = 100 |
| `data[24]` | 24 | `'A'` | stock[0] = A |
| `data[32]` | 32 | `0x05` | price byte 0 |
| `data[35]` | 35 | `0x00` | price last byte |
| `venue` | — | `NASDAQ` | Correctly tagged |
| `protocol` | — | `ITCH` | Correctly tagged |

---

## Results

```
Running main() from googletest/src/gtest_main.cc
[==========] Running 1 test from 1 test suite.
[----------] Global test environment set-up.
[----------] 1 test from NetworkIOTest
[ RUN      ] NetworkIOTest.NetworkIOTestAll
[       OK ] NetworkIOTest.NetworkIOTestAll (237 ms)
[----------] 1 test from NetworkIOTest (237 ms total)
[----------] Global test environment tear-down
[==========] 1 test from 1 test suite ran. (237 ms total)
[  PASSED  ] 1 test.
```

---

## Notes

- The 237ms runtime includes TCP handshake latency on loopback, login sequence synchronization, and condition variable wait overhead — not representative of production packet processing latency.

