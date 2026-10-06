# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` � `--parallel 4` � 14 samples over
60s at 2.0s intervals � raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.88 of 4 slots (97%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a � not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 4517 |

Highest sampled value was **3.88 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Under a 50-user load, continuous batching is clearly verified in action:
- `llamacpp:n_busy_slots_per_decode` peaked at **3.88 out of 4 slots** (~97% slot saturation), proving that incoming requests are dynamically batched together into concurrent decode steps rather than executed sequentially.
- With only 4 parallel slots available, `llamacpp:requests_deferred` hovered between 41 and 46, demonstrating severe queueing delays as requests waited for an available slot. This confirms that the server is operating well past its saturation point.