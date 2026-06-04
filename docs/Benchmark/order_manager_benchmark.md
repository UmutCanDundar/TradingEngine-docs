## Order Manager Benchmark

### Scenarios

- **Single** — a single message processed in isolation;
covers the minimal order manager path for that message type.
- **Flow** — a sequence of related messages processed in
order; for example FIX: NewOrder followed by ExecutionReport 
and Cancel. Reflects realistic message flow through the order 
manager.
- **Mixed Traffic** — 3 messages per protocol interleaved
in random order across all protocols and venues. Reflects
realistic production traffic patterns where messages from
different protocols arrive concurrently.

---

## Order Manager Dispatch — Mixed Traffic

The dispatch benchmark processes a mixed sequence of messages
from all protocols and venues through the order manager in a
single thread.

| Scenario | Median (ns) | p50 cycles | p99 cycles | p999 cycles |
|----------|-------------|------------|------------|-------------|
| Mixed Traffic | 2570 (3.51%) | 7067 (4.07%) | 8585 (7.67%) | 178638 (8.72%) |

> 15 messages per iteration (3 per protocol × 5 protocols).
> Average cost per message: 471 cycles (7067 / 15).

---

## FIX Order Manager

| Scenario | Median (ns) | p50 cycles | p99 cycles | p999 cycles |
|----------|-------------|------------|------------|-------------|
| Single | 71.7 (0.08%) | 120 (0.00%) | 134 (1.59%) | 165 (2.46%) |
| Flow | 107 (0.06%) | 220 (0.00%) | 280 (0.00%) | 294 (3.33%) |

---

## ITCH BIST Order Manager

| Scenario | Median (ns) | p50 cycles | p99 cycles | p999 cycles |
|----------|-------------|------------|------------|-------------|
| Single | 80.8 (0.90%) | 148 (1.38%) | 166 (1.14%) | 192 (4.75%) |
| Flow | 148 (1.34%) | 342 (1.58%) | 405 (1.96%) | 469 (4.36%) |

---

## ITCH NASDAQ Order Manager

| Scenario | Median (ns) | p50 cycles | p99 cycles | p999 cycles |
|----------|-------------|------------|------------|-------------|
| Single | 69.2 (0.48%) | 114 (0.85%) | 132 (2.52%) | 162 (1.43%) |
| Flow | 143 (1.04%) | 324 (1.37%) | 400 (1.06%) | 454 (3.56%) |

---

## OUCH BIST Order Manager

| Scenario | Median (ns) | p50 cycles | p99 cycles | p999 cycles |
|----------|-------------|------------|------------|-------------|
| Single | 168 (0.61%) | 210 (1.35%) | 519 (0.80%) | 59361 (48.51%) |
| Flow | 202 (0.42%) | 312 (0.70%) | 632 (3.19%) | 80404 (35.86%) |

---

## OUCH NASDAQ Order Manager

| Scenario | Median (ns) | p50 cycles | p99 cycles | p999 cycles |
|----------|-------------|------------|------------|-------------|
| Single | 168 (0.59%) | 214 (1.46%) | 516 (0.66%) | 71257 (36.60%) |
| Flow | 208 (0.60%) | 326 (1.04%) | 644 (2.63%) | 74328 (33.45%) |

---

## Order Manager Summary — Notes

**OUCH p999 anomaly.**
OUCH BIST and OUCH NQ show p999 values of 59k–80k cycles
against p50 values of 210–326 cycles — a 200–380x ratio.
FIX and ITCH p999/p50 ratios stay under 2x. OUCH order
manager handles acknowledgement matching against a pending
order table; occasional hash table probe depth variance or
cache miss on the awaiting-ack lookup is the suspected cause
of these tail spikes — not yet isolated. This warrants a
dedicated follow-up test with hash table access instrumented
separately.

**ITCH single is faster than FIX single.**
ITCH messages carry market data that follows a simpler
order manager path — no awaiting-ack table lookup required.
FIX single at 120 cycles includes the full ack-matching path.

**Flow cost is roughly 2x single across all protocols.**
Flow scenarios include both a new order and an execution
report in sequence. The near-linear scaling confirms the
order manager processes each message atomically in arrival
order with no batching or reordering — minimal shared state
overhead between consecutive messages.

