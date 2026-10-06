# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` � llama.cpp `b10488` �
retrieval backend: **keyword overlap** � 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 1.9 | 5305.8 | 5307.8 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 4432.3 | 4432.4 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 4745.3 | 4745.5 |

Mean per stage (ms): embed **0.0** � retrieve **0.7** �
llm **4827.8** � total **4828.6**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real?

| Day | Piece | Status | Notes |
|:--|:--|:--|:--|
| N16 | Cloud / IaC | stub | Running locally on Windows 11 host |
| N17 | Data pipeline | stub | Default in-memory toy documents |
| N18 | Lakehouse / Store | stub | Local memory corpus |
| N19 | Vector / Retrieval | stub | Keyword overlap retrieval fallback |
| N20 | Serving endpoint | real | Local `llama-server` OpenAI-compatible endpoint with Vulkan backend |

### Reflection on latency split
As demonstrated by the benchmark, the `llm` stage is overwhelmingly the bottleneck, taking 4827.8 ms out of 4828.6 ms total (~100% of pipeline latency), while keyword retrieval takes only 0.7 ms. To reduce the end-to-end latency of this pipeline by 2x, optimization efforts must target the LLM stage directly: leveraging Prefix Caching for the static system prompt to eliminate prefill latency, using Speculative Decoding to accelerate token generation, or deploying a smaller, faster draft model.
