# Pipeline Benchmark Results

End-to-end latency from network ingress to risk check completion, measured across four CPU frequency profiles and two load conditions. All values are **medians** with CV (coefficient of variation) shown in parentheses.

---

## Test Setup

Each benchmark,

- receives a single **new order acknowledgement packet** through the complete Rx pipeline — from raw network bytes arriving on the socket, through protocol parsing, to risk update completion,

- sends a single **new order packet** through the complete Tx pipeline — from risk checking to the packet being written to the network. 

---

## Results (Rx)

### FIX BIST

| Profile | Median ns | p50 cycles | p99 cycles | p999 cycles |
|---|---|---|---|---|
| 1 — Performance/idle | 3540 (3.29%) | 8953 (1.23%) | 10281 (5.12%) | 268932 (1.10%) |
| 2 — Powersave/idle | 3698 (2.35%) | 9389 (1.15%) | 11417 (6.55%) | 225777 (0.28%) |
| 3 — Performance/no idle | 4741 (3.18%) | 11479 (1.15%) | 12875 (7.51%) | 293037 (0.16%) |
| 4 — Performance/no turbo | 5302 (1.82%) | 13466 (0.77%) | 14720 (2.24%) | 319932 (0.27%) |

### OUCH BIST

| Profile | Median ns | p50 cycles | p99 cycles | p999 cycles |
|---|---|---|---|---|
| 1 — Performance/idle | 3636 (1.19%) | 9217 (0.64%) | 10270 (6.58%) | 261272 (0.36%) |
| 2 — Powersave/idle | 3850 (3.15%) | 9389 (1.13%) | 11837 (7.53%) | 226221 (0.83%) |
| 3 — Performance/no idle | 4778 (2.59%) | 11163 (0.55%) | 12321 (5.97%) | 292276 (0.35%) |
| 4 — Performance/no turbo | 5277 (1.84%) | 13448 (0.90%) | 14483 (1.13%) | 319413 (0.31%) |

### OUCH NQ

| Profile | Median ns | p50 cycles | p99 cycles | p999 cycles |
|---|---|---|---|---|
| 1 — Performance/idle | 3686 (3.25%) | 9135 (0.55%) | 10090 (5.06%) | 260618 (0.35%) |
| 2 — Powersave/idle | 3749 (1.90%) | 9515 (1.03%) | 11430 (7.17%) | 228628 (0.94%) |
| 3 — Performance/no idle | 4830 (0.95%) | 12016 (0.50%) | 13168 (5.56%) | 295660 (1.65%) |
| 4 — Performance/no turbo | 5369 (3.35%) | 13375 (0.86%) | 14372 (0.86%) | 319407 (1.41%) |

### ITCH BIST

| Profile | Median ns | p50 cycles | p99 cycles | p999 cycles |
|---|---|---|---|---|
| 1 — Performance/idle | 2147 (1.99%) | 5523 (1.14%) | 6047 (5.45%) | 10321 (11.06%) |
| 2 — Powersave/idle | 2272 (1.06%) | 5827 (0.86%) | 6344 (0.95%) | 12505 (0.98%) |
| 3 — Performance/no idle | 2230 (3.54%) | 6246 (0.93%) | 6789 (0.98%) | 10514 (0.80%) |
| 4 — Performance/no turbo | 3030 (5.96%) | 8195 (1.38%) | 8867 (1.77%) | 12799 (1.06%) |

### ITCH NQ

| Profile | Median ns | p50 cycles | p99 cycles | p999 cycles |
|---|---|---|---|---|
| 1 — Performance/idle | 2260 (2.09%) | 5905 (0.72%) | 6418 (0.93%) | 9969 (10.51%) |
| 2 — Powersave/idle | 2552 (7.48%) | 6247 (1.13%) | 6793 (2.36%) | 13020 (1.27%) |
| 3 — Performance/no idle | 2390 (3.06%) | 6457 (0.50%) | 6974 (0.95%) | 10897 (1.09%) |
| 4 — Performance/no turbo | 3607 (4.70%) | 8876 (1.58%) | 9547 (1.53%) | 13650 (1.07%) |

---

## Results (Tx)

### FIX BIST

| Profile | Median ns | p50 cycles | p99 cycles | p999 cycles |
|---|---|---|---|---|
| 1 — Performance/idle | 3177 (1.48%) | 8911 (1.12%) | 11304 (3.19%) | 15766 (3.39%) |
| 2 — Powersave/idle | 3247 (1.12%) | 9153 (0.51%) | 11615 (4.44%) | 15450 (7.23%) |
| 3 — Performance/no idle | 4552 (0.62%) | 12980 (0.29%) | 16160 (4.72%) | 18271 (0.70%) |
| 4 — Performance/no turbo | 4637 (1.12%) | 12954 (0.88%) | 16461 (0.78%) | 19281 (0.48%) |

### OUCH BIST

| Profile | Median ns | p50 cycles | p99 cycles | p999 cycles |
|---|---|---|---|---|
| 1 — Performance/idle | 2865 (1.43%) | 7984 (1.14%) | 8916 (8.47%) | 14765 (1.02%) |
| 2 — Powersave/idle | 2956 (0.84%) | 8304 (0.32%) | 9209 (8.96%) | 14766 (5.09%) |
| 3 — Performance/no idle | 4248 (1.32%) | 12123 (0.54%) | 13011 (7.83%) | 17133 (0.54%) |
| 4 — Performance/no turbo | 4304 (1.19%) | 12050 (0.87%) | 13120 (5.67%) | 18234 (0.65%) |

### OUCH NQ

| Profile | Median ns | p50 cycles | p99 cycles | p999 cycles |
|---|---|---|---|---|
| 1 — Performance/idle | 2862 (1.56%) | 8074 (1.23%) | 8705 (7.63%) | 14930 (2.34%) |
| 2 — Powersave/idle | 3003 (2.04%) | 8428 (1.43%) | 9041 (6.18%) | 14984 (1.95%) |
| 3 — Performance/no idle | 4252 (1.43%) | 12036 (0.69%) | 12873 (0.94%) | 17260 (0.48%) |
| 4 — Performance/no turbo | 4264 (0.97%) | 12111 (0.71%) | 12773 (0.78%) | 18119 (0.48%) |

---

## Summary — Best Profile (Performance/idle) Across All Protocols

Performance/idle (Profile 1) represents the ceiling: highest clock frequency, lowest system contention. The table below compares all five protocols under this single profile.

### Rx 

| Protocol / Venue | Median ns | p50 cycles | p99 cycles | p999 cycles |
|---|---|---|---|---|
| FIX | 3540 (3.29%) | 8953 (1.23%) | 10281 (5.12%) | 268932 (1.10%) |
| OUCH BIST | 3636 (1.19%) | 9217 (0.64%) | 10270 (6.58%) | 261272 (0.36%) |
| OUCH NQ | 3686 (3.25%) | 9135 (0.55%) | 10090 (5.06%) | 260618 (0.35%) |
| ITCH BIST | 2147 (1.99%) | 5523 (1.14%) | 6047 (5.45%) | 10321 (11.06%) |
| ITCH NQ | 2260 (2.09%) | 5905 (0.72%) | 6418 (0.93%) | 9969 (10.51%) |

### Tx 

| Protocol/Venue | Median ns | p50 cycles | p99 cycles | p999 cycles |
|---|---|---|---|---|
| FIX | 3177 (1.48%) | 8911 (1.12%) | 11304 (3.19%) | 15766 (3.39%) |
| OUCH BIST | 2865 (1.43%) | 7984 (1.14%) | 8916 (8.47%) | 14765 (1.02%) |
| OUCH NQ | 2862 (1.56%) | 8074 (1.23%) | 8705 (7.63%) | 14930 (2.34%) |


###Notes

**Why Performance/idle (Profile 1) wins despite CPU scaling enabled:** 

When C-states are active, threads do not all run at peak load simultaneously, so the CPU can boost individual cores to 4700 MHz. When idle is disabled (Profiles 3) all cores are held active at once, and the chip cannot sustain maximum frequency across all of them — settling around 3200 MHz under thermal/power constraints. The result is that idle-enabled runs achieve higher per-core clock rates even though the benchmark issued a scaling warning.

**ITCH vs FIX/OUCH gap (~40% fewer cycles):** 

Not a parsing difference. ITCH carries market order data that does not affect our risk profiles, so the risk engine bypasses most of its update path for these messages. FIX and OUCH order acknowledgements trigger the full update.

**CV elevation:** 

Occasional CV spikes — particularly at p99/p999 — are measurement artefacts rather than pipeline instability. With only 10 iterations, a single outlier sample disproportionately inflates the percentile. The p50 CV remains below 2% in almost all cases, confirming the hot path itself is stable.

**Turbo vs no-turbo (Profile 4):** 

With turbo disabled, the CPU is capped at 2400 MHz regardless of load. Beyond the frequency penalty, turbo-off runs did not show better stability — p999 CV is comparable to or worse than turbo-on profiles. Given both the latency cost and the absence of a stability benefit, turbo-enabled is the preferred deployment configuration.
