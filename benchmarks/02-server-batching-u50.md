# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 8 samples over
30s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.91 of 4 slots (98%) |
| `requests_processing` | 0 |
| `requests_deferred` | 0 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 19452 |

Highest sampled value was **3.91 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` stayed at zero: every request found a free slot on arrival.

## Your observation

**Peak batch width:** 3.91 of 4 slots (98% utilization)

**Does it match effective concurrency?** Yes, indirectly:
- Effective concurrency at 50 users: 39.5 requests in flight
- Peak batch width: 3.91 slots (only 4 requests processed simultaneously at decode step)
- The difference (39.5 vs 4) represents queued requests waiting for slots

**Analysis:**
- `n_busy_slots_per_decode` measures how many slots are actively decoding at each decode step
- With 98% utilization (3.91/4), the scheduler is near saturation
- `requests_deferred = 0` means no request was rejected, but they queued up
- Effective concurrency of 39.5 = 4 active + 35.5 queued requests

**Trust:** I trust both metrics because they measure different things:
- Batch width = instantaneous concurrent decode operations (4 max)
- Effective concurrency = total requests in flight including queue (39.5)

This confirms saturation: 39.5 requests competing for 4 decode slots means heavy queueing.
