# 02 — Server Load Test Results

Hardware: Apple M5 · 10 cores · 16 GB · Metal (`n_gpu_layers=99`)
Server: `llama-cpp-python` Python server · model: `tinyllama-1.1b-chat-v1.0.Q4_K_M.gguf`
Settings: `n_threads=10`, `n_ctx=2048`, `n_gpu_layers=99`

> **Note:** Python server has NO `/metrics` endpoint. Prometheus metrics (n_busy_slots_per_decode etc.) require `make build-llama` + `make serve-native`. See Track 02 README.

---

## Load Test — 10 concurrent users (`-u 10 -r 1 -t 1m`)

| Endpoint | # Reqs | Fails | Avg (ms) | Min (ms) | Max (ms) | Median | req/s |
|---|---:|---:|---:|---:|---:|---:|---:|
| `POST /v1/chat/completions` short | 56 | 0 | 5365 | 641 | 7792 | 6000 | 1.02 |
| `POST /v1/chat/completions` long-rag | 19 | 0 | 6099 | 2266 | 7813 | 6800 | 0.35 |
| **Aggregated** | **75** | **0** | **5551** | **641** | **7813** | **6100** | **1.37** |

### Percentiles — 10 users

| Endpoint | P50 | P75 | P90 | P95 | P99 | P100 |
|---|---:|---:|---:|---:|---:|---:|
| short | 6000 | 6700 | 7400 | 7600 | 7800 | 7800 |
| long-rag | 6800 | 7100 | 7800 | 7800 | 7800 | 7800 |
| **Aggregated** | **6100** | **6900** | **7400** | **7700** | **7800** | **7800** |

---

## Load Test — 50 concurrent users (`-u 50 -r 2 -t 1m`)

| Endpoint | # Reqs | Fails | Avg (ms) | Min (ms) | Max (ms) | Median | req/s |
|---|---:|---:|---:|---:|---:|---:|---:|
| `POST /v1/chat/completions` short | 64 | 0 | 19461 | 555 | 32649 | 23000 | 1.09 |
| `POST /v1/chat/completions` long-rag | 21 | 0 | 16403 | 1877 | 32332 | 16000 | 0.36 |
| **Aggregated** | **85** | **0** | **18705** | **555** | **32649** | **21000** | **1.45** |

### Percentiles — 50 users

| Endpoint | P50 | P75 | P90 | P95 | P99 | P100 |
|---|---:|---:|---:|---:|---:|---:|
| short | 24000 | 29000 | 31000 | 32000 | 33000 | 33000 |
| long-rag | 16000 | 23000 | 29000 | 31000 | 32000 | 32000 |
| **Aggregated** | **21000** | **28000** | **31000** | **32000** | **33000** | **33000** |

---

## 10-user vs 50-user Comparison

| Metric | 10 users | 50 users | Δ |
|---|---:|---:|---:|
| Aggregated P50 (ms) | 6100 | 21000 | **+3.4×** |
| Aggregated P95 (ms) | 7700 | 32000 | **+4.2×** |
| short P50 (ms) | 6000 | 24000 | **+4.0×** |
| short P95 (ms) | 7600 | 32000 | **+4.2×** |
| long-rag P50 (ms) | 6800 | 16000 | **+2.4×** |
| Throughput (req/s) | 1.37 | 1.45 | **+6%** |

---

## Observations

### 10-user run
- **0 failures** across 75 requests — server handled all load without errors.
- Short prompts: P50 = **6.0 s**, P95 = **7.6 s**. Latency is high because the Python server has **no continuous batching** — each request is serialised. With 10 concurrent users queuing up, requests wait their turn.
- Long-rag prompts: P50 = **6.8 s**, P95 = **7.8 s** — longer context (prefill cost) pushes latency up ~800 ms vs short.
- Min latency of **641 ms** (short prompt, low queue depth) shows the server can respond fast when not contended.
- Throughput: **1.37 req/s** aggregate.

### 50-user run
- **0 failures** across 85 requests — server stayed stable but latency exploded.
- Short P50 jumped from **6.0 s → 24.0 s** (+4×). P95 hit **32.0 s** — near the 120 s client timeout threshold.
- Throughput barely moved: **1.37 → 1.45 req/s** (+6%). This is the classic **server-bottleneck saturation** pattern: adding more users doesn't add more throughput because the single inference slot is the ceiling.
- Min latency still **555 ms** — early requests with empty queues are still fast; all others wait.

### Key insight: Queue wait dominates at 50 users
The Python `llama_cpp.server` processes one request at a time (no `--parallel` / continuous batching). At 50 users, the request queue grows unbounded — every 640 ms of decode time adds ~640 ms to every waiting request. This is exactly the **goodput@SLO** problem: throughput (req/s) stays flat while latency SLO is massively violated.

**Fix**: native `llama-server` with `--parallel 4 --cont-batching` would interleave decoding across 4 slots simultaneously, reducing P95 dramatically without changing the total compute budget.
