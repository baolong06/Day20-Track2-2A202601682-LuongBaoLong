# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 3144.0 | 3144.1 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 2800.4 | 2800.5 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 2805.0 | 2805.1 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **2916.5** · total **2916.6**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

| Piece | Real or stub? | Reason |
|-------|---------------|--------|
| N16 Cloud/IaC | stub | Keyword overlap fallback - no real infrastructure |
| N17 Data pipeline | stub | No real data pipeline |
| N18 Lakehouse | stub | No real data lake |
| N19 Vector + features | stub | Keyword overlap fallback for retrieval |

**Is dominant stage (llm = 100%) what you expected?**
Yes, LLM inference dominates because:
1. Embedding/retrieval use keyword matching (fast, <1ms)
2. LLM inference requires GPU compute (attention, FFN, dequantization)

**To halve latency, attack LLM stage:**
- Use smaller quantization (UD-Q2_K_XL or UD-Q3_K_XL)
- Increase parallel slots for batch processing
- Use speculative decoding (bonus C1 with MTP head)
