# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **8 physical · 16 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 125.2 | 100% |
| 4 | 125.8 | 100% |
| 8 | 123.8 | 98% |
| 16 | 125.3 | 100% |
| 32 | 125.1 | 99% |

**Best**: `-t 4` at 125.8 tok/s
**Slowest tested**: `-t 8` at 123.8 tok/s (1.02x spread)
**Against the physical-core default** (`-t 8`, 123.8 tok/s): 1.02x

Use this in your run:

```bash
LAB_N_THREADS=4 make bench
```

## Your explanation

**Knee location**: Curve almost flat - no clear knee because **GPU offload is active**.

**Ket qua phan tich:**

| threads | tok/s | vs best |
|---------|-------|---------|
| 1 | 125.2 | 100% |
| 4 | 125.8 | 100% |
| 8 | 123.8 | 98% |
| 16 | 125.3 | 100% |
| 32 | 125.1 | 99% |

**Co che chinh:**

1. **GPU-accelerated decode**: May co RTX 2070 Super voi 8GB VRAM. Cau hinh `ngl=99` offload tat ca layers len GPU, bay gio decode chay tren CUDA, khong phai CPU.

2. **CPU threads khong anh huong**: Khi decode dien ra tren GPU, so luong CPU threads chi anh huong:
   - Phan nho cua prefill (neu prompt ngan)
   - Overhead scheduling

3. **Tang threads co the lam cham hon**: Tai `-t 8` (physical cores), tok/s gap 2% vi:
   - Thread scheduling overhead
   - Memory contention khi nhieu threads truy cap RAM cung luc
   - GPU da busy roi, CPU threads chi doi

4. **Sut rac giua cac muc**: 1.02x spread la rat nho, cho thay GPU la bottleneck chinh khong phai CPU.

**Ket luan**: tren may nay, **thread count khong tao hieu qua lon** vi:
- GPU-accelerated inference bo qua CPU bottleneck
- Speedup 1.02x voi `-t 4` khong dang ke

**Implication cho REFLECTION §5**: Thay doi nao tao speedup lon nhat? Tren may nay, **GPU offload** (ngl=99) la thay doi quan trong nhat. Neu tat GPU offload (`ngl=0`), decode se chay CPU-only va thread count se co y nghia lon hon.

**Luu y**: Ket qua nay rat khac voi may CPU-only. Tren laptop 8GB RAM khong co GPU, thread tuning co the tao 10-20% speedup thuc su.
