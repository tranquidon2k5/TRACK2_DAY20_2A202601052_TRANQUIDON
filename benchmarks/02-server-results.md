# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=6` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 108 | 1.90 | 3900 | 5800 | 7000 | 7.9 | 0.0% |
| 50 | 106 | 1.85 | 24000 | 27000 | 28000 | 37.2 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.98x** (20% of linear) |
| P95 latency | **4.66x** |
| Effective concurrency at 50 users | 37.2 vs `--parallel 4` slots (occupancy/slot ratio 9.30) |

**Saturated.** Throughput delivered only 0.98x for 5x the offered load, and effective concurrency (37.2) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.98x while P95 moved 4.66x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

1. **Vị trí Bão hòa & Bằng chứng (Saturation Point & Evidence)**:
   - Server rơi vào trạng thái **bão hòa (Saturated)** rõ rệt khi số lượng user tăng từ 10 lên 50.
   - Bằng chứng số liệu: Khi tải gửi vào tăng **5x**, Throughput thực tế (RPS) bị chững lại (từ **1.90 req/s xuống 1.85 req/s**, chỉ đạt **0.98x**, tương đương 20% tăng trưởng tuyến tính). Ngược lại, **P95 Latency bùng nổ tăng gấp 4.66x** (từ 5,800 ms lên 27,000 ms).
   - Định luật Little chỉ ra Effective Concurrency vọt lên **37.2**, gấp 9.3 lần tổng số slot của `--parallel 4`. Điều này chứng minh toàn bộ 4 decode slot đã hoạt động 100% công suất, và lượng tải dư thừa biến thành thời gian chờ trong hàng đợi (queue time) kéo tụt P95.

2. **Đề xuất Tối ưu Goodput cho SLO**:
   - Nếu đặt mục tiêu SLO cho P95 <= 8,000 ms, núm vặn đầu tiên cần điều chỉnh là **tăng số slot `--parallel` (ví dụ từ 4 lên 8 slots)** nếu VRAM còn dư, kết hợp áp dụng **Load Shedding / Queue Timeout** ở phía Gateway.
   - Lý do chọn núm vặn này: Hiện tại nghẽn nút cổ chai nằm ở việc thiếu slot xử lý song song dẫn đến hàng đợi bị ùn tắc (`requests_deferred`). Việc tăng `--parallel` hoặc từ chối bớt request vượt quá năng lực phục vụ của 4 slots sẽ giữ độ trễ P95 nằm trong ngưỡng SLO, từ đó tối đa hóa Goodput thực tế.
