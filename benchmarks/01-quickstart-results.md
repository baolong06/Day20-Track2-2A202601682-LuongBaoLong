# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=8` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 97671 | 289 / 40398 | 9.2 / 10.3 | 867 / 41044 / 41044 | 108.3 |
| UD-Q2_K_XL | 2.24 | 6436 | 296 / 111463 | 10.1 / 13.2 | 927 / 112239 / 112239 | 99.4 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.09x SLOWER** than `UD-Q4_K_XL` here, despite being 0.73 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

**Ket qua chi tiet:**

- **UD-Q4_K_XL (2.97 GB)**: Load nhanh hon dong thoi cho toc do decode tot hon (108.3 tok/s)
- **UD-Q2_K_XL (2.24 GB)**: Nho hon 0.73 GB nhung decode cham hon 1.09 lan

**Phan tich co che:**

May cua toi co RTX 2070 Super voi GPU offload (ngl=99), tuc la GPU-accelerated inference. Trong cau hinh nay:
1. **Memory bandwidth** khong con la bottleneck chinh vi GPU co bandwidth rat cao
2. **Compute cost** cua dequantization trong Q2 thuc su nhieu hon Q4, nhung khong duoc bu tru bang việc tiết kiem memory bandwidth
3. **TTFT P95 cao bat thuong** (40398ms/111463ms) la do cold start - model chua nam trong GPU cache

**Ket luan:**

 tren may nay, **UD-Q4_K_XL dang duoc su dung**. Ly do:
- Toc do decode tot hon (108.3 vs 99.4 tok/s)
- Model load nhanh hon dang ke (97671ms vs 111463ms cho cold start dau tien)
- 2.97 GB van la gap Be vua voi 8GB VRAM cua RTX 2070 Super
- Che do Q2 khong mang lai loi nhuan ve mat toc do, chi lam giam chat luong model

Neu may chi co 4-6GB VRAM, Q2 co the la lua chon tot hon de dam bao khong bi OOM.
