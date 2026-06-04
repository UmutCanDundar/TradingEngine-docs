## Builder Dispatch — Mixed Traffic

The dispatch benchmark reflects realistic production traffic —
build requests from all protocols and venues are interleaved
and processed through a single builder dispatch thread. The
dataset contains FIX, OUCH BIST, and OUCH NASDAQ orders of
varying types processed in sequence.

| Scenario | Median (ns) | p50 cycles | p99 cycles | p999 cycles |
|----------|-------------|------------|------------|-------------|
| Mixed Traffic | 262 (0.93%) | 664 (0.10%) | 814 (0.91%) | 985 (114.47%) |

> 3 orders per iteration (1 per protocol × 3 protocols).
> Average cost per order: 221 cycles (664 / 3).

---

## Builder Benchmark

### Scenarios

- **Single** — a single new order processed in isolation;
covers the minimal build path for that protocol.
- **Mixed** — a sequence of varying order types processed
in order: new order, replace, modify, cancel, and additional
new orders. Reflects realistic production traffic where
different order operations arrive in sequence.

---

## FIX Builder

| Scenario | Median (ns) | p50 cycles | p99 cycles | p999 cycles |
|----------|-------------|------------|------------|-------------|
| Single | 175 (0.10%) | 440 (0.00%) | 498 (0.27%) | 583 (5.10%) |

---

## OUCH BIST Builder

| Scenario | Median (ns) | p50 cycles | p99 cycles | p999 cycles |
|----------|-------------|------------|------------|-------------|
| Single | 30.9 (0.08%) | 30 (0.00%) | 34 (0.00%) | 40 (4.71%) |
| Mixed | 55.3 (0.06%) | 92 (0.00%) | 104 (1.33%) | 136 (1.03%) |

---

## OUCH NASDAQ Builder

| Scenario | Median (ns) | p50 cycles | p99 cycles | p999 cycles |
|----------|-------------|------------|------------|-------------|
| Single | 31.5 (0.10%) | 32 (0.00%) | 34 (1.85%) | 41 (5.27%) |
| Mixed | 57.3 (0.16%) | 102 (0.00%) | 146 (0.70%) | 152 (1.74%) |

---

## Builder Summary — Cost Breakdown

Mixed traffic dataset contains one FIX, one OUCH BIST, and
one OUCH NASDAQ new order — all three processed per iteration.

| Stage | cycles |
|-------|--------|
| FIX build — Single | 440 |
| OUCH BIST build — Single | 30 |
| OUCH NASDAQ build — Single | 32 |
| Total build cost | 502 |
| Builder dispatch full pipeline | 664 |
| Dispatch overhead | 162 |
| Build % of total | 75.6% |
| Dispatch overhead % | 24.4% |
| Dispatch overhead per order (3 orders/iter) | 54 cycles (8.1%) |

**Notes:** Build cost accounts for ~76% of total dispatch cost —
the inverse of the parser where dispatch overhead dominates
at 73–99%. This means the builder's optimization target is
FIX serialization itself, not the dispatch path.

FIX message construction involves field-by-field ASCII
encoding — a known cost of the protocol. At 440 cycles, FIX
costs ~15x more than OUCH binary formats at 30–32 cycles.
For high-order-rate scenarios, FIX serialization is the
primary bottleneck in the build path.

OUCH binary formats at ~30 cycles each reflect the minimal
cost of fixed-length struct serialization with no field
encoding overhead.

