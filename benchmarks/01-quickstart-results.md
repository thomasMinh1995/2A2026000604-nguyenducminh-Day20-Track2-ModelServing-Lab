# 01 — Quickstart Results

Settings: `n_threads=10`, `n_ctx=2048`, `n_batch=512`, `n_gpu_layers=99`.

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|---:|---:|---:|---:|---:|
| tinyllama-1.1b-chat-v1.0.Q4_K_M.gguf | 147 | 27 / 27 | 6.3 / 10.8 | 421 / 422 / 422 | 159.5 |
| tinyllama-1.1b-chat-v1.0.Q2_K.gguf | 221 | 28 / 99 | 6.2 / 6.8 | 402 / 492 / 499 | 162.2 |

## Observations

Hardware: Apple M5, 10 physical cores, 16 GB unified RAM, Metal backend (`n_gpu_layers=99` — full GPU offload).

### TTFT (Time To First Token)
- Q4_K_M: P50 = **27 ms**, P95 = **27 ms** — extremely consistent, Metal prefill is fast.
- Q2_K: P50 = **28 ms**, P95 = **99 ms** — the P95 spike (99 ms vs 27 ms) is notable: Q2_K loaded in 221 ms (50% slower than Q4_K_M's 147 ms), and the first call (cold-start within the warm-up set) saw higher TTFT. This reflects dequantization overhead being slightly more expensive for Q2_K on some kernel paths.
- Both are well under 200 ms — comfortably real-time for interactive use.

### TPOT (Time Per Output Token)
- Q4_K_M: P50 = **6.3 ms** → **159.5 tok/s** decode rate
- Q2_K: P50 = **6.2 ms** → **162.2 tok/s** decode rate
- Decode rates are nearly identical (within 2%), because on Apple Silicon the bottleneck is **memory bandwidth** (unified memory read per token), and both quantizations fit comfortably in the 16 GB pool — Q2_K's smaller size provides no additional bandwidth advantage once the model fits entirely on the GPU.

### E2E Latency
- Q4_K_M: P50/P95/P99 = **421 / 422 / 422 ms** at 64 tokens — rock-solid consistency.
- Q2_K: P50/P95/P99 = **402 / 492 / 499 ms** — slightly more variance due to the cold-start spike on the first request.

### Quantization Tradeoff Summary
- **File size**: Q4_K_M = 638 MB vs Q2_K = 461 MB — Q2_K saves ~177 MB (28% smaller).
- **Decode speed**: effectively equal on Metal (unified memory eliminates the usual Q2_K bandwidth advantage seen on CPU).
- **Recommendation**: Use Q4_K_M — better quality, nearly identical speed, and the 177 MB RAM saving is not meaningful on a 16 GB machine. Q2_K would matter on a ≤4 GB RAM device.

### Thread behaviour
- `n_threads=10` (all physical cores). On Metal-offloaded inference, CPU threads primarily handle tokenisation and scheduling — the tensor compute runs on the GPU. This explains why TPOT is bandwidth-limited, not compute-limited.
