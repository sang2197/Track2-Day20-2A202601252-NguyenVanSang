# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** _Nguyễn Văn Sáng_
**Cohort:** _A20-K4_
**Ngày submit:** _2026-08-20_

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11 (AMD64)
- **CPU:** AMD Ryzen 7 7735HS with Radeon Graphics
- **Cores:** 8 physical / 16 logical
- **CPU extensions:** AVX2 + FMA (backend `ggml-cpu-haswell.dll` được nạp; không có AVX-512)
- **RAM:** 15.2 GB
- **Accelerator:** NVIDIA GeForce RTX 4060 Laptop GPU, 8188 MiB (CUDA, GPU offload ACTIVE) — Vulkan cũng sẵn có
- **llama.cpp asset đã tải:** `llama-b10488-bin-win-cuda-12.4-x64.zip` (+ `cudart-llama-bin-win-cuda-12.4-x64.zip`)
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL, 2.97 GB (primary) + UD-Q2_K_XL, 2.24 GB (compare)

**Chạy ở đâu:** Laptop của tôi

**Setup story** (≤ 80 chữ): `lab.ps1` gốc lưu UTF-8 không có BOM; Windows PowerShell 5.1
đọc file theo ANSI codepage hệ thống khi thiếu BOM, làm hỏng ký tự em-dash trong chuỗi
và báo lỗi parse trước khi chạy được lệnh nào. Thêm BOM UTF-8 vào đầu file là fix. Sau đó
`setup` chạy suôn, không cần workaround nào khác.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 6208 | 525 / 978 | 10.6 / 11.6 | 1178 / 1649 / 1649 | 94.4 |
| UD-Q2_K_XL | 2.24 | 6168 | 502 / 894 | 9.4 / 10.5 | 1086 / 1483 / 1483 | 106.7 |

**Quan sát** (≤ 60 chữ): Q2_K_XL nhanh hơn 1.13x, nhẹ hơn 0.73 GB (~25%). Đã hỏi cùng
2 câu (giải thích khái niệm + toán suy luận) trên cả hai server (`--port 8080` vs
`--compare --port 8090`). Không thấy khác biệt chất lượng rõ rệt với prompt ngắn/trung bình → đáng dùng Q2_K_XL cho lab này.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 3.82 | 1600 | 2800 | 4400 | 6.6 | 0.0% |
| 50 | 3.65 | 13000 | 14000 | 14000 | 42.5 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 0.96×
- **P95 tăng:** 5.00×
- **Effective concurrency ở 50 users:** 42.5 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.98 / 4 slots

**Saturation reading** (≤ 80 chữ): Server bão hoà ở hoặc dưới 10 users. Bằng chứng:
RPS gần như đứng yên (0.96×) dù tải tăng 5×; `n_busy_slots_per_decode` đỉnh 3.98/4 (99%)
xác nhận decode engine full; effective concurrency (6.6 ở 10 users, đã > 4 slots) cho
thấy hàng đợi hình thành từ sớm. Latency thêm là **queue time** (chờ slot), không phải
compute time (TPOT/token không đổi theo tải, §2). Sẽ đổi `--parallel` (4→8) trước, vì
đúng cơ chế: batching amortize chi phí đọc VRAM qua nhiều sequence/step, và GPU còn dư
~7 GB VRAM để tăng slot. Đổi `-t` vô ích — §5 đã chứng minh threads không phải bottleneck
khi `ngl=99`.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | — | Stub ("localhost only") |
| N17 Data pipeline | `TOY_DOCS` | Stub (in-memory list) |
| N18 Lakehouse | `TOY_DOCS` | Stub (vẫn là dict, không có Delta/Iceberg) |
| N19 Vector + features | keyword overlap | Stub (chưa nối vector index thật) |
| N20 Serving | `llama-server` | Real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.3 ms
- llm: 3023.4 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): `llm` chiếm gần 100% — đúng kỳ vọng vì N17-N19 đang stub nên
embed/retrieve gần như miễn phí. Điều bất ngờ: bên trong `llm`, wall-time client đo được
(~3023ms) lớn hơn nhiều tổng prefill+decode server tự báo (~470-670ms) — overhead ngoài
model compute. Muốn giảm 2× latency, việc đầu tiên là tìm và cắt khoảng overhead đó
(client `httpx.post` không giữ keep-alive), chứ không phải đổi quantization trước.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** `LAB_N_THREADS` sweep (`make tune`): `-t 1` → `-t 16` (best trong sweep)

```
before:  107.5 tok/s  (-t 1)
after:   113.1 tok/s  (-t 16)
speedup: 1.05×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Điều đáng nói không phải con số 1.05× — mà là nó nhỏ hơn nhiều so với kỳ vọng từ deck, và lý do lộ ra ngay từ config: `ngl=99` nghĩa là gần như toàn bộ layer của Gemma 4 E2B được offload lên GPU. Phần compute nặng nhất của decode là nhân ma trận mỗi token sinh ra lại chạy trên CUDA và bị chặn bởi băng thông đọc trọng số từ VRAM, không phải bởi số CPU thread. `-t` trong benchmark này chỉ quyết định bao nhiêu thread CPU lo việc điều phối (dispatch kernel, sampling, quản lý KV cache), nên sweep từ 1 → 32 thread gần không thay đổi (95%–100%), không có knee rõ ở physical core (8) như mô tả cho CPU-only decode, và `-t 32` cũng chỉ giảm nhẹ 0.7% chứ không sập vì scheduling overhead.

Nói cách khác: trên máy có GPU offload đầy đủ, bottleneck của TPOT đã chuyển từ CPU threading sang GPU memory bandwidth trước khi thread count kịp gây ảnh hưởng đáng kể. Muốn có speedup lớn hơn ở TPOT trên máy này, không phải `-t` mà là những tác động lên lượng byte phải đọc từ VRAM mỗi token — ví dụ quantization thấp hơn hoặc `--cache-type-k/v` cho KV cache nhỏ hơn.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** _<B1 build-compare / B2 sweep nào / B4 challenge nào / B5 lựa chọn nào>_

**Numbers:**

```
before:  <số>
after:   <số>
speedup: <X.Y>×
```

**Điều này nói lên gì mà deck chưa nói:**

_(để trống nếu bạn không làm phần này)_

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

_(để trống nếu bạn không làm phần này)_

---

## 8. Self-check trước khi push

- [ ] `hardware.json` committed
- [ ] `models/active.json` committed
- [ ] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [ ] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [ ] `benchmarks/02-server-results.md` committed (`make load-report`)
- [ ] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [ ] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [ ] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [ ] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [ ] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
