# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` � llama.cpp `b10488` �
`--parallel 4` � `ctx=2048` � `threads=10` �
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 16 | 0.28 | 29000 | 41000 | 41000 | 7.4 | 0.0% |
| 50 | 26 | 0.44 | 38000 | 55000 | 58000 | 14.6 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.60x** (32% of linear) |
| P95 latency | **1.34x** |
| Effective concurrency at 50 users | 14.6 vs `--parallel 4` slots (occupancy/slot ratio 3.66) |

**Saturated.** Throughput delivered only 1.60x for 5x the offered load, and effective concurrency (14.6) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

P95 grew no faster than throughput (1.34x vs 1.60x), so this server still has headroom at 50 users.

> **Small sample.** Only 16 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Trust the throughput-scaling
> row over the concurrency row here, and run longer (`-t 3m`) if you want firmer numbers.

## Your reading

The server reaches saturation well below 50 users, with two clear pieces of evidence:
1. **Throughput Scaling Flattened**: While offered load grew by 5x (10 to 50 users), actual delivered throughput only grew by 1.60x (0.28 to 0.44 RPS), capturing only 32% of linear scaling.
2. **Effective Concurrency vs Slots (Little's Law)**: At 50 users, effective concurrency reached 14.6—over 3.6x the available capacity of `--parallel 4` slots. Combined with our Prometheus observation of 41–46 deferred requests, the bulk of the P95 latency (55,000 ms) is spent waiting in the arrival queue rather than active GPU computation.

To improve Goodput@SLO under high load, the primary knob I would change first is **`--parallel`** (increasing slots from 4 to 6 or 8) along with configuring a request admission timeout (dropping/rejecting excess queued requests). Increasing parallel slots directly relieves queue bottlenecking via continuous batching, while admission control prevents queue accumulation from inflating P95/P99 for in-flight requests.