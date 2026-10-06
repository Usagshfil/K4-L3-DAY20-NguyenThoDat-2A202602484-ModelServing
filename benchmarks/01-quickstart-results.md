# 01 - Measure: latency baseline

Model `Gemma 4 E2B` � host `Windows-AMD64` � llama.cpp `b10488`
Settings: `threads=10` `ngl=99` `ctx=2048`
`max_tokens=64` � warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 � `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 26588 | 883 / 4195 | 59.1 / 60.7 | 4580 / 8017 / 8017 | 16.9 |
| UD-Q2_K_XL | 2.24 | 15184 | 1349 / 12015 | 111.3 / 140.1 | 8730 / 19002 / 19002 | 9.0 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.88x SLOWER** than `UD-Q4_K_XL` here, despite being 0.73 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead � few cores, no GPU offload � the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

On my machine (Intel i7-1250U with Intel Iris Xe Graphics via Vulkan), the 2-bit quantization (UD-Q2_K_XL) is definitely NOT worth it. Although it saves 0.73 GB disk/RAM (2.24 GB vs 2.97 GB), its decode speed drops from 16.9 tok/s to only 9.0 tok/s (1.88x slower), and TTFT P50 degrades from 883 ms to 1349 ms. This occurs because the hardware is compute-limited on the integrated GPU: the complex dequantization arithmetic required for 2-bit weights heavily outweighs the modest memory bandwidth savings. Furthermore, 2-bit quantization noticeably degrades generation quality. Therefore, UD-Q4_K_XL is vastly superior in both speed and quality.