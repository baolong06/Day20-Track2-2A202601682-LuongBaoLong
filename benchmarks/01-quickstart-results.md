# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=8` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 4037 | 239 / 310 | 8.0 / 8.3 | 736 / 831 / 831 | 125.8 |
| UD-Q2_K_XL | 2.24 | 5572 | 244 / 499 | 8.9 / 9.3 | 804 / 1073 / 1073 | 111.9 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.12x SLOWER** than `UD-Q4_K_XL` here, despite being 0.73 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

**Khong, Q2 khong worth it tren may nay.**

So sanh:
- Q4_K_XL: 2.97 GB, decode 125.8 tok/s
- Q2_K_XL: 2.24 GB, decode 111.9 tok/s

Q2 nho hon 0.73 GB nhung cham hon 1.12x. Ly do: may co RTX 2070 Super voi GPU offload, memory bandwidth tren GPU (448 GB/s) khong con la bottleneck. Compute cost cua dequantization Q2 nhieu hon bytes tiet kiem duoc, net effect la cham hon.

**Ket luan:** Tren GPU-accelerated system, Q4_K_XL tot hon - nhanh hon, load time ngan hon, va chat luong tot hon. Neu may chi co CPU hoac VRAM < 4GB, Q2 co the worth it de tranh OOM.
