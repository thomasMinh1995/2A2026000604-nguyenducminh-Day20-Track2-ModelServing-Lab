# 03 — Integration Notes

## Which N16–N19 pieces were connected

This integration uses a **simplified standalone stack** since prior-day deliverables (N16–N19) are not yet fully connected. The pipeline demonstrates the full RAG flow end-to-end using in-memory components:

| Layer | What was used | Status |
|---|---|---|
| **N16** Cloud/IaC | Not connected (no K8s/Compose) | Faked — runs locally |
| **N17** Data Pipelines | Not connected (no Airflow DAG) | Faked — docs hardcoded in `TOY_DOCS` |
| **N18** Lakehouse | Not connected (no Delta Lake) | Faked — in-memory dict list as "corpus" |
| **N19** Vector Index | Not connected (no embedding index) | Faked — keyword overlap scoring (`q_terms & doc_terms`) |
| **N20** Serving | ✅ `llama-server` at `http://localhost:8080/v1` | Real — TinyLlama-1.1B Q4_K_M via llama-cpp-python |

The `retrieve()` function uses bag-of-words keyword overlap instead of a real embedding index. This is sufficient to demonstrate the retrieval → prompt assembly → LLM call pipeline shape.

## Where time goes — latency breakdown

Measured with `time.perf_counter()` around each stage across 3 queries:

| Query | Retrieve (ms) | LLM (ms) | Total (ms) |
|---|---:|---:|---:|
| "Why is goodput more useful than throughput?" | 0.0 | 1144.2 | 1144.2 |
| "What problem does PagedAttention actually solve?" | 0.0 | 694.0 | 694.0 |
| "When should I think about disaggregated serving?" | 0.0 | 599.3 | 599.4 |

**Key observation:** Retrieval is effectively **0 ms** — keyword scoring over 5 in-memory docs is trivial. **100% of latency is in the `llama-server` call.** With a real vector index (Qdrant, FAISS, pgvector), retrieval would add ~10–50 ms over network but still be negligible compared to LLM generation time (~600–1200 ms per query at 160 tok/s).

The first query is ~2× slower than subsequent ones (**1144 ms vs 600–700 ms**) — this is **prefix cache warm-up**: the system prompt is identical across all three queries, so after the first call the KV cache holds the system prompt prefix and subsequent calls skip its prefill entirely. This is the `llamacpp:prompt_tokens_total` growing slower than `tokens_predicted_total` pattern described in the Track 03 README.

## Pipeline output (sample run)

```
=== Why is goodput more useful than throughput? ===
  contexts: ['n20-paged', 'n20-radix', 'n20-disagg']
  timings : {'retrieve': 0.0, 'llm': 1144.2, 'total': 1144.2}

=== What problem does PagedAttention actually solve? ===
  contexts: ['n20-paged', 'n20-radix', 'n20-disagg']
  timings : {'retrieve': 0.0, 'llm': 694.0, 'total': 694.0}

=== When should I think about disaggregated serving? ===
  contexts: ['n20-disagg', 'n20-paged', 'n20-radix']
  timings : {'retrieve': 0.0, 'llm': 599.3, 'total': 599.4}
```

## Improvements for a production stack

1. **Real vector retrieval**: replace `retrieve()` with calls to a running Qdrant/FAISS index using `sentence-transformers` embeddings — latency budget: ~10–50 ms.
2. **Token budget enforcement**: use `llm.tokenize()` from `llama-cpp-python` (not `tiktoken`) to truncate context blocks to fit within `n_ctx` before calling the server.
3. **Prefix caching**: keep the system prompt wording identical across all calls — llama.cpp reuses the KV cache for the shared prefix, observable via `llamacpp:prompt_tokens_total` growing slower than `tokens_predicted_total`.
4. **Streaming**: enable `stream=True` in `call_llm()` to start showing tokens to the user after TTFT (~27 ms) instead of waiting for full generation.
