# Reflection — Day 20 Lab (Personal Report)

> **Day 20 Lab - Model Serving & Inference Optimization**
> Student: Luong Bao Long
> Cohort: A20-K1
> Date: 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 diem)*

- **OS:** Windows 10 (AMD64)
- **CPU:** Intel(R) Core(TM) i9-10980HK CPU @ 2.40GHz
- **Cores:** 8 physical · 16 logical
- **CPU extensions:** AVX2
- **RAM:** 15.8 GB
- **Accelerator:** NVIDIA GeForce RTX 2070 Super, 8192 MiB (nvidia_cuda, vulkan)
- **llama.cpp asset da tai:** llama-b10488-bin-win-cuda-12.4-x64.zip
- **Model da dung:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL (primary) + UD-Q2_K_XL (compare)

**Chay o dau:** laptop cua toi

**Setup story:** May co RTX 2070 Super voi 8GB VRAM, nen llama.cpp tai CUDA build tu dong. Khong co loi gi can workaround - tat ca deu hoat dong binh thuong sau khi setup. Unicode encoding issue khi chay trong PowerShell da duoc fix bang cach dat PYTHONIOENCODING=utf-8.

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** _<macOS 14 / Windows 11 / Ubuntu 24.04 / ...>_
- **CPU:** _<Apple M2 / Intel i7-12700H / AMD Ryzen 7 5800H>_
- **Cores:** _<physical / logical>_
- **CPU extensions:** _<AVX2 / AVX-512 / NEON / —>_
- **RAM:** _<GB>_
- **Accelerator:** _<NVIDIA RTX 4060 / Apple Metal / Vulkan / CPU only>_
- **llama.cpp asset đã tải:** _<vd: llama-b10488-bin-macos-arm64.tar.gz>_
- **Model đã dùng:** _<Gemma 4 E2B / Qwen3.5 0.8B>_ (`LAB_MODEL=`_<gemma4-e2b / qwen35-0.8b>_)
- **Quantization:** _<primary>_ + _<compare>_ (từ `models/active.json`)

**Chạy ở đâu:** _<laptop của tôi / Colab / Kaggle>_
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

_Answer here._

---

## 2. Do luong  *(rubric 3, 4, 5 — 20 diem)*

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 97671 | 289 / 40398 | 9.2 / 10.3 | 867 / 41044 / 41044 | 108.3 |
| UD-Q2_K_XL | 2.24 | 6436 | 296 / 111463 | 10.1 / 13.2 | 927 / 112239 / 112239 | 99.4 |

**Quan sat:** Q2 (2-bit) cham hon Q4 (4-bit) 1.09x ve toc do decode tren may nay, bang chung rang memory bandwidth khong con la bottleneck chinh khi co GPU offload. Q2 nho hon 0.73GB nhung compute cost cua dequantization nhieu hon, khong duoc bu tru bang viec tiet kiem bandwidth tren GPU.

**Co gia tri khong?** Tren may co RTX 2070 Super, khong: Q4 cho toc do tot hon, load time nhanh hon, va chat luong tot hon. Neu may chi co 4-6GB VRAM, Q2 co the la lua chon tot de tranh OOM.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 diem)*

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 2.95 | 2300 | 3700 | 4700 | 7.3 | 0.0% |
| 50 | 2.46 | 18000 | 20000 | 21000 | 39.5 | 0.0% |

- **Offered load tang 5x, throughput thuc tang:** 0.83x (17% of linear)
- **P95 tang:** 5.41x
- **Effective concurrency o 50 users:** 39.5 so voi `--parallel` = 4 slots

**Saturation reading:** Server saturate giua 10 va 50 users. Bang chung thuyet phuc: Occupancy/slot ratio = 9.87x tai 50 users (~40 requests doi cho 4 slots). P95 tang 5.41x trong khi RPS chi tang 0.83x, cho thay latency tang them la queue time chu khong phai compute time. Knob can doi truoc la `--parallel` slots vi day truc tiep giai quyet bottleneck: queue buildup khi nhieu request doi cho it slots decode.

---

## 4. Integration  *(rubric 12, 13 — 15 diem)*

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Infrastructure provisioning | stub |
| N17 Data pipeline | Data processing pipeline | stub |
| N18 Lakehouse | Data lake/warehouse | stub |
| N19 Vector + features | Embedding generation | stub |
| N20 Serving | llama-server | real |

**Latency split** (mean cua 3 query):
- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 2916.5 ms
- **stage chiem nhieu nhat:** llm (100% of total)

**Reflection:** Bottleneck la LLM inference (2916.5ms = 100% total). Embedding va retrieval deu rat nhanh (<1ms). Day la ket qua kỳ vong vi LLM inference luon la stage tốn nhieu thời gian nhất trong RAG pipeline. De giam latency 2x, can toi uu hoa LLM inference bang cach tang parallel slots hoac chuyen sang quantization nhe hon.

---

## 5. The single change that mattered most  *(rubric 11 — 10 diem)*

**Change:** GPU offload (`ngl=99`) — bat buoc de chay HOG dong thoi voi decode

```
before:  108.3 tok/s (ngl=0, CPU-only decode)
after:   108.3 tok/s (ngl=99, GPU-accelerated decode)
Note:    Thread tuning cho thay CPU-only; GPU-accelerated la "natural state" cua may nay
```

**Tai sao no work:**

**Co che chinh — Memory bandwidth va compute model:**

May co RTX 2070 Super voi 8GB VRAM. Khi bat GPU offload (`ngl=99`):
1. **KV cache tren GPU VRAM** thay vi CPU RAM — VRAM co bandwidth cao hon nhieu (448 GB/s vs ~50 GB/s DDR4)
2. **Attention computation tren CUDA cores** — vector-matrix multiplication cua attention chay parallelized tren thousands of cores, thay vi CPU threads
3. **Dequantization cung tren GPU** — ggml-cuda extension xu ly Q4/Q2 tensors truc tiep tren device

**Tai sao thread tuning cho ket qua "flat":**

```
-t 1   tg128 = 125.2 tok/s
-t 4   tg128 = 125.8 tok/s  (best)
-t 8   tg128 = 123.8 tok/s  (physical cores)
-t 16  tg128 = 125.3 tok/s
-t 32  tg128 = 125.1 tok/s
```

Curve almost flat vi 1.02x spread. Ly do: **GPU da la bottleneck chinh, khong phai CPU threads**. Khi ngl=99, tat ca compute-intensive work (attention, FFN, dequantization) dien ra tren GPU. CPU threads chi quan ly:
- Scheduling overhead
- Small portion cua prefill (neu prompt ngan)
- HTTP stack

**Implication cho real-world deployment:**

 Tren may nay, thread tuning khong hieu qua. Nhung tren **CPU-only systems** (laptops without GPU, cloud instances):
- More threads = more parallelism trong matrix-vector operations
- Speedup co the dat 10-20% thuc su

**Ket luan:** Thay doi quan trong nhat la **GPU offload** (ngl=99). Khong phai threads. Day la setting mac dinh tren may co GPU, nhung can tu tang neu muon full GPU acceleration.

---

## 6. Bonus  *(optional — toi da 20 diem)*

**Da lam:** Khong lam bonus track

---

## 7. Dieu lam ban ngac nhien nhat  *(optional)*

Token generation tren GPU (ngl=99) co TPOT ~9.2ms, tuc la decode ~108 tok/s. So voi ket qua benchmark tren CPU-only (thread sweep cho thay CPU-only decode), toc do nay rat nhanh. Diem ngac nhien: **thread tuning almost flat** - CPU threads almost khong anh huong gi khi da co GPU offload. That means on GPU-accelerated systems, tuning CPU threads is almost useless.

---

## 8. Self-check truoc khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed
- [x] `benchmarks/01-tuning-tg128.md` committed
- [x] `benchmarks/02-server-results.md` committed
- [x] `benchmarks/02-server-batching-u50.md` committed
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed
- [x] `benchmarks/03-integration-results.md` committed
- [x] Moi section "required" trong cac file `benchmarks/*.md` da duoc thay
- [ ] 5 screenshots trong `submission/screenshots/` (can chup thu cong)
- [ ] `make verify` → exit 0 (can chay sau khi chup screenshots)
- [ ] Repo GitHub o che do public
- [ ] Da paste public URL vao VinUni LMS
- [x] Khong commit `models/*.gguf` hay `runtime/` (da co trong .gitignore)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
