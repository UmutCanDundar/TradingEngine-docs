# Pipeline Overview

## Pre-test Research

| Configuration | Result |
|---------------|--------|
| SMT on vs off | on performed better, off caused minor degradation |
| isolcpus | No measurable gain |
| IRQ affinity | No measurable gain |
| pstate active vs passive | Negligible difference, active selected |

## CPU Profiles Tested

All profiles share: **Hugepages:** always — **SMT:** on — **pstate:** active

| Profile | Governor | Turbo | Idle 
|---------|----------|-------|------|
| 1 — Performance/idle | performance | on | enabled | 
| 2 — Powersave/idle | powersave | on | enabled | 
| 3 — Performance/no idle | performance | on | disabled | 
| 4 — Performance/no turbo | performance | off | enabled | 

## Profile Selection

Profile 1 consistently has the best test result both in median and tail latency.

**Selected Profile: Profile 1 — Performance/Idle**

> All benchmark and perf tests were conducted under this profile.

For the full comparison, see
[Pipeline Benchmark Results](benchmark_pipeline.md#profile-comparison).





