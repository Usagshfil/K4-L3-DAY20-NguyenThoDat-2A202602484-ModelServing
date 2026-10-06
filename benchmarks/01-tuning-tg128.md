# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` � host `Windows-AMD64` � llama.cpp `b10488`
CPU: **10 physical � 12 logical** cores � `ngl=99` � metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 16.7 | 94% |
| 5 | 15.1 | 85% |
| 10 | 15.9 | 90% |
| 12 | 17.5 | 99% |
| 24 | 17.8 | 100% |

**Best**: `-t 24` at 17.8 tok/s
**Slowest tested**: `-t 5` at 15.1 tok/s (1.18x spread)
**Against the physical-core default** (`-t 10`, 15.9 tok/s): 1.11x

Use this in your run:

```bash
LAB_N_THREADS=24 make bench
```

## Your explanation

On my machine, the sweep curve shows a relatively flat profile across thread counts (ranging from 15.1 to 17.8 tok/s) with the knee effectively at **-t 12** (17.5 tok/s), and diminishing returns up to -t 24 (17.8 tok/s, a marginal 1.7% gain). 

This behavior departs from a pure CPU-bound curve (which typically peaks sharply around physical cores and drops due to memory bus contention) for two primary architectural reasons:
1. **Vulkan GPU Offload (`ngl: 99`)**: All model layers are offloaded to the Intel Iris Xe integrated GPU via Vulkan. As a result, the primary bottleneck is the iGPU compute and unified memory bandwidth, while CPU worker threads primarily manage command dispatch and Vulkan queue synchronization rather than heavy GEMM computations.
2. **Intel Hybrid Architecture (Alder Lake 2 P-cores + 8 E-cores)**: Increasing threads to 12 (matching logical cores) ensures full utilization across both P-cores and E-cores without starving GPU submission queues. Oversubscription at -t 24 keeps the dispatch pipeline saturated, but yields almost no further speedup since the GPU is already fully loaded.