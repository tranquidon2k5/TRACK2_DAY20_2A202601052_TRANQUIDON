# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **6 physical · 12 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 77.7 | 100% |
| 3 | 75.5 | 97% |
| 6 | 73.0 | 94% |
| 12 | 71.3 | 92% |
| 24 | 70.1 | 90% |

**Best**: `-t 1` at 77.7 tok/s
**Slowest tested**: `-t 24` at 70.1 tok/s (1.11x spread)
**Against the physical-core default** (`-t 6`, 73.0 tok/s): 1.06x

Use this in your run:

```bash
LAB_N_THREADS=1 make bench
```

## Your explanation

1. **Phân tích vị trí Peak & Knee**:
   - Tốc độ sinh chuỗi (decode `tg128`) đạt cao nhất ở **`-t 1`** với **77.7 tok/s**.
   - Khi tăng số thread lên 3, 6, 12, 24, tốc độ giảm dần từ 77.7 tok/s xuống 70.1 tok/s (giảm ~10%).

2. **Giải thích Nguyên nhân Cơ chế (GPU Offload vs CPU Threading)**:
   - Toàn bộ các layer của mô hình Gemma 4 E2B đã được **offload lên GPU NVIDIA RTX 3050 (`ngl=99`)** thông qua CUDA.
   - Khi chạy hoàn toàn trên GPU, CPU chỉ đóng vai trò phân phối (dispatching) các CUDA kernel và điều khiển luồng tính toán. Việc tăng số luồng CPU (`-t 3`, `-t 6`, `-t 12`, `-t 24`) không hề tăng tốc độ tính toán cho GPU, mà ngược lại còn phát sinh **chi phí đồng bộ luồng (thread synchronization overhead)**, tranh chấp mutex và context switching giữa các CPU core.
   - Đối với mô hình chạy thuần CPU, tốc độ thường đạt đỉnh ở số core thực (6 physical cores). Nhưng khi có **GPU acceleration (`ngl=99`)**, 1 luồng CPU (`-t 1`) mang lại hiệu năng cao nhất nhờ tối thiểu hóa overhead điều khiển luồng CPU.
