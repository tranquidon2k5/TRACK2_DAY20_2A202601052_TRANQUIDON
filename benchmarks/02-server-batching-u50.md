# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 14 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.93 of 4 slots (98%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 11654 |

Highest sampled value was **3.93 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

1. **Peak Batch Width**:
   - Peak sampled của `n_busy_slots_per_decode` đạt **3.93 trên tổng số 4 slots** (tỷ lệ lấp đầy 98%).
   - Điều này chứng minh tính năng **Continuous Batching** của `llama-server` đã hoạt động cực kỳ hiệu quả, liên tục ghép các request song song vào cùng một bước decode.

2. **So sánh với Effective Concurrency (35.0) trong `02-server-results.md`**:
   - Chỉ số **Effective Concurrency** ($L = 35.0$) trong `02-server-results.md` đo tổng số request tồn tại trong hệ thống ở mọi thời điểm (bao gồm 4 request đang tính toán + 46 request nằm chờ trong hàng đợi `requests_deferred`).
   - Ngược lại, gauge **`n_busy_slots_per_decode` (3.93)** phản ánh số slot GPU thực sự đang hoạt động ở từng bước decode.
   - Khi đánh giá khả năng batching của server, ta tin tưởng chỉ số `n_busy_slots_per_decode` từ `/metrics` vì nó đo trực tiếp từ llama.cpp engine, cho thấy server đã khai thác tối đa gần như 100% dung lượng 4 slot `--parallel 4`. Chi phí trễ P95 tăng vọt chủ yếu là do hàng đợi (queueing delay) của 46 request bị defer.
