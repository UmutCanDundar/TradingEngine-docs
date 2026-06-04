## Parser Benchmark

## Parser Dispatch — Mixed Traffic

The dispatch benchmark reflects realistic production traffic — 
raw network packets from all protocols and venues are interleaved 
and processed through a single parser dispatch thread. The dataset 
contains both single-packet and partial (multi-packet) messages 
across FIX, OUCH BIST, OUCH NASDAQ, ITCH BIST, and ITCH NASDAQ.

| Scenario | Median (ns) | p50 cycles | p99 cycles | p999 cycles |
|----------|-------------|------------|------------|-------------|
| Mixed Traffic | 1585 (0.41%) | 4413 (0.39%) | 5426 (0.64%) | 11204 (1.74%) |

> 13 packets per iteration (variable per protocol × 5 protocols).
> Average cost per packet: 339 cycles (4413 / 13).

To isolate the contribution of the SPSC queue push, the same benchmark was re-run with queue push disabled:

| Scenario | Median (ns) | p50 cycles | p99 cycles | p999 cycles |
|----------|-------------|------------|------------|-------------|
| Mixed Traffic (no queue push) | 901 (1.07%) | 2510 (1.17%) | 2862 (2.01%) | 5740 (0.61%) |

Removing the queue push drops median latency from 1585 ns to 901 ns — a ~43% reduction — confirming that the SPSC queue push is the dominant cost in the dispatch path. This strengthens the case for replacing the current boost lockfree queue with a custom queue implementation tailored to this pipeline's access pattern.

---

## FIX Parser

The FIX parser layer consists of three measurable stages:

- **ParserFIX** — top-level class; receives raw packets, calls 
`nextFixMsg()` to extract messages, dispatches parsed structs 
to the Order Manager queue.
- **nextFixMsg()** — extracts a single FIX message boundary 
from a raw byte buffer; handles partial and multi-message packets.
- **parse()** — constructs the message struct from a complete 
FIX message boundary.

### ParserFIX — Full Pipeline

| Scenario | Median (ns) | p50 cycles | p99 cycles | p999 cycles |
|----------|-------------|------------|------------|-------------|
| Best (single msg) | 178 (0.16%) | 408 (0.40%) | 444 (0.98%) | 519 (12.84%) |
| Normal (full msg, single pkt) | 344 (1.04%) | 924 (6.42%) | 1169 (4.08%) | 1464 (9.22%) |
| Worst — seq in-order | 409 (2.04%) | 1075 (1.99%) | 1377 (3.74%) | 1718 (16.23%) |
| Worst — seq out-of-order | 388 (4.42%) | 953 (4.41%) | 1253 (4.76%) | 1438 (10.71%) |

### nextFixMsg() Function

| Scenario | Median (ns) | p50 cycles | p99 cycles | p999 cycles |
|----------|-------------|------------|------------|-------------|
| Best | 43.5 (0.08%) | 60 (0.00%) | 64 (1.59%) | 84 (1.89%) |
| Normal | 63.1 (0.10%) | 118 (0.00%) | 124 (0.00%) | 210 (1.25%) |
| Worst — seq in-order | 78.4 (0.11%) | 160 (0.53%) | 166 (0.58%) | 266 (1.01%) |

### parse() Function

| Message Type | Median (ns) | p50 cycles | p99 cycles | p999 cycles |
|--------------|-------------|------------|------------|-------------|
| NewOrder | 69.0 (0.06%) | 138 (0.00%) | 168 (0.38%) | 256 (1.85%) |
| ExecReport | 63.6 (0.13%) | 122 (0.00%) | 140 (0.74%) | 202 (1.77%) |

### FIX Summary — Cost Breakdown

Using **Normal** and **Worst (seq in-order)** scenarios.
`parse()` cost is taken as the average of NewOrder and ExecReport.

| Stage | Normal cycles | Worst cycles |
|-------|--------------|--------------|
| nextFixMsg() | 118 | 160 |
| parse() avg | 130 | 130 |
| nextFixMsg + parse | 248 | 290 |
| ParserFIX full pipeline | 924 | 1075 |
| Dispatch overhead | 676 | 785 |
| nextFixMsg % of total | 12.8% | 14.9% |
| parse % of total | 14.1% | 12.1% |
| Dispatch overhead % | 73.1% | 73.0% |

> Dispatch overhead includes queue push to Order Manager, 
> protocol routing, and buffer management.

**Notes:** Dispatch overhead accounts for ~73% of total pipeline 
cost in both scenarios — parse() and nextFixMsg() together 
represent under 30%. Queue push cost is therefore the primary 
optimization target for FIX parser latency reduction.

The seq out-of-order scenario costs slightly less than seq 
in-order despite higher complexity; with partial packets spread 
across non-contiguous boundaries, fewer messages complete per 
iteration, reducing queue push frequency.

p999 CV is elevated across all FIX scenarios — with only 10 
repetitions, a single outlier disproportionately inflates tail 
percentiles. Production-grade tail measurement requires a minimum 
of 100 repetitions for stable p999 values. p50 CV stays below 
2%, confirming hot path stability.

---

## ITCH BIST Parser

The ITCH BIST parser receives binary UDP packets. Each packet 
contains one or more complete fixed-length messages — partial 
messages do not occur over UDP. Message extraction is handled 
within the dispatch layer and is not measured separately; its 
cost is included in the full pipeline numbers.

### ParserITCH BIST — Full Pipeline

| Scenario | Median (ns) | p50 cycles | p99 cycles | p999 cycles |
|----------|-------------|------------|------------|-------------|
| Best (single msg) | 154 (0.42%) | 312 (1.14%) | 422 (0.72%) | 534 (1.74%) |
| Worst (multi-msg, single pkt) | 306 (0.23%) | 752 (0.41%) | 1044 (0.83%) | 1254 (9.57%) |

### parse() Function

| Message Type | Median (ns) | p50 cycles | p99 cycles | p999 cycles |
|--------------|-------------|------------|------------|-------------|
| OrderBookDir | 44.6 (0.08%) | 58 (0.00%) | 76 (0.83%) | 112 (0.87%) |
| AddOrder | 37.4 (0.11%) | 40 (0.00%) | 50 (0.00%) | 68 (1.23%) |
| Exec | 37.9 (0.05%) | 42 (0.00%) | 54 (0.00%) | 70 (0.90%) |

### ITCH BIST Summary — Cost Breakdown

| Stage | Best cycles | Worst cycles |
|-------|-------------|--------------|
| parse() — AddOrder | 40 | 40 |
| ParserITCH full pipeline | 312 | 752 |
| Dispatch overhead | 272 | 712 |
| parse % of total | 12.8% | 5.3% |
| Dispatch overhead % | 87.2% | 94.7% |

**Notes:** parse() is among the lightest across all protocols — 
fixed-length binary fields with no delimiter scanning. This is 
expected: a targeted struct copy with no lookup overhead.
Dispatch overhead grows from 87% in the best case to 95% in 
the worst, as multi-message packets require repeated routing 
and queue push operations while parse cost per message stays 
constant.

p999 CV elevation follows the same pattern as other components — 
with 10 repetitions a single outlier inflates tail percentiles. 
Production tail measurement requires minimum 100 repetitions. 
p50 CV stays below 2%.

---

## ITCH NASDAQ Parser

The ITCH NASDAQ parser follows the same structure as ITCH BIST. 
Binary UDP packets contain one or more complete fixed-length 
messages. Message extraction is handled within the dispatch 
layer and its cost is included in the full pipeline numbers.

### ParserITCH NASDAQ — Full Pipeline

| Scenario | Median (ns) | p50 cycles | p99 cycles | p999 cycles |
|----------|-------------|------------|------------|-------------|
| Best (single msg) | 150 (0.87%) | 312 (1.53%) | 371 (3.85%) | 421 (8.38%) |
| Worst (multi-msg, single pkt) | 307 (0.63%) | 758 (0.59%) | 967 (1.53%) | 1119 (13.92%) |

### parse() Function

| Message Type | Median (ns) | p50 cycles | p99 cycles | p999 cycles |
|--------------|-------------|------------|------------|-------------|
| StockDir | 37.5 (0.09%) | 44 (0.00%) | 56 (0.00%) | 72 (0.00%) |
| AddOrder | 35.1 (0.07%) | 36 (0.00%) | 42 (2.41%) | 62 (1.69%) |
| Exec | 36.4 (0.06%) | 40 (0.00%) | 48 (0.00%) | 60 (0.00%) |

### ITCH NASDAQ Summary — Cost Breakdown

| Stage | Best cycles | Worst cycles |
|-------|-------------|--------------|
| parse() — AddOrder | 36 | 36 |
| ParserITCH full pipeline | 312 | 758 |
| Dispatch overhead | 276 | 722 |
| parse % of total | 11.5% | 4.8% |
| Dispatch overhead % | 88.5% | 95.2% |

**Notes:** Results are near-identical to ITCH BIST — same 
dispatch structure, same UDP delivery model, marginally 
lighter parse() cost per message. The cost breakdown 
pattern is consistent: parse() is a small fraction of 
total pipeline cost in both scenarios.

To investigate whether batching queue pushes could reduce contention, an additional variant was benchmarked: all messages in a packet are parsed first, then pushed to the queue in a single batch operation rather than one push per message.

### ParserITCH NASDAQ — Batch Queue Push

| Scenario | Median (ns) | p50 cycles | p99 cycles | p999 cycles |
|----------|-------------|------------|------------|-------------|
| Best (single msg) | 154 (1.84%) | 318 (3.09%) | 349 (3.54%) | 426 (12.56%) |
| Worst (multi-msg, single pkt) | 226 (0.17%) | 526 (0.00%) | 722 (0.46%) | 843 (5.22%) |

The worst-case scenario drops from 307 ns to 226 ns (~26% reduction), confirming that batching reduces queue contention when multiple messages arrive in a single packet. The best-case result is unchanged, as a single-message packet already issues only one push.

Despite the improvement, batch push is not used in the current production path. Accumulating messages before the push introduces a handoff delay between pipeline units — downstream components receive nothing until the full packet is processed, which may create bottlenecks under real stress conditions that microbenchmarks do not capture. The current approach of parse-and-push per message is retained for now; this will be re-evaluated with further real-load stress testing prior to production deployment.

---

## OUCH BIST Parser

The OUCH BIST parser receives binary TCP packets. Like FIX, 
partial messages can occur due to TCP fragmentation. Message 
boundary extraction is handled by a dedicated packet handler 
function before dispatch to parse.

### ParserOUCH BIST — Full Pipeline

| Scenario | Median (ns) | p50 cycles | p99 cycles | p999 cycles |
|----------|-------------|------------|------------|-------------|
| Best (single msg, single pkt) | 451 (1.32%) | 1109 (1.45%) | 1344 (0.86%) | 1653 (15.79%) |
| Normal (multi-msg, single pkt) | 3104 (0.57%) | 8508 (0.42%) | 9819 (1.48%) | 15617 (10.81%) |
| Worst — seq in-order | 838 (1.06%) | 2206 (0.23%) | 2683 (1.72%) | 4333 (3.99%) |

### Packet Handler Function

| Scenario | Median (ns) | p50 cycles | p99 cycles | p999 cycles |
|----------|-------------|------------|------------|-------------|
| Best | 41.2 (0.09%) | 32 (0.00%) | 66 (0.00%) | 70 (1.91%) |
| Normal | 46.2 (0.10%) | 44 (1.90%) | 78 (0.00%) | 82 (0.77%) |
| Worst | 56.3 (0.18%) | 72 (0.88%) | 112 (0.00%) | 145 (2.22%) |

### parse() Function

| Message Type | Median (ns) | p50 cycles | p99 cycles | p999 cycles |
|--------------|-------------|------------|------------|-------------|
| Accepted | 29.7 (0.05%) | 24 (0.00%) | 26 (2.45%) | 28 (3.38%) |
| Cancelled | 29.7 (0.08%) | 24 (0.00%) | 26 (3.80%) | 28 (5.78%) |
| Executed | 29.7 (0.09%) | 24 (0.00%) | 26 (4.10%) | 28 (6.70%) |

### OUCH BIST Summary — Cost Breakdown

| Stage | Normal cycles | Worst cycles |
|-------|--------------|--------------|
| Packet handler | 44 | 72 |
| parse() — Accepted | 24 | 24 |
| Handler + parse | 68 | 96 |
| ParserOUCH full pipeline | 8508 | 2206 |
| Dispatch overhead | 8440 | 2110 |
| Handler % of total | 0.5% | 3.3% |
| parse % of total | 0.3% | 1.1% |
| Dispatch overhead % | 99.2% | 95.6% |

The normal scenario costs ~4x more than the worst — an apparent inversion driven by queue push frequency. Normal delivers 3 complete messages in a single packet, each pushed to the queue individually: 3 sequential pushes per iteration. Worst delivers the same 3 messages across 3 partial packets, so pushes are spread across boundaries. To confirm this, the benchmark was re-run with queue push disabled:

### ParserOUCH BIST — Full Pipeline (without SPSC queue push)

| Scenario | Median (ns) | p50 cycles | p99 cycles | p999 cycles |
|----------|-------------|------------|------------|-------------|
| Best (single msg, single pkt) | 297 (1.19%) | 740 (0.52%) | 887 (1.63%) | 1104 (17.88%) |
| Normal (multi-msg, single pkt) | 481 (1.73%) | 1256 (0.12%) | 1565 (2.74%) | 2319 (20.91%) |
| Worst — seq in-order | 468 (0.28%) | 1218 (0.21%) | 1529 (1.52%) | 2104 (24.86%) |

Without the queue push, normal and worst converge to the same range (~480 ns / ~1220–1260 cycles), confirming that the inversion in the full pipeline numbers is entirely attributable to queue push cost. parse() and packet handler together remain under 5% of total cost in both scenarios. This result strengthens the case for replacing the current boost lockfree queue with a custom queue implementation — the queue push alone accounts for the majority of observable latency difference between scenarios.

---

## OUCH NASDAQ Parser

OUCH NASDAQ shares the same dispatch and packet handler 
implementation as OUCH BIST. Only the parse() function 
differs per venue and is measured separately. Dispatch 
and packet handler costs can be referenced from OUCH BIST results.

### parse() Function

| Message Type | Median (ns) | p50 cycles | p99 cycles | p999 cycles |
|--------------|-------------|------------|------------|-------------|
| StockDir | 37.7 (0.12%) | 42 (0.00%) | 52 (0.00%) | 72 (0.00%) |
| AddOrder | 36.0 (0.05%) | 38 (0.00%) | 44 (1.43%) | 54 (1.55%) |
| Exec | 38.2 (0.06%) | 44 (0.00%) | 56 (1.71%) | 98 (0.00%) |
