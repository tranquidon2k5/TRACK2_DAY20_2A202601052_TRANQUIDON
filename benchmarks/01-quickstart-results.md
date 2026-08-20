# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=6` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 5574 | 319 / 551 | 13.9 / 16.3 | 1171 / 1424 / 1424 | 72.2 |
| UD-Q2_K_XL | 2.24 | 4311 | 301 / 550 | 16.9 / 18.8 | 1401 / 1576 / 1576 | 59.2 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.22x SLOWER** than `UD-Q4_K_XL` here, despite being 0.73 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

1. **So sánh Tốc độ & Latency**:
   - Bản 4-bit (`UD-Q4_K_XL`) đạt tốc độ decode **72.2 tok/s** (TPOT P50 = 13.9 ms), nhanh hơn **1.22x** so với bản 2-bit (`UD-Q2_K_XL`) vốn chỉ đạt **59.2 tok/s** (TPOT P50 = 16.9 ms).
   - TTFT P50 tương đương nhau (~319 ms vs 301 ms).
   - Nguyên nhân: Máy chạy trên GPU NVIDIA RTX 3050 Laptop với CUDA acceleration. Ở cấu hình này, việc dequantize các trọng số 2-bit phức tạp tiêu tốn thêm chu kỳ tính toán (compute overhead), lớn hơn lợi ích tiết kiệm băng thông bộ nhớ.

2. **Dung lượng & Tài nguyên**:
   - Bản 2-bit tiết kiệm được 0.73 GB dung lượng (2.24 GB so với 2.97 GB) và load nhanh hơn ~1.2s (4311 ms vs 5574 ms).
   - Tuy nhiên, GPU có 4GB VRAM và RAM máy có 15.8 GB, nên bản 4-bit (2.97 GB) hoàn toàn nằm gọn trong VRAM mà không bị tráo đổi (swapping) hay tràn memory.

3. **Kết luận**:
   - Bản 2-bit **KHÔNG ĐÁNG DÙNG** trên máy này. Bản 4-bit không những mang lại tốc độ phản hồi vượt trội (72.2 tok/s so với 59.2 tok/s) mà còn duy trì độ chính xác và chất lượng câu trả lời cao hơn hẳn so với bản 2-bit bị nén quá mức.
