## Risk Benchmark

### Scenarios

- **Update Single** — a single market data message updates
the risk profile; covers the minimal update path.
- **Update Flow** — a sequence of related market messages
processed in order: AddOrder followed by Execution and
Cancel. Reflects a complete order lifecycle update.
- **Check Single** — a single new order passes through risk
validation before being forwarded to the next pipeline stage
for exchange submission.

---

## Risk Update

| Scenario | Median (ns) | p50 cycles | p99 cycles | p999 cycles |
|----------|-------------|------------|------------|-------------|
| Single | 50.4 (0.14%) | 50 (0.00%) | 56 (1.13%) | 66 (11.89%) |
| Flow | 63.7 (0.12%) | 90 (0.00%) | 98 (0.98%) | 112 (12.99%) |

## Risk Check

| Scenario | Median (ns) | p50 cycles | p99 cycles | p999 cycles |
|----------|-------------|------------|------------|-------------|
| Single | 131 (1.34%) | 214 (0.30%) | 342 (0.82%) | 390 (2.65%) |

---

## Risk Summary — Notes

**Risk Update is the lightest component in the pipeline.**
Single update completes in 50 cycles — the lowest p50 across
all benchmarked components. This is expected: update is a
targeted write to a pre-allocated pool with no lookup
overhead. Flow at 90 cycles covers a full order lifecycle
update with minimal overhead increase, suggesting efficient
state management with no significant contention between
consecutive updates.

**Risk Check costs ~4x more than Risk Update.**
Check at 214 cycles vs Update at 50 cycles. Check validates
position limits and order constraints against the current
risk state — a read-heavy operation involving multiple field
comparisons. An open question for production sizing: is 214
cycles on an E-core within the acceptable latency budget for
the check path? This depends on the order submission rate
and acceptable risk gate overhead.

Part of the gap may also reflect the core assignment
difference: Update runs on a P-core while Check runs on an
E-core, which has lower single-thread throughput. See
[Threading Model](../architecture.md#threading-model) for
core assignment details.

**Risk Update and Risk Check run on separate cores by design.**
Update is write-heavy and runs on a P-core. Check is
read-heavy and runs on an E-core. Placing them on the same
core would cause L1/L2 cache contention — writes from Update
would invalidate cache lines read by Check.
