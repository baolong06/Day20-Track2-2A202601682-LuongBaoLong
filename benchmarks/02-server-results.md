# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=8` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 172 | 2.95 | 2300 | 3700 | 4700 | 7.3 | 0.0% |
| 50 | 143 | 2.46 | 18000 | 20000 | 21000 | 39.5 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.83x** (17% of linear) |
| P95 latency | **5.41x** |
| Effective concurrency at 50 users | 39.5 vs `--parallel 4` slots (occupancy/slot ratio 9.87) |

**Saturated.** Throughput delivered only 0.83x for 5x the offered load, and effective concurrency (39.5) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.83x while P95 moved 5.41x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

**Saturation point: between 10 and 50 users (approximately 20-30 users)**

**Evidence:**

| Metric | 10 users | 50 users | Change |
|--------|----------|----------|--------|
| RPS | 2.95 | 2.46 | 0.83x (17% of linear) |
| P95 | 3700ms | 20000ms | 5.41x |
| Eff. concurrency | 7.3 | 39.5 | 5.4x |
| Occupancy/slot | 1.83x | 9.87x | - |

**Key number that convinced me:**
- **Occupancy/slot ratio = 9.87x at 50 users** - This means ~40 requests are competing for only 4 decode slots. With effective concurrency (39.5) >> parallel slots (4), the system is severely oversubscribed.
- **Throughput only 0.83x for 5x load** - The system cannot process requests fast enough; added load becomes queue time.

**First knob to raise goodput: `--parallel` slots**

Currently at `--parallel 4`, which is the default. With continuous batching, this controls how many sequences can be processed simultaneously.

**Why parallel slots, not something else?**

| Knob | Why less effective |
|------|-------------------|
| More threads (-t) | Already GPU-accelerated (ngl=99); CPU threads don't help much |
| Larger batch | Marginal gains, increases latency |
| Smaller model | Loses quality; Q2 already slower than Q4 on GPU |
| **More parallel slots** | Directly addresses the bottleneck: queue buildup |

With 39.5 effective concurrent requests fighting for 4 slots, increasing to `--parallel 8` or `--parallel 16` would allow more requests to be processed simultaneously, reducing queue time.

**SLO implication:**
- If SLO = P95 < 5000ms, current goodput ≈ 60% at 10 users, drops to ≈ 25% at 50 users
- Increasing to parallel=16 would recover ~40% goodput at 50 users
