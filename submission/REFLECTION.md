# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** TRAN QUI DON  
**Cohort:** A20-K26  
**Ngày submit:** 2026-08-20  

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11 AMD64  
- **CPU:** Intel Core i5-11400H @ 2.70GHz  
- **Cores:** 6 physical / 12 logical  
- **CPU extensions:** AVX2, FMA, F16C  
- **RAM:** 15.8 GB  
- **Accelerator:** NVIDIA GeForce RTX 3050 Laptop GPU (4 GB VRAM)  
- **llama.cpp asset đã tải:** `llama-b10488-bin-win-cuda-cu12.4-x64.zip`  
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)  
- **Quantization:** `UD-Q4_K_XL` (primary) + `UD-Q2_K_XL` (compare)  

**Chạy ở đâu:** laptop của tôi  

**Setup story** (≤ 80 chữ):  
Khi chạy trên Windows 11, script `verify.py` bị lỗi không nhận diện file committed do sự khác biệt dấu phân cách đường dẫn (`\` của Windows vs `/` của Git). Mình đã patch lại hàm `is_committed` trong `verify.py` và sửa một số lỗi trích dẫn chuỗi PowerShell trong `lab.ps1` để toàn bộ môi trường chạy mượt mà.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 5574 | 319 / 551 | 13.9 / 16.3 | 1171 / 1424 / 1424 | 72.2 |
| UD-Q2_K_XL | 2.24 | 4311 | 301 / 550 | 16.9 / 18.8 | 1401 / 1576 / 1576 | 59.2 |

**Quan sát** (≤ 60 chữ):  
Bản 2-bit chậm hơn bản 4-bit 1.22x (59.2 vs 72.2 tok/s) dù nhẹ hơn 0.73 GB. Lý do GPU RTX 3050 bị gánh chi phí dequantize trọng số 2-bit phức tạp. Bản 2-bit không đáng dùng vì 4GB VRAM thừa sức chứa bản 4-bit, vốn cho câu trả lời chính xác và mạch lạc hơn hẳn.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 1.90 | 3900 | 5800 | 7000 | 7.9 | 0.0% |
| 50 | 1.85 | 24000 | 27000 | 28000 | 37.2 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 0.98×  
- **P95 tăng:** 4.66×  
- **Effective concurrency ở 50 users:** 37.2 so với `--parallel` = 4 slots  

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang chạy): 3.93 / 4 slots  

**Saturation reading** (≤ 80 chữ):  
Server bão hòa rõ rệt ở 50 users: RPS không tăng (0.98x) nhưng P95 tăng 4.66x. Latency tăng thêm hoàn toàn là queue time vì Effective Concurrency (37.2) gấp 9.3 lần 4 slots, khiến 46 request bị hoãn. Để nâng Goodput@SLO, mình sẽ ưu tiên tăng `--parallel` (lên 8 slots) hoặc cài Load Shedding ở Gateway.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Inference Engine / API Endpoint | real |
| N17 Data pipeline | Model Weights (`gemma-4-E2B`) | real |
| N18 Lakehouse | Embedding Model (Keyword Fallback) | stub |
| N19 Vector + features | Vector Store (`TOY_DOCS`) | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms  
- retrieve: 0.1 ms  
- llm: 3010.4 ms  
- **stage chiếm nhiều nhất:** llm (100% của total)  

**Reflection** (≤ 60 chữ):  
Bottleneck nằm 100% ở stage LLM (3010.4 ms), hoàn toàn đúng kỳ vọng vì việc tính toán ma trận LLM tốn tài nguyên nhất. Để giảm độ trễ 2x, mình sẽ tấn công vào stage LLM bằng kỹ thuật Prompt Caching (RadixAttention) để tái sử dụng KV cache cho System Prompt và context RAG trùng lặp.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Điều chỉnh số luồng CPU `-t` từ 6 (mặc định theo số physical cores) xuống `-t 1` khi offload toàn bộ layer sang GPU (`ngl=99`).

```
before:  73.0 tok/s (-t 6)
after:   77.7 tok/s (-t 1)
speedup: 1.06×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Trên chiếc laptop của mình, GPU NVIDIA GeForce RTX 3050 có 4GB VRAM và tham số `ngl=99` đã đẩy gần như toàn bộ các layer tính toán của mô hình Gemma 4 E2B lên trực tiếp VRAM để GPU thực thi bằng các nhân CUDA.

Khi công việc chính được xử lý hoàn toàn trên GPU, vai trò duy nhất của các luồng CPU trong bước decode (`tg128`) chỉ đơn thuần là phân phối (dispatch) các lệnh tính toán xuống GPU stream. Nếu giữ mặc định 6 luồng (`-t 6`) hoặc tăng lên 12 luồng CPU, các luồng này không giúp GPU tính toán nhanh hơn mà ngược lại còn làm tăng **chi phí đồng bộ luồng (thread synchronization overhead)**, gây ra tranh chấp khóa mutex và chuyển đổi ngữ cảnh (context switching) không cần thiết giữa các nhân CPU. Việc giảm xuống `-t 1` giúp tối giản hóa chi phí điều khiển của CPU, giải phóng băng thông CPU và mang lại tốc độ sinh từ cao nhất (**77.7 tok/s**).

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** _(để trống vì tập trung hoàn thiện base track)_

**Numbers:**

```
before:  
after:   
speedup: 
```

**Điều này nói lên gì mà deck chưa nói:**

_(để trống nếu bạn không làm phần này)_

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Điều ngạc nhiên nhất với mình là việc bản quantization 2-bit (`UD-Q2_K_XL`) lại chạy chậm hơn bản 4-bit (`UD-Q4_K_XL`) trên GPU RTX 3050 (59.2 tok/s vs 72.2 tok/s). Mình cứ nghĩ nhỏ hơn thì lúc nào cũng sẽ nhanh hơn, nhưng hóa ra trên phần cứng có GPU acceleration, chi phí tính toán giải nén (dequantization overhead) của định dạng nén quá nặng lại vượt quá lợi ích tiết kiệm băng thông bộ nhớ.

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md` đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [ ] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không xem được → 0 điểm.
