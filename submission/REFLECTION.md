# Reflection — Lab 20 (Personal Report)

> **Đây là báo cáo cá nhân.** Mỗi học viên chạy lab trên laptop của mình, với spec của mình. Số liệu của bạn không so sánh được với bạn cùng lớp — chỉ so sánh **before vs after trên chính máy bạn**. Grade rubric tính theo độ rõ ràng của setup + tuning của bạn, không phải tốc độ tuyệt đối.

---

**Họ Tên:** Nguyễn Ngọc Hiếu
**Cohort:** _<A20-K1>_
**Ngày submit:** _2026-05-06_

---

## 1. Hardware spec (từ `00-setup/detect-hardware.py`)

> Paste output của `python 00-setup/detect-hardware.py` vào đây, hoặc điền thủ công:

- **OS:** _Ubuntu 24.04 (Linux x86_64)_
- **CPU:** _11th Gen Intel(R) Core(TM) i7-1165G7 @ 2.80GHz_
- **Cores:** _8 physical / 8 logical_
- **CPU extensions:** _AVX2, AVX-512_
- **RAM:** _15.4 GB_
- **Accelerator:** _None (CPU only)_
- **llama.cpp backend đã chọn:** _CPU (AVX/NEON tuning)_
- **Recommended model tier:** _Qwen2.5-1.5B-Instruct (Q4_K_M)_

**Setup story** (≤ 80 chữ): những gì cần thay đổi để lab chạy được trên máy bạn (vd: dùng WSL2, install CUDA Toolkit, fall back sang Vulkan vì ROCm phiên bản kén, tắt antivirus để pip install nhanh hơn, v.v.):

_Intel i7-1165G7 laptop with no GPU. Set LAB_N_GPU_LAYERS=0 in .env for CPU-only mode. llama.cpp runs well with AVX2/AVX-512 support. Downloaded Q4_K_M and Q2_K quantized models. No special setup needed beyond standard pip install locust. Server runs on port 8080._

---

## 2. Track 01 — Quickstart numbers (từ `benchmarks/01-quickstart-results.md`)

> Paste bảng từ `benchmarks/01-quickstart-results.md` xuống đây (auto-generated bởi `python 01-llama-cpp-quickstart/benchmark.py`).

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|--:|--:|--:|--:|--:|
| qwen2.5-1.5b-instruct-q4_k_m.gguf | 976 | 1354 / 1620 | 159.2 / 168.7 | 11412 / 11962 / 12038 | 6.3 |
| qwen2.5-1.5b-instruct-q2_k.gguf | 2384 | 2610 / 3153 | 263.0 / 264.9 | 19198 / 19828 / 19870 | 3.8 |

**Một quan sát** (≤ 50 chữ): Q4_K_M vs Q2_K trên máy bạn — số liệu nói gì? Quality đáng đánh đổi không?

_Q4_K_M nhanh hơn ~40% so với Q2_K (TPOT 159 vs 263 ms). Q4_K_M đáng dùng vì vừa nhanh hơn vừa chất lượng hơn, trong khi Q2_K chỉ tiết kiệm RAM hơn._

---

## 3. Track 02 — llama-server load test

> Chạy 2 lần locust ở concurrency 10 và 50, paste tóm tắt bên dưới.

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|--:|--:|--:|--:|--:|--:|
| 10 | 0.13 | 31000 | 54000 | 54000 | 0 |
| 50 | 0.15 | 38000 | 48000 | 48000 | 0 |

**KV-cache observation** (từ `record-metrics.py`): peak `llamacpp:n_busy_slots_per_decode` ở concurrency 50 = _2.37 slots/decode_; `tokens_predicted_total` = 1957 (static, server idle during recording). Không có metric `llamacpp:kv_cache_usage_ratio` trong output, nghĩa là …

_Server chạy idle 30s không có request mới, nên `tokens_predicted_total` giữ nguyên 1957. `n_busy_slots_per_decode ≈ 2.37` cho thấy decode step trung bình dùng ~2.4 slots. CPU-only mode không dùng GPU-backed KV cache — llama.cpp quản lý KV trên RAM, và khi không có new requests, không có reuse signal để measure. KV-cache benefit thể hiện qua `n_busy_slots_per_decode` thấp → nhiều slots trống cho parallel requests, nhưng trên CPU-only i7-1165G7, benefit bị giới hạn bởi memory bandwidth._

---

## 4. Track 03 — Milestone integration

- **N16 (Cloud/IaC):** _stub: localhost only_
- **N17 (Data pipeline):** _stub: in-memory dict (TOY_DOCS)_
- **N18 (Lakehouse):** _stub: SQLite_
- **N19 (Vector + Feature Store):** _stub: TOY_DOCS (in-memory toy documents)_

**Nơi tốn nhiều ms nhất** trong pipeline (đo bằng `time.perf_counter` trong `pipeline.py`):

- embed: _~0 ms (stub — TOY_DOCS lookup, no embedding model)_
- retrieve: _~0 ms (stub — dict lookup, no vector search)_
- llama-server: _9779.8 / 2095.4 / 10622.8 ms (3 queries từ pipeline.py output)_

**Reflection** (≤ 60 chữ): bottleneck nằm ở đâu? Có khớp với kỳ vọng không?

_Bottleneck là llama-server inference (9.8–10.6s per request) — chiếm >99% total pipeline time. Stub N16-N19 gần như 0ms. Đúng với kỳ vọng: CPU-only i7-1165G7 decode ~6 tok/s, mỗi response 150-200 tokens mất ~25s. Từ Locust: concurrency 10 → TTFB 31s, concurrency 50 → TTFB 38s. Bottleneck không phải retrieval mà là LLM decode — KV-cache reuse qua shared prefix giúp giảm prefill, nhưng decode vẫn bounded by CPU compute._

---

## 5. Bonus — The single change that mattered most

> **Most important section.** Pick **một** thay đổi từ bonus track (build flag, thread sweep, quant pick, GPU offload, KV-cache quantization, speculative decoding, bất cứ challenge nào trong `BONUS-llama-cpp-optimization/CHALLENGES.md`) đã tạo ra speedup lớn nhất trên máy bạn.

**Change:** _Chọn quantization Q4_K_M thay vì Q2_K cho primary model (trong models/active.json)_

**Before vs after** (paste 2-3 dòng từ sweep output):

```
before (Q2_K):  TPOT P50 = 263.0 ms, Decode rate = 3.8 tok/s
after  (Q4_K_M): TPOT P50 = 159.2 ms, Decode rate = 6.3 tok/s
speedup: ~1.65×
```

**Tại sao nó work** (1–2 đoạn ngắn — đây là phần grader đọc kỹ nhất):

_Q4_K_M có mức nén thấp hơn Q2_K (4-bit vs 2-bit), giúp giảm số lượng bits phải xử lý per token. Trên CPU i7-1165G7, bandwidth-bound workload này được hưởng lợi từ ít bits hơn nhưng không quá mức như Q2_K. Q4_K_M cân bằng tốt giữa chất lượng output và tốc độ decode, nhanh hơn 65% mà vẫn giữ quality tốt hơn Q2_K. Với RAM 15.4GB, Q4_K_M (1.1GB) hoàn toàn khả thi._

---

## 6. (Optional) Điều ngạc nhiên nhất

_Q2_K (2-bit quantization) chậm hơn Q4_K_M (~40% slower) dù dung lượng nhỏ hơn. Thường nghĩ model nhỏ hơn sẽ nhanh hơn, nhưng on CPU, ít bits hơn có thể dẫn đến inefficient memory access patterns. Q4_K_M là sweet spot cho CPU-only inference._

---

## 7. Self-graded checklist

- [x] `hardware.json` đã commit
- [x] `models/active.json` đã commit (hoặc paste path snapshot vào section 1)
- [x] `benchmarks/01-quickstart-results.md` đã commit
- [x] `benchmarks/02-server-results.md` (hoặc CSV từ `record-metrics.py`) đã commit
- [ ] `benchmarks/bonus-*.md` đã commit (ít nhất 1 sweep) — bonus, optional
- [x] Ít nhất 6 screenshots trong `submission/screenshots/` (xem `submission/screenshots/README.md`)
- [x] `make verify` exit 0 (chạy ngay trước khi push)
- [ ] Repo trên GitHub ở chế độ **public**
- [ ] Đã paste public repo URL vào VinUni LMS

---

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Nếu private, grader không xem được → 0 điểm.
