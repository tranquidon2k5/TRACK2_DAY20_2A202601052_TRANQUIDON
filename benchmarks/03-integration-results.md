# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 3134.5 | 3134.6 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 2842.6 | 2842.7 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 2815.1 | 2815.2 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **2930.7** · total **2930.8**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

1. **Trạng thái thực tế (Real vs Stubbed) của các thành phần**:
   - **N16 (Serving Engine / API Endpoint)**: **REAL** — `llama-server` thật đang chạy phục vụ cổng OpenAI-compatible API (`/v1/chat/completions`).
   - **N17 (Model Weights & Quantization)**: **REAL** — Sử dụng mô hình thật Gemma 4 E2B (`gemma-4-E2B-it-UD-Q4_K_XL.gguf`).
   - **N18 (Embedding Model)**: **STUBBED** — Sử dụng giải thuật fallback Keyword Overlap.
   - **N19 (Vector Search Index & Lakehouse)**: **STUBBED** — Sử dụng danh sách tài liệu mẫu `TOY_DOCS` lưu trong bộ nhớ.

2. **Đánh giá Dominant Stage & Đề xuất giảm 50% Latency**:
   - **Dominant Stage**: Stage **`llm`** chiếm **100%** tổng thời gian thực thi của pipeline (trung bình **2930.7 ms** trên tổng **2930.8 ms**). Kết quả này hoàn toàn đúng kỳ vọng vì việc tính toán prefill và tự sinh token (decode loop) của mô hình LLM là tác vụ nặng nhất.
   - **Chiến lược giảm 50% Latency**: Để cắt giảm một nửa độ trễ pipeline, ta bắt buộc phải tấn công vào stage **`llm`**:
     - Áp dụng **Prompt Caching** (như RadixAttention trong llama.cpp) để tái sử dụng KV cache cho phần System Prompt và Context trùng lặp giữa các truy vấn, giúp bỏ qua thời gian prefill.
     - Tinh chỉnh tham số nén context và giới hạn `max_tokens` đầu ra của LLM.
